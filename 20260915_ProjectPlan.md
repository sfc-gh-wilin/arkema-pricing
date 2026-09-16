# Arkema Pricing Cockpit — App Productionalization Project Plan

**Date:** September 15, 2026
**Author:** William Lin (App Productionalization workstream owner)
**Status:** DRAFT — for review ahead of App Review meeting with Rishabh Raman (Sep 16 AM)

**Sources reviewed:**
- App source: `arkema-pricing-ss/` (`arc/` and `scale/arc/` trees)
- Andrew Carson's Cortex Code audit: `~/Docs/Snowflake/20260904 Arkema/Andrew/Andrew - Arkema_POC_Pricing/working/audits/`
- Andrew's Slack summary and prep-call transcript (`andrew-slack-msg.md`, `Transcript_Arkema with Andrew.txt`)
- Signed engagement collateral: `ref/Arkema — Snowflake PS TDR Proposal ... August 2026.pptx` (20 slides), `ref/20260909_PrepMeeting_Summary.md`, `ref/20260909_PrepMeeting_Transcript.txt`

No Snowflake account was queried for this plan (local-file research only, per instruction).

---

## Glossary

Plain-language definitions of terms used in this doc. Skip if already familiar.

| Term | Meaning |
|---|---|
| **BOM** | Bill of Materials — the recipe for a finished product: which raw materials/components go into it, and in what quantity. Used here to calculate the true material cost of a product (`arc/ddl` files, `app_requirements/.../data-guide.md`). |
| **ZFSC07** | An SAP-sourced table name (custom SAP report code, prefix `Z` = customer-built). It holds **standard cost** data per material/plant/period. In this app it lands as `RAW.NLC2_ZFSC07` and feeds the "standard cost" side of margin calculations. Not a generic term — it's specific to Arkema's SAP system. |
| **NLC2** | Internal project/dataset code name used throughout table and file names (e.g. `NLC2_SALES`, `NLC2_BOM_FLAT`). Refers to the specific product line/business unit this MVP was built for (Bostik C&C Silicones). |
| **FP** | Finished Product — the sellable end-item (as opposed to a raw material or component). |
| **MOVC** | Moving Cost / cost simulation logic — the stored procedure `SP_SIMULATE_MOVC` computes simulated material cost changes when a price scenario runs. |
| **DAMPE** | Another SAP-sourced table/feed — holds **future/forecast price** data (as opposed to ZFSC07's standard cost). Used as the top tier of the price-source waterfall. |
| **WAC** | Weighted Average Cost — a blended cost figure (e.g. `FG_WAC` = Finished Good Weighted Average Cost). |
| **MARM** | An SAP table for Unit of Measure (UOM) conversions (e.g. converting KG to PC, or TO to KG) — needed because different source files use different units for the same material. |
| **RBAC** | Role-Based Access Control — who is allowed to see or do what, based on their assigned role (here, one of the app's "personas": e.g. Commercial, FP&A, Pricing, Steward). |
| **SSO/SCIM** | Single Sign-On / System for Cross-domain Identity Management — lets Arkema employees log in with their existing corporate credentials, and lets Snowflake automatically provision/deprovision their accounts. |
| **UAT** | User Acceptance Testing — the customer (Arkema) tests the finished app/model against real use cases and formally signs off that it works. |
| **SPCS** | Snowpark Container Services — Snowflake's way of running a Docker container (like this app's FastAPI backend + React frontend) directly inside Snowflake's infrastructure. This is the current, contracted deployment target. |
| **SAR** | Snowflake App Runtime — a newer, different Snowflake mechanism for running web apps. Not what this app is built for today, and not in the signed contract (see §6). |
| **Cortex Analyst / Cortex Search / Cortex Agent** | Snowflake's built-in AI features: Cortex Analyst answers natural-language questions against structured data; Cortex Search does document/text search; Cortex Agent is the orchestrator that combines both to power the app's "Ask ARC" chat feature. |
| **Elasticity (β / beta)** | An economics term measuring how much sales volume changes when price changes (e.g. β = -1.0 means a 1% price increase causes roughly a 1% volume drop). Used by the ML model to recommend prices without losing too many customers. |
| **LOE** | Level of Effort — the estimated hours/days budgeted for a piece of work in the contract. |
| **DRI** | Directly Responsible Individual — the one person accountable for a specific task or decision. |
| **RACI** | A responsibility-assignment chart: **R**esponsible (does the work), **A**ccountable (owns the outcome), **C**onsulted (asked for input), **I**nformed (kept up to date). |

---

## 1. Engagement Context

| Item | Detail |
|---|---|
| Customer | Arkema Inc. — global specialty chemicals (~€9.5B revenue), on-demand/trial Snowflake account pending CAP1 |
| App | "Pricing Cockpit" / ARC (Arkema Resilience Cockpit) — pricing simulation + margin/AI-recommendation tool, built by **Rishabh Raman** (Snowflake Industry team) as a prototype, not by pHData (pHData was going to deploy it, then backed out) |
| Contract vehicle | Snowflake PS TDR Proposal, "Platform, ML & App Productionalization," August 2026 — 8-week engagement, 3 parallel workstreams |
| My workstream | **App Productionalization** — deploy the app as-is to production, connect to production data, run 2-week UAT. **No new features, no bug fixes, no code modifications are contractually in scope.** |
| Other workstreams | Platform Foundation (Wole Babalola) — RBAC, SSO/SCIM, Snowpipe for 12 tables. ML Productionalization (Sharanya Krishnamurthi + Luis Villavicencio) — port 3 ML models to Model Registry. |
| SDM | Brendan Owens (temporary; full-time SCM TBD) |
| My start date | September 14, 2026 (confirmed in prep call) |
| Key near-term event | **App walkthrough with Rishabh Raman — tomorrow morning, Sep 16.** Rishabh stops editing the app **Sep 21**; after that it's fully handed to Services with no formal KT. |
| Budgeted LOE (App stream) | M1‑03 App Discovery: 12h · M3 App Productionalization: 64h · M5 Testing/UAT (App portion): 40h · M6 Launch Support: 20h → **~136h app-specific**, inside a ~140h total App SA allocation over 8 weeks (Slide 9: App SA 140h across weeks 2–9) |
| Independent effort audit | Andrew Carson ran a 5-pass Cortex Code audit against the repo (no live Snowflake). App-tier + service-tier findings alone total **145.5 person-days** of remediation work if done exhaustively; a compressed "fastest path" is **49–54 days**, which alone consumes 77–84% of the full 64-day/three-workstream engagement budget. |

**Bottom line entering the walkthrough:** the contracted scope ("deploy as-is") and the audited state of the code (numbers are wrong, no auth, blocking single-user runtime) are in tension. This needs to be resolved explicitly with Rishabh/Brendan before committing to a deployment date — see §5 Risks and §7 Questions for Rishabh.

---

## 2. Current App Architecture (as documented in `arkema-pricing-ss/`)

| Tier | Stack | Notes |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite + Tailwind CSS, Leaflet (map), Vega-Lite/Vega/Recharts | 16 screens exist (docs say 12); persona switcher is a client-side `<select>` |
| Backend | FastAPI (Python), 44 routes | Runs via `snow` CLI locally, or containerized for SPCS in production |
| Data layer | Snowflake DB `ARKEMA_PRICING_DB` (legacy/current) and `ARKEMA_PRICING_DB_SCALE` (scale variant target, **never created by any script**); schemas RAW → ATOMIC → MART → ML → GOV → DOCS → SPCS |
| AI layer | Cortex Analyst (semantic model), Cortex Search, Cortex Agent (`ARC_AGENT`, currently granted to `ROLE PUBLIC`) |
| **Deployment target per contract & repo** | **Snowpark Container Services (SPCS)** — confirmed by `arc/deploy/service_spec.yaml`, `arc/deploy/deploy.sh` (build → push to image repo → `CREATE/ALTER SERVICE`), and the signed proposal's RACI ("Implementation of existing prototype (deployed to SPCS)", M3-01 "SPCS Setup & Deploy") |
| Repo structure | Two divergent trees: `arc/` (root, has newer feature DDL: F11 PI tracking, lineage/scenario, price-ref dedup, a separate MVP cockpit variant) and `scale/arc/` (performance variant targeting 50K materials × 150K customers × 24mo, adds clustering/search-optimization/dynamic tables) — **not a superset/subset of each other; they diverged** |
| Features (PRD v2.22) | F1 Ingestion/DQ · F2 Simulation engine · F3 BOM rollup/health score/lineage · F4 Customer margin impact wizard · F5 Margin rollup · F6 Recommendations · F7 Reporting · F8 Ask ARC (GenAI) · F9 Price Index Evolution · F10 Performance Bridge · F11 Price Increase Tracking · F12 Component Price Coverage · F13 Procurement History · F14 Multi-Currency |

**Confirmed for the record:** the app was designed and built for **SPCS**, not Streamlit-in-Snowflake, not Snowflake App Runtime (SAR). The Docker image, service spec, compute pool, and image repository setup already exist and match SPCS conventions.

---

## 3. Independent Audit Findings (Andrew Carson, Cortex Code 5-pass audit)

Summarized here because they materially affect any deployment-architecture decision. Andrew's own caveat: *"This app is very, very good... one of the best prototypes I've ever seen"* — findings below describe gaps expected in a fast-built prototype, not a condemnation of the work.

### 3.1 Blockers (must be resolved before any real-user production traffic)
1. **No authentication anywhere.** Persona is a client-controlled `X-ARC-Persona` header; 37 of 44 routes have no server-side check; when the header is absent it **fails open to `Commercial`**, a write-capable persona — an anonymous request can trigger a paid-for simulation run or a governance write.
2. **Zero parameterized SQL.** 53 f-string-built SQL statements; unescaped in at least 15 call sites in `arc_service.py` — SQL injection is reachable from a public endpoint.
3. **Numbers are not reliable.** Andrew: *"Of ~14 numbers shown to a user, none is provably correct."* Concrete defects:
   - AI "recommendation" is 3 hardcoded policy constants (+10/+8/+5%); the elasticity gate (`beta < -1.0`) can never fire because beta is clamped to exactly `-1.0` — the ML model never actually influences price.
   - Headline margin-impact figure compares a full standard cost to a materials-only variable cost (mismatched bases).
   - No currency conversion exists; currency is dropped before `SUM(sales)`.
4. **Maximum safe concurrent users today: one** — a correctness limit, not a performance one. All routes are `async def` wrapping blocking calls; 2 uvicorn workers serve one request at a time each.
5. **BOM fan-out risk on real data.** Latent with synthetic data (no `ZROH` rows); with real ZFSC07 data the scan can multiply **up to 144×**, and with no statement timeout a run doesn't fail — it burns credits for hours with no cancel path.
6. **The deployment cannot be reproduced from the repo.** `ARKEMA_PRICING_DB_SCALE` (the scale target) has no `CREATE DATABASE` anywhere; the deployed app is served entirely by **synthetic data** (the real-data loader targets the legacy DB, not the one the app reads).
7. **Error handling doesn't exist.** Failed queries return `[]` with HTTP 200; a total backend outage renders a "settled," healthy dashboard reading €0.00M at risk; 3 write endpoints return hardcoded success regardless of outcome.

### 3.2 Effort vs. budget reality check
- Individually priced, Passes 1–4 (app tier, service tier, data tier, concurrency) total **280 person-days**.
- Andrew's own "fastest path" re-sequencing compresses this to **49–54 days**, still **77–84% of the entire 64-day, three-workstream budget**, before deployment fixes (M3-01, "underpriced by ~15 days" per audit) or building the ingestion pipeline (doesn't exist yet, ~10 days).
- Contract explicitly states: *"deploy as-is; no new features, no bug fixes, no code modifications."* This is very hard to reconcile with the audit — the app cannot go live against real SAP/production data without at minimum entitlement enforcement and error-handling fixes, both of which are "modifications" by the contract's own definition.

