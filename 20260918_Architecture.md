# ARC / Arkema Pricing Cockpit — Architecture

**Date:** September 18, 2026
**Author:** William Lin (Snowflake PS — App Productionalization workstream)
**Source:** read directly from `arkema-pricing-ss/arc/` on Sep 18, 2026. Every figure below is measured from the repository, not taken from the architecture doc in it.
**Companion diagram:** `20260918_Architecture.pptx`

> **How to read this.** Everything stated as fact here was verified against a file, and the file is cited. Where the repo's own `docs/ARCHITECTURE.md` and the code disagree, the code wins and the disagreement is called out in §9.

---

## 1. Headline finding: `arc/` contains two complete, separate applications

This is the single most important thing to understand, and it is not stated in `docs/ARCHITECTURE.md`. The `arc/` tree holds **two app variants, targeting two different Snowflake databases, with two different data architectures**. They are not a refactor of one another — they coexist deliberately.

| | **Variant A — POC / NLC2** | **Variant B — MVP Cockpit** |
|---|---|---|
| **Database** | `ARKEMA_PRICING_DB` | **`ARKEMA_PRICING_MVP`** |
| **Data architecture** | `RAW → ATOMIC → MART` + `ML`, `GOV`, `DOCS`, `SPCS` | **Medallion:** `LANDING → BRONZE → GOLD` + `GOV`, `UTIL` |
| **DDL** | `ddl/*.sql` — 21 files, 3,298 lines | `ddl/mvp/*.sql` — **27 files, 10,098 lines** |
| **Frontend** | `frontend/src/pages/` — 12 pages | `frontend/src/mvp/pages/` — 11 pages, 16 registered views |
| **Backend routes** | `backend/api/main.py` — 49 routes | `mvp_routes.py` (30) + `mvp_bom_routes.py` (5) + `mvp_f9_routes.py` (5) + `mvp_f10_routes.py` (4) = **44 routes** |
| **Personas** | 4 (`Commercial`, `FPA`, `PricingAnalyst`, `Steward`) | **6** (`cm`, `fpa`, `pa`, `ds`, `sr`, `dampe`) |
| **AI layer** | Cortex Agent + Cortex Search + Cortex Analyst | GenAI view exists; **`backing: ''`** — not wired |
| **ML** | `ml/*.py` — 5 scripts, 975 lines, M1/M2/M3 models | None of its own |
| **Service spec** | `deploy/service_spec.yaml` | `deploy/service_spec_mvp.yaml` |
| **Authoritative?** | The POC that is currently running | **Built to PRD v2.22, which the project plan names as authoritative** |

### 1.1 Why they are separate — in the repo's own words

From `ddl/mvp/000_mvp_database.sql`, lines 8–14:

> *"WHY A SEPARATE DATABASE (MVP Bridge §7 Phase 0.2): The live POC SPCS service reads `ARKEMA_PRICING_DB`. A `CREATE OR REPLACE` in that database previously destroyed 18 live scenarios. The MVP data foundation is a rebuild (new sales view, 12× BOM, reversed key convention), not an extension — so it lands in its own database. The POC keeps running untouched, and this database is a clean handover artifact for PS."*

Two things follow from that paragraph, and both matter to our workstream:

1. **The isolation is a safety control, not an accident.** It exists because a destructive DDL run has already caused data loss once. It should be preserved.
2. **`ARKEMA_PRICING_MVP` is explicitly described as "a clean handover artifact for PS"** — i.e. for us. That is a strong signal about which variant we are meant to productionalize.

`deploy/service_spec_mvp.yaml` reinforces it, pointing the MVP container at `SNOWFLAKE_DATABASE: ARKEMA_PRICING_MVP` with the comment that the MVP router fully qualifies every object so *"even a stray unqualified name resolves inside the MVP database and cannot touch the live POC objects."*

**This needs an explicit decision before deployment — see §10, item 1.** It is the largest open architectural question in the workstream.

---

## 2. Tier architecture (common to both variants)

Both variants share one process, one container, and one Snowflake connection model. The split is in the routers and the data they read.

```
Browser (React 18 + TypeScript + Vite)
   │  HTTPS, X-ARC-Persona header
   ▼
nginx  (deploy/nginx.conf)  — serves static build, reverse-proxies /api
   │
   ▼
FastAPI  (backend/api/main.py + 4 mvp routers)   ── 93 routes total
   │
   ├── ArcService      (backend/services/arc_service.py)      — POC queries
   ├── Nlc2Service     (backend/services/nlc2_service.py)     — NLC2/v2 queries
   └── ArcAgentClient  (backend/services/arc_agent_client.py) — Cortex Agent REST
   │
   ▼
Snowflake  (SPCS OAuth token at /snowflake/session/token, or snow CLI locally)
   │
   ├── ARKEMA_PRICING_DB    — POC: RAW / ATOMIC / MART / ML / GOV / DOCS / SPCS
   ├── ARKEMA_PRICING_MVP   — MVP: LANDING / BRONZE / GOLD / GOV / UTIL
   └── SNOWFLAKE_INTELLIGENCE.AGENTS — ARC_AGENT, ARC_V2_AGENT
```

### 2.1 Frontend

- **Stack:** React 18, TypeScript, Vite 5, Tailwind 3, Leaflet 1.9 (map), Vega/Vega-Lite 5 + Recharts 2 (charts), react-markdown. 12,571 lines across `frontend/src/`.
- **Two mount points:** `src/main.tsx` → `src/App.tsx` (POC) and `src/mvp/main.tsx` → `src/mvp/App.tsx` (MVP).
- **Persona is client-side.** `src/lib/api.tsx:36` sets the `X-ARC-Persona` header from a UI selector. There is no authentication anywhere — see §7.
- **The MVP view registry is unusually honest.** `src/mvp/lib/views.ts` records, per screen, which GOLD objects back it, with `backing: ''` meaning still on mock data. It drives a "DATA" badge in the topbar *"so nobody mistakes a mock for a wired view during a demo."* Four of the 16 MVP views are unbacked: `pi-tracking`, `pi-targets`, `sales-queue`, `genai`.

### 2.2 Backend

- **Stack:** FastAPI (Python), 4,378 lines across `backend/`. Run by `deploy/start.sh` under nginx.
- **93 route decorators** across 5 modules (49 POC + 44 MVP).
- **Snowflake connectivity** (`arc_service.py`): primary path is the `snow` CLI as a subprocess; fallback is `snowflake.connector` using the SPCS OAuth token read from `/snowflake/session/token`, with `SNOWFLAKE_HOST` from the environment. Includes auto-reconnection on token expiry (`_maybe_reconnect`).
- **Caching is deliberate and documented.** `main.py` warms an in-process cache at startup because *"the first visitor to each view waits 10–40s on a spinner while Snowflake answers; afterwards the in-process cache serves in ~2ms."* This is a real fix for a measured problem (MVP-BRIDGE D28), not premature optimisation.

### 2.3 Data layer — Variant A (`ARKEMA_PRICING_DB`)

Created by `ddl/001_database.sql`. Seven schemas:

| Schema | Purpose (from the DDL comments) | Contents measured |
|---|---|---|
| `RAW` | Landing zone + marketplace passthrough views | 8 tables (`NLC2_SALES`, `NLC2_ZFSC07`, `NLC2_DAMPE`, `NLC2_BOM_FLAT`, `NLC2_MARM`, `NLC2_MARC`, `NLC2_CUSTOMER_SEG`, `NLC2_MACRO`), 1 stage, 2 file formats |
| `ATOMIC` | Normalized enterprise data model | 13 views (`NLC2_FACT_SALES`, `NLC2_FACT_BOM`, `NLC2_FACT_PRICE_REF`, `NLC2_FACT_FG_WAC`, …) |
| `MART` | Analytics views & data products | ~12 tables (`NLC2_CUSTOMER_MARGIN`, `NLC2_COST_SIMULATION_RUN`, `NLC2_AI_RECOMMENDATION`, `NLC2_PI_TARGET`, …), 6 views, 6 procedures, `SEMANTIC_MODELS` stage |
| `ML` | Models, predictions, explainability | 4 tables (`NLC2_ELASTICITY`, `NLC2_CROSS_ELASTICITY`, `NLC2_VOLUME_RISK`, `NLC2_PI_EVENT`), 2 procedures, **5 tasks** |
| `GOV` | Data quality, lineage, governance | 14 DQ/lineage views, 4 tables, 1 procedure |
| `DOCS` | Unstructured market notes for Cortex Search | `MARKET_NOTES` table + `MARKET_NOTES_STAGE`, 13 synthetic market-note markdown files |
| `SPCS` | Container Services artifacts | `ARC_IMAGES` image repository |