### 3.3 What's genuinely fine (don't rebuild it)
- Scenario isolation holds at the DB level for different scenario runs.
- SPCS OAuth secret handling is correct and never logged.
- Stored procedures bind parameters properly — SQL injection risk is entirely at the FastAPI service tier, not the procs.
- Browser never talks to Snowflake directly.
- GOV/DQ schema and objects are well-built.
- ML statistical machinery (OLS, XGBoost, SUR) is real and sound — it's just structurally disconnected from what the UI shows.

---

## 4. Proposed Phased Plan (App Productionalization workstream)

This maps the contracted LOE milestones (M1, M3, M5, M6 — App columns) to concrete actions, informed by the audit.

### Phase 0 — Pre-Kickoff Alignment (now → Sep 16 walkthrough)
- [ ] Attend Sep 16 walkthrough with Rishabh Raman. Bring the audit findings and the open questions in §7.
- [ ] Confirm with Brendan/Rishabh: which repo tree is authoritative — `arc/` or `scale/arc/`? (They have diverged; deploying the wrong one wastes the 8-week clock.)
- [ ] Confirm which database is the real deployment target (`ARKEMA_PRICING_DB` vs `ARKEMA_PRICING_DB_SCALE`) — critical since neither is fully reproducible from the current scripts.
- [ ] Get the SPCS-vs-SAR question resolved in writing (see §6) before any deployment work starts.