### 2.4 Data layer — Variant B (`ARKEMA_PRICING_MVP`)

Created by `ddl/mvp/000_mvp_database.sql`. A medallion model, and **substantially the larger of the two** at 10,098 lines of DDL:

| Schema | Purpose (from the DDL comments) | Contents measured |
|---|---|---|
| `LANDING` | File formats + stages; raw files land here before `COPY INTO BRONZE` | 1 stage |
| `BRONZE` | Raw landed source data, 1:1 with delivered extracts. No business logic, no filters | **12 tables** |
| `GOLD` | Curated facts/dims/refs per data catalog v1.5. Silver logic applied here as views — no separate silver layer at MVP | **57 views + 24 tables** |
| `GOV` | Data quality, contract tests, reconciliation results | **43 views + 4 tables**, incl. `MVP_CHECK_SUMMARY`, `MVP_PARAMETER`, `MVP_DATA_REQUEST` |
| `UTIL` | Shared contract functions. *"Every ingest MUST use these — they encode the traps in MVP-BRIDGE.md §4"* | Contract functions |

Two design choices worth noting, because they are good and should survive productionalization:

- **`GOV.MVP_PARAMETER`** — a durable parameter table replacing session variables. Added because *"a session variable cannot serve this — views built on one fail to expand in any other session"* (MVP-BRIDGE D14). This is the right pattern.
- **`GOV.MVP_CHECK_SUMMARY`** — the 43 GOV check views are validation harnesses, not serving objects; unioning all 14 suites takes >120s. They are materialised with an as-of timestamp and refreshed as the last step of a deploy (D26).

### 2.5 AI layer (Variant A only)

Four distinct Snowflake AI components. None of them exist in the Arkema account today.

| Component | Object | Defined in |
|---|---|---|
| **Cortex Analyst** semantic model | **A YAML file on a stage** — `@ARKEMA_PRICING_DB.MART.SEMANTIC_MODELS/arc_v2_semantic_model.yaml`. **Not** a native `SEMANTIC VIEW` object. | `cortex/arc_semantic_model.yaml`, `cortex/arc_v2_semantic_model.yaml`; stage in `ddl/001_database.sql:62` |
| **Cortex Search** | `DOCS.ARC_MARKET_INTEL` (v1) and `DOCS.ARC_V2_MARKET_INTEL` (v2), over `DOCS.MARKET_NOTES`. Both `TARGET_LAG = '1 day'` on `ARC_COMPUTE_WH`. | `cortex/deploy_cortex.sql`, `cortex/deploy_agent_v2.sql` |
| **Cortex Agent** | `SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_AGENT` (v1) and `ARC_V2_AGENT` (v2). | v1 via Snowsight UI from `cortex/arc_agent.json`; **v2 has real `CREATE AGENT` DDL** in `cortex/deploy_agent_v2.sql` |
| **Agent transport** | REST, not SQL — `POST /api/v2/databases/{db}/schemas/{schema}/agents/{name}:run`, streamed as SSE. | `backend/services/arc_agent_client.py:36` |

**The v2 agent has four tools**, which is the part most likely to be missed in a grants review:

| Tool | Type | Resource |
|---|---|---|
| `data_analyst` | `cortex_analyst_text_to_sql` | the semantic-model YAML on stage |
| `market_intel` | `cortex_search` | `ARKEMA_PRICING_DB.DOCS.ARC_V2_MARKET_INTEL` |
| `run_simulation` | `generic` → **stored procedure** | `ARKEMA_PRICING_DB.MART.SP_NLC2_AGENT_SIMULATE` |
| `run_recommendations` | `generic` → **stored procedure** | `ARKEMA_PRICING_DB.ML.SP_NLC2_AGENT_RECOMMEND` |

So the agent is not read-only: through `run_simulation` and `run_recommendations` it **writes new scenarios and runs the ML pricing engine**. That is a meaningful security property, and `cortex/deploy_agent_v2.sql` ends by granting the agent to `ROLE PUBLIC` — see §7.

### 2.6 ML layer (Variant A only)

975 lines across 5 scripts in `ml/`. Three models:

| Model | Script | Method |
|---|---|---|
| **M1** — own-price elasticity | `02_train_elasticity_M1.py` | Segment-pooled OLS panel, segment × ln(price) interactions, month dummies, customer fixed effects, macro covariates (Henry Hub gas, EU HICP) |
| **M2** — cross-elasticity | `03_train_cross_elasticity_M2.py` | Matrix at Usage-Group grain (T1 Construction vs T2 DIY) |
| **M3** — volume-loss risk | `04_train_volume_risk_M3.py` | PI-event extraction (net price move >2% month-on-month) + XGBoost classifier on 60-day pre/post volume, labelled `volume_loss_60d = 1` if post < 0.9 × prior |
| — | `05_recommend.py` | Constrained-optimum recommendation combining M1 + M3 |

**The scheduling is a stub, and the DDL says so.** `ddl/021_nlc2_ml_tasks.sql` creates 5 tasks (`NLC2_TASK_INGEST_MACRO`, `TRAIN_M1`, `TRAIN_M2`, `TRAIN_M3`, `RECOMMEND`) on a Sunday-evening cron cascade — but each one only calls `ML.SP_NLC2_ML_KICK`, which **inserts a `PENDING` row into `GOV.NLC2_ML_RUN_LOG`** and returns. The header states plainly: *"the actual training scripts run inside SPCS / a Python runner; these tasks call CALL stubs that mark the run started so that an external orchestrator (cron / GitHub Actions / SPCS scheduled job) can pick up."*

**There is no such orchestrator in the repo.** The retrain chain is inert as delivered. This confirms the audit's finding independently.

---

## 3. Deployment topology (SPCS)

Per the signed SOW (M3-01 "SPCS Setup & Deploy") and the repo's own artifacts.

| Element | Value | Source |
|---|---|---|
| Container | Two-stage build: `node:20-alpine` builds the Vite frontend → `python:3.11-slim` runtime with nginx + uvicorn | `deploy/Dockerfile` |
| Port / health | 8080, `/health` readiness probe, Docker `HEALTHCHECK` every 30s | `deploy/Dockerfile`, both service specs |
| Image repository | `ARKEMA_PRICING_DB.SPCS.ARC_IMAGES` | `ddl/001_database.sql`, `deploy/spcs_setup.sql` |
| Compute pool | `ARC_COMPUTE_POOL` — single node, `CPU_X64_S`, auto-suspend 3600s | `deploy/spcs_setup.sql` |
| Warehouse | `ARC_COMPUTE_WH` — **`XSMALL`** in the DDL, auto-suspend 60s | `ddl/001_database.sql` |
| POC service resources | requests 2Gi/1cpu, limits 4Gi/2cpu | `deploy/service_spec.yaml` |
| MVP service resources | requests 1Gi/0.5cpu, limits 2Gi/1cpu | `deploy/service_spec_mvp.yaml` |
| Egress | 8 hosts: 6 OSM/CDN tile hosts + 2 Google Fonts, via `ARC_EXTERNAL_ACCESS` | `deploy/spcs_setup.sql`, `deploy/network_rules.sql` |
| Public endpoint | `public: true` on both specs | both service specs |

### 3.1 The compute pool is already crowded

`service_spec_mvp.yaml` sizes the MVP container to **co-exist** on `ARC_COMPUTE_POOL`, which its comment says is *"a single-node `CPU_X64_S` already running `ARC_SERVICE` (live) and `ARC_SCALE_SERVICE`"* — and warns that *"asking for more than this risks failing to schedule or destabilising the live service."*