### Phase 1 — App Discovery (M1-03, budgeted 12h)
- [ ] Pin the baseline commit/tag to deploy (resolve `arc/` vs `scale/arc/` divergence first).
- [ ] Stand up local dev environment; run the app once end-to-end locally against a fresh, real DB build (Andrew's "1-hour experiment #1" — never done yet).
- [ ] Walk all screens (confirmed 16, not the documented 12) and confirm actual persona count (confirmed 4, not the documented 5).
- [ ] Map each screen's API calls to the confirmed production consumption layer (MART) that Platform workstream will deliver.
- [ ] Confirm write paths (scenario save, override, governance actions) and the export column set.
- [ ] Inspect real ZFSC07/customer-segment extracts once Platform lands sample data (Andrew's "experiment #2") to determine if the 144× BOM fan-out is real exposure or theoretical.

### Phase 2 — App Productionalization (M3, budgeted 64h)
Per contract LOE line items, re-sequenced by the audit's chokepoint analysis where it doesn't expand scope:

| LOE item | Budgeted | Action |
|---|---|---|
| M3-01 SPCS Setup & Deploy | 8h | Image repo, compute pool, service spec, pinned image, repeatable deploy + rollback script (currently `scale/arc/deploy.sh` is a stub `TODO`) |
| M3-02 RBAC Wiring | 0h* (audit: needs 11h min) | **What "M3-02" is:** LOE line item #2 under Milestone 3 (App Productionalization) in the signed proposal deck (Slide 16, "LOE Breakdown M3"). Its description there: *"Bind 5 roles to server-side authorization. Role drives screen access. Remove browser-claimed persona. Audit events."* This is the exact fix for the audit's #1 blocker (§3.1, item 1: persona is just a client header today, e.g. `App.tsx`, `arc/backend/api/main.py` `PERSONA_PERMS`). **Flag as change-order risk — budgeted at 0 hours in the contract.** |
| M3-04 Write Path Hardening | 16h | Saved scenarios, price-increase assumptions, transaction boundaries, idempotent re-runs |
| M3-05 Error Handling & Observability | 12h | Stop returning empty-as-success on failed queries; add loading/empty/error/unauthorized UI states; parameterize the highest-risk SQL call sites |
| Misc. Fixes | 28h | Contingency bucket — prioritize by risk, not by convenience |

**Concern:** M3-02 (entitlement enforcement) is the #1 blocker in the audit and is budgeted at 0 hours. This is the single highest-risk line item in the entire App workstream and needs explicit sign-off from Brendan/Rishabh on how it will be resourced.

### Phase 3 — Integrated Testing & UAT (M5, App portion budgeted 40h)
- [ ] Build test plan from acceptance criteria; seed test users across the (confirmed) 4–5 roles.
- [ ] End-to-end testing: app → API → consumption layer → ML output, with per-role access checks and FP&A calculation spot-checks against known-good numbers.
- [ ] Support Arkema-driven 2-week UAT; Snowflake reverse-shadows testing in parallel.
- [ ] Track UAT defects against the audit's known-issue list so nothing already identified gets "rediscovered" mid-UAT.

### Phase 4 — Launch Support (M6, budgeted 20h)
- [ ] Final production deploy at end of Week 8.
- [ ] Smoke test all screens across all roles.
- [ ] Verify the daily refresh ran and models scored (confirm the ML retrain task chain actually executes — audit found it's currently inert, only inserting `PENDING` log rows).
- [ ] Deliver admin runbook (deploy, rollback, refresh failure, permission changes) and one KT session.
- [ ] Hand off named support owner and open issue list.

---

## 5. Key Risks (from contract + audit, consolidated)

| Risk | Source | Impact | Mitigation |
|---|---|---|---|
| Entitlement/auth is unbudgeted but is the #1 blocker | Audit F-2.01–03, M3-02 = 0h | High — cannot go live with an anonymous write path | Raise explicitly with Brendan before Week 3; likely change-order candidate |
| "Deploy as-is" contract clause vs. numbers being provably wrong | Contract Slide 4/10/11 vs. Audit Pass 3 | High — going live with incorrect pricing/margin numbers is a business risk, not just a technical one | Get explicit written sign-off from Arkema on what "as-is" covers once they see the specific wrong-number findings; do not silently fix or silently ship |
| Repo has diverged (`arc/` vs `scale/arc/`), deployment target DB never created | Audit lineage.md, Pass 0 | High — wrong choice wastes days of the 8-week clock | Resolve in Phase 0 with Rishabh before any build work |
| 144× BOM fan-out with real data, no timeout/cancel | Audit Pass 4 | High once real data lands | Run "experiment #2" early; if confirmed, raise as explicit change order (13 days, per audit) rather than absorbing silently |
| Synchronous blocking runs cap concurrency at 1 correct user | Audit Pass 4 | Medium-High for multi-user UAT | Decide interim mitigation (serialize + queue) vs. full async (9.5 days, change-order candidate) before UAT starts |
| Production data schema may deviate from mock/dev schema | Contract Slide 12 (flagged High in the SOW itself) | High | Pre-kickoff schema validation is a contractual pre-req; do not assume match |
| No formal KT; Rishabh stops editing Sep 21 | Prep meeting notes | Medium | Front-load discovery/walkthrough time before Sep 21; document everything discovered, since self-serve from the repo is the only fallback |
| SPCS vs. SAR architecture decision unresolved (see §6) | This plan's own note vs. signed contract | High if changed without sign-off | Resolve before Phase 2 starts — do not begin re-platforming work on assumption alone |

---

## 6. Note on SAR (Snowflake App Runtime) — Flagged for Discussion, Not Yet a Decision

You noted a preference to push the app to **SAR (Snowflake App Runtime)**. Flagging clearly before the Sep 16 walkthrough:

- The **signed TDR proposal** scopes deployment explicitly to **SPCS**: the RACI table lists "Implementation of existing prototype (**deployed to SPCS**)," and LOE item M3-01 is literally "**SPCS** Setup & Deploy." The Delivery Assumptions slide also states *"App and ML deployed as-is; no new features, integrations, or **rearchitecture** in scope."*
- The **existing app repo is already built for SPCS**: Docker image, `service_spec.yaml`, compute pool, and image-repo scripts all assume SPCS. There is no SAR-oriented code (`app.yml`, Next.js/App-Runtime scaffolding) anywhere in the repo.
- Moving to SAR would be a genuine re-platform: different deployment manifest, different runtime model for the FastAPI backend + React frontend, and a change to what "deploy as-is" contractually means. Per the SOW's own assumptions, this reads as a **rearchitecture** and would need a **change order**, not something to fold into the current 64-hour App budget.
- **Recommendation:** raise the SAR idea with Rishabh/Brendan as an explicit option at tomorrow's walkthrough, with the trade-off stated plainly (why SAR, what it buys, what it costs against the fixed 8-week/64-hour budget) — rather than assuming it as the plan. I have not built this plan around a SAR migration; it stays scoped to SPCS per contract until you get sign-off.

---

## 7. Questions for Rishabh Raman / Brendan Owens (Sep 16 Walkthrough)

1. Which tree is authoritative for deployment — `arc/` or `scale/arc/`? They have diverged and neither fully matches its own documentation.
2. Which database is the real production target — `ARKEMA_PRICING_DB` or `ARKEMA_PRICING_DB_SCALE`? Neither is fully created by the current scripts.
3. Has this app ever actually been run end-to-end against a freshly built (non-synthetic) database? (Andrew's audit suggests no.)
4. Given M3-02 (entitlement enforcement) is budgeted at 0 hours but is the top blocker in the audit — how should this be resourced? Is a change order expected here?
5. Does Arkema (the customer) know the specific pricing/margin numbers currently shown by the app are not reliable (audit Pass 3)? Is "deploy as-is" intended to cover shipping those numbers unchanged, or is there tolerance to fix the clearly-broken arithmetic (e.g., the beta-clamp bug that disables the ML model's influence entirely — see code location below) within "productionalization" rather than "new feature"?
   - **Where to find this in the code:** `arkema-pricing-ss/arc/ddl/014_nlc2_agent_recommend.sql`. The elasticity gate is `if beta < -1.0:` (line 48), right below where `beta` is clamped to a floor of exactly `-1.0` by the `shrink_beta()` function (line 33). Because the clamp's output can never be *less than* `-1.0`, the `<` condition on line 48 can never be true, so the code path that uses the ML elasticity (`p_unc`, line 49) never executes.
   - **Where the "not reliable" numbers surface in the UI:** the margin/recommendation screens driven by `arc/backend/services/nlc2_service.py` (reads `ML.NLC2_ELASTICITY`, line ~396) and the stored procedure `SP_BUILD_RECOMMENDATIONS` referenced in `arc/docs/ARCHITECTURE.md`.
6. Is real ZFSC07 (BOM) data available yet to test the 144× fan-out risk before UAT starts?
7. Confirm target go-live date and whether Week 5 Platform Foundation milestone (Snowpipe for 12 tables) is on track — App workstream cannot connect to production data until that lands.
8. Raise SAR vs. SPCS explicitly (§6) and get a documented decision either way.

---

## 8. Open Items / My Concerns Summary

- **Biggest concern:** the contract's "deploy as-is, no code modifications" framing does not obviously accommodate the audit's top findings (no auth, wrong numbers, blocking runtime). This gap needs to be surfaced to Brendan/Rishabh explicitly and in writing before committing to the Week 5/Week 8 milestone dates — not discovered mid-UAT.
- **Second concern:** repo divergence (`arc/` vs `scale/arc/`) means "the app" is not a single unambiguous artifact right now. This must be resolved in Phase 0, or later phases risk building against the wrong target.
- **Third concern:** SAR is not yet a scoped or budgeted deployment target under the signed SOW. Treating it as a foregone conclusion risks a scope dispute; treating it as a discussion item at tomorrow's walkthrough does not.
- Everything above is sourced directly from the repo, the audit, and the signed proposal deck — no numbers or findings in this plan were invented.

---

*This is a local planning document only. No Snowflake database objects were created, altered, or queried in producing this plan.*