That is three services on one small single node. It works in a demo account; **it is not a production topology**, and re-sizing is a Phase 2 item (§10 item 5).

---

## 4. Request flow — a cost simulation, end to end

The core F2/F4 use case, traced through Variant A.

1. User picks a component and a shock % in the UI (`SimulationWizard.tsx`) → `POST /api/v2/nlc2/run`.
2. FastAPI calls `MART.SP_NLC2_RUN_SCENARIO_E2E`.
3. That procedure: shocks the component price → re-rolls BOM cost (`016_nlc2_bom_rollup.sql`) → writes `MART.NLC2_SIMULATED_COST` → recomputes `MART.NLC2_CUSTOMER_MARGIN` via `SP_NLC2_RECOMPUTE_MARGIN` → registers a row in `MART.NLC2_COST_SIMULATION_RUN` with a new `scenario_id`.
4. UI polls `/api/v2/nlc2/scenarios/{id}/kpi` and `/margin` for results.
5. Optionally `/api/v2/nlc2/scenarios/{id}/recommendations` → `ML.SP_NLC2_AGENT_RECOMMEND`, which reads `ML.NLC2_ELASTICITY` + `ML.NLC2_VOLUME_RISK` and writes `MART.NLC2_AI_RECOMMENDATION`.
6. Lineage for any figure: `GOV.NLC2_LINEAGE_TRACE_SCEN` via `/api/v2/nlc2/lineage/{fp_code}`.

The same flow is reachable **from the Cortex Agent** via its `run_simulation` tool, bypassing the UI entirely.

---

## 5. Feature-to-object map

From the current MVP PRD feature set (F1–F14) to where each lives.

| Feature | Variant A object | Variant B object |
|---|---|---|
| F1 Ingestion / DQ | `RAW.NLC2_*`, `012_nlc2_dq_engine.sql`, `GOV.NLC2_DQ_*` | `BRONZE.*`, `UTIL` contract functions, `GOV` check suites |
| F2 Simulation engine | `MART.SP_NLC2_SIMULATE`, `SP_SIMULATE_MOVC` | `GOLD.V_SIMULATED_VC_UNIT` |
| F3 BOM rollup / health score / lineage | `016_nlc2_bom_rollup.sql`, `GOV.NLC2_DQ_HEALTH_SCORE`, `GOV.NLC2_LINEAGE_TRACE` | `GOLD.V_F3_11_HEALTH_SCORE`, `V_F3_FP_HEALTH` |
| F4 Customer margin impact wizard | `MART.NLC2_CUSTOMER_MARGIN` | `GOLD.F4_RUN`, `F4_SCENARIO(_LINE)`, `V_F4_SCENARIO_RESULT`, `V_F4_RUN_LIST` |
| F5 Margin rollup | `MART.NLC2_CUSTOMER_MARGIN` | `GOLD.V_CELL_SALES_LOOKBACK` |
| F6 Recommendations (ML) | `ML.*`, `MART.NLC2_AI_RECOMMENDATION`, `014_nlc2_agent_recommend.sql` | — |
| F8 Ask ARC / GenAI | Cortex Agent + Search + Analyst (§2.5) | **`genai` view exists, `backing: ''` — not wired** |
| F9 Price Index Evolution | `018_nlc2_lineage_scenario.sql`, `/api/v2/nlc2/price-index` | `GOLD.V_F9_INDEX`, `V_F9_CELL_RELATIVE` |
| F10 Performance Bridge | `/api/v2/nlc2/bridge` | `GOLD.V_F10_NODE`, `V_F10_CELL` |
| F11 Price Increase Tracking | `019_nlc2_f11_pi_tracking.sql`, `MART.NLC2_PI_TARGET*` | **not backed** (`pi-tracking`, `pi-targets`, `sales-queue` all `backing: ''`) |
| F12 Component Price Coverage | — | `GOLD.V_F12_IMPACT_RANK`, `V_F12_COLLECTION_TEMPLATE` |
| F13 Procurement History | — | `GOLD.V_F13_COMPONENT_TREND`, `V_F13_PROCUREMENT_HISTORY` |
| F14 Multi-Currency | — | `GOLD.DIM_FX_RATE`, `DIM_FX_RATE_ZCOR/ZERX` |

**Read this table for what it implies:** F12, F13 and F14 exist **only** in Variant B. F6 ML and F8 GenAI exist **only** in Variant A. Neither variant covers the full PRD. That is the crux of §10 item 1.

---

## 6. RBAC and personas

### 6.1 Snowflake roles (Variant A DDL)

`ddl/001_database.sql` creates five roles: `ARC_COMMERCIAL`, `ARC_FPA`, `ARC_PRICING`, `ARC_STEWARD`, and `ARC_APP` (the SPCS service role). **This resolves the "4 vs 5 personas" discrepancy in the project plan: there are 4 human personas plus 1 service role.**

### 6.2 Application personas

| Variant A — `main.py:128` `PERSONA_PERMS` | Permissions |
|---|---|
| `Commercial` | view, scenarios, export |
| `FPA` | view, scenarios, dq, export |
| `PricingAnalyst` | view, scenarios, compare |
| `Steward` | view, dq, lineage, override |

| Variant B — `mvp/lib/types.ts:266` `PERSONAS` | Role | Views allowed |
|---|---|---|
| `cm` | BU Commercial Manager | 11 of 16 |
| `fpa` | FP&A Analyst | 13 of 16 |
| `pa` | Pricing Analyst | 11 of 16 |
| `ds` | Data Steward | 4 of 16 |
| `sr` | Sales Representative | 3 of 16 |
| `dampe` | DAMPE · Procurement | 3 of 16 |

**Variant B's RBAC is materially better built.** Per-persona view allow-lists are enforced in the sidebar, section headers and wizard button; a persona switch that strands a user on a forbidden view redirects to their first allowed one; and queue deep-links are chosen from the persona's own allow-list so they cannot point at a disabled view (MVP-BRIDGE D34, D37).

**But it is still client-side.** See §7.

---

## 7. Security posture — as built

Stated plainly, because these are the findings that drive M3-02.

1. **There is no authentication in either variant.** Persona arrives as a client-supplied `X-ARC-Persona` header (`frontend/src/lib/api.tsx:36`) and `assert_perm` defaults to `"Commercial"` when it is absent (`main.py:138`) — a write-capable persona. Variant B's richer allow-lists are UI-side only and equally bypassable.
2. **`ARC_V2_AGENT` is granted to `ROLE PUBLIC`.** The last line of `cortex/deploy_agent_v2.sql`. Because the agent's tools write scenarios and run the ML engine, this is not a read-only exposure. **Must not be carried into the Arkema account.**
3. **Sharing in Variant B is application metadata, not RBAC** — and the repo says so: *"the cockpit runs under one service role, so `CURRENT_USER` is the same string for every persona; ownership and sharing are keyed on display name"* (D44). `F4_RUN_SHARE` is labelled pilot-grade in both the DDL and the UI. Honest, but not access control.
4. **SQL is built by f-string, not bound parameters**, in the FastAPI tier. Stored procedures bind correctly; the service tier does not.
5. **What is done right:** the SPCS OAuth token is read from `/snowflake/session/token` and never logged; the browser never talks to Snowflake directly; scenario isolation holds at the database level.

---

## 8. Known-good engineering worth preserving

It would be unfair to document only the gaps. Several things here are better than typical prototype quality and should survive productionalization:

- **`docs/MVP-BRIDGE.md` is an exceptional artifact** — 398 lines, a 46-entry drift register (D1–D46) reconciling PRD v2.22 against the delivered data against the implementation, with measured row counts and named check IDs for each. It distinguishes spec defects from *"our own defect"* explicitly. This is the single most valuable document in the repo.
- **Regression guards are named and permanent.** e.g. D17 — `DIV0NULL(x,0)` returns `0`, not NULL, which stamped `0.00` onto 27,787 of 58,432 price cells (48%); fixed with `NULLIF`, with `SA-29`/`SA-30` left in place as permanent guards.
- **`GOV.MVP_PARAMETER`** replaces session variables with a durable, auditable parameter table (D14).
- **`views.ts` `backing` field** makes mock-vs-wired visible in the UI rather than hiding it.
- **The ML statistical machinery is real** — OLS panel with fixed effects, cross-elasticity matrix, XGBoost classifier. It is structurally disconnected from the UI, not fake.

---

## 9. Corrections to earlier documents

Producing this architecture required correcting three things I had previously stated or repeated. Recorded here so the error does not propagate further.

| # | Earlier claim | Actual | Impact |
|---|---|---|---|
| 1 | *"`ARKEMA_PRICING_DB` has no `CREATE DATABASE` in the scripts"* — in `20260915_ProjectPlan.md` §2 and repeated in my `20260918_GrantsNeeded.md` §3 | **False.** `ddl/001_database.sql` contains `CREATE DATABASE IF NOT EXISTS ARKEMA_PRICING_DB`. The audit's finding was about **`ARKEMA_PRICING_DB_SCALE`** — the `scale/` tree, now out of scope. | I over-stated a gap. The grants doc needs this corrected; the DDL is reproducible from empty. |
| 2 | The semantic model is a native `SEMANTIC VIEW` object | It is a **YAML file on the `MART.SEMANTIC_MODELS` stage**. | Already corrected in `20260918_GrantsNeeded.md` §1a/§6a. |
| 3 | Deployment targets `ARKEMA_PRICING_DB` | There are **two** target databases, and `ARKEMA_PRICING_MVP` is described in the repo as the PS handover artifact. | The grants doc currently covers only `ARKEMA_PRICING_DB`. **It needs a second database, or a decision — §10 item 1.** |

Also worth noting: `docs/ARCHITECTURE.md` in the repo is 77 lines and describes only Variant A. It does not mention `ARKEMA_PRICING_MVP`, the medallion layers, or the MVP router. It is not wrong so much as badly out of date — which is why this document was built from the code.

---

## 10. Open architectural questions

Ordered by how much they change the plan.

1. **Which variant are we productionalizing?** This is the decision everything else waits on. Variant B is built to the authoritative PRD v2.22, has 3× the DDL, better RBAC, and is called *"a clean handover artifact for PS"* in its own header — but has **no AI layer**, and F8 GenAI is a customer-confirmed "Must" (Michael THAI, written, Sep 16). Variant A has the AI layer and the ML but is the POC. **The likely answer is "Variant B plus Variant A's AI layer", which is an integration task nobody has scoped.** Needs Rishabh before his cutoff.
2. **Which agent — `ARC_AGENT` or `ARC_V2_AGENT`?** `service_spec.yaml` names v1, but v1 was hand-built in the Snowsight UI and has no reproducible DDL. v2 has DDL and matches the NLC2 pilot. Deploying v1 puts an unreproducible object on the critical path.
3. **The ML retrain chain is inert.** Five tasks insert `PENDING` rows and stop; the external orchestrator they expect does not exist in the repo. Either build it, or run the models manually, or descope — but it cannot ship as-is and be described as scheduled.
4. **Four MVP views have no data** (`pi-tracking`, `pi-targets`, `sales-queue`, `genai`). F11 needs a PI ledger that the delivery does not contain (D38). If Variant B is the target, these are UAT gaps today.
5. **The compute pool is a demo topology** — three services on one `CPU_X64_S` node. Needs re-sizing before UAT, and the warehouse is `XSMALL` in the DDL (I have proposed `SMALL` plus a statement timeout in the grants doc).
6. **Two open FP&A decisions carried in MVP-BRIDGE**, neither ours to close: **DR-17** — `material_pct` measures 85.71% aggregate vs the catalog's ~91%, and it scales the F1.8 baseline that every margin delta depends on, so a ~6pp bias shifts every figure the same way. **DR-18** — 44,540 zero-volume rows carry revenue on credit/rebate doc types, netting −€2.77m (−3.66%) of CORE revenue and deflating price on 8.8% of cells. Both are flagged *"needs FP&A"*. **These are exactly the "numbers are not reliable" findings, already quantified by Rishabh.**

---

*Derived entirely from static reading of `arkema-pricing-ss/arc/` on Sep 18, 2026. No Snowflake account was queried to produce this document, and no objects were created, altered, or dropped.*
