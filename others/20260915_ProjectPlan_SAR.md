# SAR Plan — ARCHIVED (superseded, kept for reference)

> **Status: not proceeding.** Decision Sep 18, 2026: the app stays on **SPCS**. The blocker is that Snowflake App Runtime runs **Node.js only** today, and the ARC backend is Python (FastAPI). The active plan is `../20260915_ProjectPlan.md` (Rev 2, SPCS). Grants: `../20260918_GrantsNeeded.md`.
>
> This document is retained because the SAR analysis stays valid and will be worth revisiting once SAR supports Python.

---

## Slack message to Daniel — ready to post

Short and direct. Copy the block below.

--- My version

Hi Daniel. Making this App a SAR would be super!
After looking more into this App, SAR needs to be hold off maybe to next phase.
This App's backend is Python FastAPI with ~4,400 lines, 93 routes.
SAR run Node.js only, Python is on the roadmap but not ready yet.

---

Hi Daniel — looked into SAR for the Pricing Cockpit. **Recommendation: stay on SPCS for this engagement.**

The blocker is simple: **SAR only runs Node.js today** (Python is on the roadmap, not shipped). Our backend is FastAPI — ~4,400 lines, 93 routes. SAR means rewriting the whole service tier, not repackaging it. With mid-October UAT, and Andrew's audit finding that none of the app's numbers is provably correct, we'd be rebuilding pricing logic with nothing reliable to validate against. Too much risk for the window.

**To be clear, SAR is the better platform** — no container to maintain, managed scaling (fixes our single-user ceiling), much cleaner deploys, and SSO identity that solves half the auth problem. Worth doing, just not inside this engagement. **Suggest we bank it as a post-go-live phase 2** and revisit when SAR supports Python. Full analysis is written up so we're not starting over.

One thing I do need from you: **I'm blocked on Arkema account access.** My role has zero privileges — I can't create a database or warehouse, and `ARKEMA_PRICING_DB` doesn't exist yet. I've prepared a step-by-step doc for their IT team. Can you help me get an ACCOUNTADMIN owner and a date? This is holding up everything, including discovery.

Happy to walk through the SAR detail whenever useful.

---

*End of Slack message. Everything below is the full Rev 3 SAR plan, retained for reference.*

---

# Arkema Pricing Cockpit — App Productionalization Project Plan (SAR variant — ARCHIVED)

**Created:** September 15, 2026
**Last updated:** September 18, 2026 (Rev 3 — **deployment target changed to SAR** per Daniel Sandler; account verification added)
**Author:** William Lin (App Productionalization workstream owner)
**Status:** ARCHIVED Sep 18, 2026 — not proceeding. See the banner at the top of this file.

**Sources reviewed:**
- App source: `arkema-pricing-ss/` (`arc/` and `scale/arc/` trees)
- Andrew Carson's Cortex Code audit: `~/Docs/Snowflake/20260904 Arkema/Andrew/Andrew - Arkema_POC_Pricing/working/audits/`
- Andrew's Slack summary and prep-call transcript (`andrew-slack-msg.md`, `Transcript_Arkema with Andrew.txt`)
- Signed engagement collateral: `ref/Arkema — Snowflake PS TDR Proposal ... August 2026.pptx` (20 slides), `ref/20260909_PrepMeeting_Summary.md`, `ref/20260909_PrepMeeting_Transcript.txt`
- Added in Rev 2: `ref/20260916 App Review's transcript.txt` (Sep 16 App Review with Rishabh Raman), `ref/20260916_ChaseEmailAndReply.md` (Chase Growney → Michael THAI scope thread), `ref/20260915_MeetingNoteFromWole.md` (Wole Babalola platform-foundation update)
- **New in Rev 3:** direction from **Daniel Sandler** to deliver the app on **SAR**; Snowflake App Runtime product documentation (privileges, limitations, account-admin setup, `app.yml`, access control); first read-only verification of the Arkema account `BR68104`, captured in `20260918_GrantsNeeded.md`

Rev 1 and Rev 2 were local-file research only. **Rev 3 is the first revision informed by the actual Arkema account** — account `BR68104`, region `AWS_EU_WEST_1`, inspected read-only on Sep 18 via `SHOW` commands. No objects were created, altered, or dropped.

---

## 0. What Changed in Rev 3 (read this first)

**One decision dominates this revision: Daniel Sandler has directed that the app be delivered on Snowflake App Runtime (SAR), not SPCS.** Rev 2 §6 had closed this in favour of SPCS. That decision is now reversed, and this plan is rebuilt around SAR. §6 has been rewritten accordingly.

I have accepted the direction and replanned. What follows is not an argument against it — it is the cost of it, stated once, so it can be resourced rather than discovered in October.

| # | Change | Source | Effect on plan |
|---|---|---|---|
| A | **Deployment target is now SAR.** Application Service object, `snow app deploy`, `app.yml` manifest. SPCS artifacts in the repo (`Dockerfile`, `service_spec.yaml`, `spcs_setup.sql`, compute pool, image repository) are **no longer the delivery path**. | Daniel Sandler, Sep 18 | **§6 rewritten. Phase 2 re-scoped — see §4.** |
| B | **SAR runs Node.js only. The app's Python backend cannot be deployed to it.** Current SAR limitations: deployable projects are Node.js (typically Next.js); **Python support is documented as "planned", not available.** The app is a FastAPI backend of ~4,400 LOC across 5 route modules with 93 route decorators. | SAR limitations doc + repo inspection | **This is a backend rewrite, not a repackaging.** Largest single scope change in the engagement. See §6.2 and §5. |
| C | **SAR is unavailable on trial accounts.** Both the limitations page and `CREATE APPLICATION SERVICE` state this; account-admin setup lists a paid account as a prerequisite. Arkema is recorded as on-demand/trial pending CAP1. | SAR docs + §1 | **Hard gate. If `BR68104` is still a trial, no SAR deploy is possible at all.** Must be confirmed before any other SAR work. |
| D | **Account verified for the first time — I have zero privileges.** `SHOW GRANTS TO ROLE "EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` returns **0 rows**. `ARKEMA_PRICING_DB` does not exist; the only visible warehouse is `SYSTEM$STREAMLIT_NOTEBOOK_WH`; all four `DEFAULT_SNOWFLAKE_APPS_*` account parameters are empty. | Read-only inspection, Sep 18 | **Everything is blocked on an `ACCOUNTADMIN`.** Full grant request in `20260918_GrantsNeeded.md`. |
| E | **App Development Setup has never been run in this account.** With the defaults empty, any deploy resolves to my personal database `USER$A6347057@ARKEMA.COM`, where `GRANT USAGE ON APPLICATION SERVICE` is **not supported**. | Account parameters, Sep 18 | **Michael's pre-UAT early access is impossible until an admin runs setup.** |
| F | **Region is fine.** `AWS_EU_WEST_1` — AWS commercial. SAR is available in AWS/Azure/GCP commercial regions (not government). | SAR limitations doc | One risk retired: no region blocker. |

### Carried forward from Rev 2 (still true)

| # | Item | Status |
|---|---|---|
| 1 | **`arc/` is the authoritative tree**; `scale/` was a POC scalability demo, to be removed by Rishabh. | Firm |
| 2 | **ML scope is contradictory across SOW / PRD / customer** — Rishabh says "zero machine learning" in the MVP PRD; SOW was scoped with ML; Michael says keep POC ML *if cheap*, F6.1 alone acceptable. | **Open — Chase owns written closure** |
| 3 | **GenAI / Cortex Agent is confirmed IN scope.** Michael: *"Cortex agent and Cowork must be included."* F8.1–F8.7 all "Must." | Firm |
| 4 | **UAT is mid-October**, 2 weeks, Arkema-driven. Pre-UAT early access for Michael TBD. | Firm-ish |
| 5 | **Customer is not Snowflake-ready.** Sources confirmed CSV; S3 bucket still being configured, no readiness date. Wole to build storage integration → stage → file format. | My hard dependency |

**And the most important non-technical finding, unchanged:** **no Arkema business user has ever seen or used this app.** Rishabh openly doubted its usability — *"who is it useful for? Maybe Michael"* — and noted the real end users are Excel users, not analysts. The app is, in his words, *"all UI"*. A functionally correct deployment can still fail UAT on usability. A platform change does not touch this risk; it only reduces the time available to address it.

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
| **SPCS** | Snowpark Container Services — Snowflake's way of running a Docker container (like this app's FastAPI backend + React frontend) directly inside Snowflake's infrastructure. This is what the existing repo was built for, and what the signed contract names. **Superseded as the delivery target in Rev 3.** |
| **SAR** | Snowflake App Runtime — a newer Snowflake mechanism for running web apps. You hand Snowflake your source; Snowflake builds it remotely and runs it as an **`APPLICATION SERVICE`** object, with a stable URL, managed scaling (`min_instances`/`max_instances`), and no compute pool or Docker image for you to manage. **This is now the delivery target (Daniel Sandler, Sep 18).** Its key constraint: it runs **Node.js only** today. |
| **Application Service** | The Snowflake object that represents a deployed SAR app (`CREATE APPLICATION SERVICE`). Distinct from an SPCS `SERVICE` — the standard SPCS commands do not work on it. Its **execution role is fixed at creation and cannot be changed**. |
| **`app.yml`** | The SAR deployment manifest — declares the app name, destination database/schema, query warehouse, instance counts, environment variables, secrets, and external access integrations. Written by `snow app setup`. Replaces `service_spec.yaml` in the new path. |
| **Personal database** | A per-user database (`USER$<login>`) that SAR deploys into by default *until an account admin runs App Development Setup*. Apps there work, but **cannot be shared with any other role** — which makes them useless for UAT. |
| **Caller's rights / owner's rights** | The two ways a SAR app can query Snowflake. **Owner's rights:** every query runs as the app's execution role, so all users see whatever that role can see. **Caller's rights:** queries run as the signed-in user, bounded by "caller grants" given to the execution role. |
| **Cortex Analyst / Cortex Search / Cortex Agent** | Snowflake's built-in AI features: Cortex Analyst answers natural-language questions against structured data; Cortex Search does document/text search; Cortex Agent is the orchestrator that combines both to power the app's "Ask ARC" chat feature. |
| **Elasticity (β / beta)** | An economics term measuring how much sales volume changes when price changes (e.g. β = -1.0 means a 1% price increase causes roughly a 1% volume drop). Used by the ML model to recommend prices without losing too many customers. |
| **LOE** | Level of Effort — the estimated hours/days budgeted for a piece of work in the contract. |
| **DRI** | Directly Responsible Individual — the one person accountable for a specific task or decision. |
| **RACI** | A responsibility-assignment chart: **R**esponsible (does the work), **A**ccountable (owns the outcome), **C**onsulted (asked for input), **I**nformed (kept up to date). |

---

## 1. Engagement Context

| Item | Detail |
|---|---|
| Customer | Arkema Inc. — global specialty chemicals (~€9.5B revenue), on-demand/trial Snowflake account pending CAP1. **Account verified Sep 18: `BR68104`, region `AWS_EU_WEST_1`. Paid-vs-trial status still unconfirmed and is now a hard gate — see §6.3.** |
| App | "Pricing Cockpit" / ARC (Arkema Resilience Cockpit) — pricing simulation + margin/AI-recommendation tool, built by **Rishabh Raman** (Snowflake Industry team) as a prototype, not by pHData (pHData was going to deploy it, then backed out) |
| Contract vehicle | Snowflake PS TDR Proposal, "Platform, ML & App Productionalization," August 2026 — 8-week engagement, 3 parallel workstreams |
| My workstream | **App Productionalization** — deploy the app as-is to production, connect to production data, run 2-week UAT. **No new features, no bug fixes, no code modifications are contractually in scope.** |
| Other workstreams | Platform Foundation (Wole Babalola) — RBAC, SSO/SCIM, Snowpipe for 12 tables. ML Productionalization (Sharanya Krishnamurthi + Luis Villavicencio) — port 3 ML models to Model Registry. |
| SDM | Brendan Owens (temporary; full-time SCM TBD) |
| My start date | September 14, 2026 (confirmed in prep call) |
| Key near-term event | Sep 16 and Sep 17 App meetings held (see §0). **Sep 18: Daniel Sandler directs delivery on SAR.** Rishabh's editing cutoff was stated as **Sep 21** in the Sep 9 prep meeting but was *not* restated on Sep 16 — confirm with Brendan. No formal KT either way. |
| Target UAT | **Mid-October**, 2 weeks, Arkema-driven (confirmed Sep 16) |
| Budgeted LOE (App stream) | M1‑03 App Discovery: 12h · M3 App Productionalization: 64h · M5 Testing/UAT (App portion): 40h · M6 Launch Support: 20h → **~136h app-specific**, inside a ~140h total App SA allocation over 8 weeks (Slide 9: App SA 140h across weeks 2–9). **The SAR backend rewrite (§6.2) does not fit inside this and is a change-order item.** |
| Independent effort audit | Andrew Carson ran a 5-pass Cortex Code audit against the repo (no live Snowflake). App-tier + service-tier findings alone total **145.5 person-days** of remediation work if done exhaustively; a compressed "fastest path" is **49–54 days**, which alone consumes 77–84% of the full 64-day/three-workstream engagement budget. |

**Bottom line as of Rev 3:** the contracted scope ("deploy as-is, no rearchitecture") was already in tension with the audited state of the code (numbers are wrong, no auth, blocking single-user runtime). **The move to SAR adds a third source of tension, because SAR cannot run the app's Python backend at all** — so "as-is" is no longer achievable on the chosen platform by definition, not by judgement. This needs to be priced, not absorbed. See §5 Risks, §6 and §7 Questions.

---

## 2. Current App Architecture (as documented in `arkema-pricing-ss/`)

| Tier | Stack | Notes |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite + Tailwind CSS, Leaflet (map), Vega-Lite/Vega/Recharts | 16 screens exist (docs say 12); persona switcher is a client-side `<select>`. **The React/TS tier ports to SAR relatively cleanly** — it is already TypeScript and npm-based. |
| Backend | FastAPI (Python) — **~4,400 LOC across 5 route modules and 3 service modules, 93 route decorators** (`api/main.py`, `api/mvp_routes.py`, `api/mvp_bom_routes.py`, `api/mvp_f9_routes.py`, `api/mvp_f10_routes.py`; `services/arc_service.py`, `services/nlc2_service.py`, `services/arc_agent_client.py`) | Runs via `snow` CLI locally, or containerized for SPCS. **SAR cannot run this tier — Node.js only.** This is the rewrite in §6.2. (Earlier revisions of this plan said "44 routes"; the decorator count in the repo is 93, so treat 44 as the documented figure and 93 as the measured one.) |
| Data layer | Snowflake DB `ARKEMA_PRICING_DB` — the target, since `scale/` (and with it `ARKEMA_PRICING_DB_SCALE`) is out of scope as of Sep 16; schemas RAW → ATOMIC → MART → ML → GOV → DOCS → SPCS. `ARKEMA_PRICING_DB` still has no `CREATE DATABASE` in the scripts. **Verified Sep 18: this database does not exist in `BR68104` at all.** The `SPCS` schema becomes vestigial under SAR. |
| AI layer | Cortex Analyst (semantic model), Cortex Search, Cortex Agent (`ARC_AGENT`, currently granted to `ROLE PUBLIC`). **Verified Sep 18: `SNOWFLAKE_INTELLIGENCE` does not exist in `BR68104`** — the agent has to be built there too. |
| **Deployment target — as built** | **Snowpark Container Services (SPCS)** — `arc/deploy/service_spec.yaml`, `arc/deploy/deploy.sh` (build → push to image repo → `CREATE/ALTER SERVICE`), `arc/deploy/spcs_setup.sql`, `arc/deploy/Dockerfile`, `nginx.conf`, `start.sh`. The signed proposal's RACI also names SPCS ("Implementation of existing prototype (deployed to SPCS)", M3-01 "SPCS Setup & Deploy"). |
| **Deployment target — as directed (Rev 3)** | **Snowflake App Runtime (SAR)** — `APPLICATION SERVICE` object, `app.yml` manifest, `snow app deploy`, remote build, managed scaling. **None of the SPCS artifacts above carry over.** No Dockerfile, no nginx, no compute pool, no image repository. See §6. |
| Repo structure | Historically two divergent trees: `arc/` (root, newer feature DDL: F11 PI tracking, lineage/scenario, price-ref dedup, an MVP cockpit variant) and `scale/arc/` (performance variant targeting 50K materials × 150K customers × 24mo, adds clustering/search-optimization/dynamic tables). **Resolved Sep 16: `arc/` is authoritative; `scale/` was a POC scalability demonstration and will be removed from the repo by Rishabh.** |
| Features (per current MVP PRD) | F1 Ingestion/DQ · F2 Simulation engine · F3 BOM rollup/health score/lineage · F4 Customer margin impact wizard · F5 Margin rollup · F6 Recommendations (**where ML lives, per Michael — F6.1 alone acceptable for MVP**) · F7 Reporting · F8 Ask ARC / GenAI Interface (**F8.1–F8.7 all "Must", customer-confirmed**) · F9 Price Index Evolution · F10 Performance Bridge · F11 Price Increase Tracking · F12 Component Price Coverage · F13 Procurement History · F14 Multi-Currency. Multiple older PRD versions exist under `app_requirements/` — **ignore them all**; only the current MVP PRD (`arc/docs/MVPbridge...`) counts. |

**For the record:** the app was designed and built for **SPCS**. The Docker image, service spec, compute pool, and image repository setup all exist and match SPCS conventions. There is no SAR-oriented code in the repo — no `app.yml`, no Node/Next.js scaffolding, no JavaScript backend. **Rev 3 therefore treats the SAR delivery as a new build of the service tier against an existing frontend and an existing data/DDL layer**, rather than as a deployment of what Rishabh handed over.

### 2a. What survives the platform change, and what does not

The single most useful thing to understand before reading §4 and §6.

| Asset | Fate under SAR | Notes |
|---|---|---|
| `arc/ddl/**` (SQL DDL, stored procs, semantic model) | **Survives intact** | Platform-independent. Still the highest-value asset in the repo. |
| `arc/cortex/**` (agent, semantic view, search) | **Survives intact** | Snowflake-side objects, unaffected by how the app is hosted. |
| `arc/ml/**` | **Survives intact** | Unaffected (and ML scope is separately unresolved). |
| `arc/frontend/**` (React 18 + TS + Vite) | **Mostly survives** | Already TypeScript/npm. Needs restructuring into a Node/Next.js project and rewiring to the new API layer. |
| `arc/backend/**` (FastAPI, ~4,400 LOC, 93 routes) | **Does not survive** | Must be re-implemented in Node.js. The SQL it issues can be lifted; the framework, routing, auth, and connection handling cannot. |
| `arc/deploy/Dockerfile`, `nginx.conf`, `start.sh` | **Discarded** | SAR builds and serves the app; no container or reverse proxy to author. |
| `arc/deploy/service_spec.yaml`, `service_spec_mvp.yaml` | **Replaced by `app.yml`** | Env vars, secrets, and EAIs move into the SAR manifest. |
| `arc/deploy/spcs_setup.sql` | **Largely discarded** | Compute pool and image repository are not SAR concepts. The network rules / EAI section is still reusable. |
| `arc/deploy/deploy.sh`, `deploy_mvp.sh` | **Replaced by `snow app deploy`** | Also means the "one consolidated deploy script" Rishabh owed is now of limited value. |

**Consequence for KT:** Rishabh's remaining hours should be spent on the data model, DDL build order, and agent wiring — the assets that survive — and **not** on the SPCS deployment path, which no longer applies. This changes what I ask him for before his cutoff (§8).

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
- SPCS OAuth secret handling is correct and never logged. **(Moot under SAR — the runtime supplies its own identity; see §3.4.)**
- Stored procedures bind parameters properly — SQL injection risk is entirely at the FastAPI service tier, not the procs.
- Browser never talks to Snowflake directly.
- GOV/DQ schema and objects are well-built.
- ML statistical machinery (OLS, XGBoost, SUR) is real and sound — it's just structurally disconnected from what the UI shows.

### 3.4 How the SAR decision interacts with the audit (new in Rev 3)

The platform change does not make the audit go away, but it does reshuffle it. This matters because it is the one place where the SAR decision genuinely helps.

| Audit blocker | Effect of moving to SAR |
|---|---|
| #1 No authentication (client-supplied `X-ARC-Persona`, fails open to a write-capable persona) | **Materially improved.** SAR apps sit behind Snowflake's security perimeter and inherit SSO — the app knows who the signed-in user is without me building auth. **But it does not solve authorization:** mapping a user to a persona and enforcing it server-side is still work, still the M3-02 line item, still budgeted at 0h. SAR gives me a trustworthy identity to enforce *against*, which is the hard half. **This is the strongest technical argument for Daniel's decision.** |
| #2 Zero parameterized SQL (53 f-string statements) | **Neutral-to-positive by accident.** Rewriting the service tier in Node is an opportunity to bind parameters from the start rather than retrofit 53 call sites. The defect does not port if the rewrite is done properly. |
| #3 Numbers are not reliable (hardcoded +10/+8/+5%; unreachable elasticity gate; mismatched cost bases; no currency conversion) | **Unchanged.** These are logic and data defects. They will be faithfully reproduced by any rewrite unless explicitly fixed, and the rewrite is a chance to reproduce them *wrongly* as well. **Highest regression risk in the whole SAR move.** |
| #4 Max safe concurrency = 1 (all routes `async def` wrapping blocking calls) | **Improved.** SAR provides managed horizontal scaling via `min_instances`/`max_instances`, and a Node rewrite is async-native. The per-instance blocking-call problem disappears if written idiomatically. |
| #5 BOM fan-out up to 144× with no statement timeout | **Unchanged.** This is in the SQL and the warehouse, not the app tier. Still needs a warehouse `STATEMENT_TIMEOUT_IN_SECONDS` guardrail. |
| #6 Deployment not reproducible from the repo | **Improved.** `app.yml` + `snow app deploy` is a materially more reproducible path than the stub `deploy.sh`. The missing `CREATE DATABASE` for `ARKEMA_PRICING_DB` is still a gap. |
| #7 Error handling doesn't exist (failed queries → `[]` with HTTP 200) | **Unchanged in nature, but must be built correctly in the rewrite** rather than fixed afterwards. Same caveat as #3: a faithful port reproduces the bug. |

**Net read:** SAR helps most on the two blockers that were hardest to fix in place (auth identity, concurrency) and helps not at all on the two that are most likely to fail UAT (wrong numbers, silent error handling). It also introduces a new risk class that did not exist before — **rewrite regression** — against an app whose correct numbers are unknown, because §3.1 item 3 says none of the ~14 numbers shown is provably correct. **There is no known-good baseline to regression-test the rewrite against.** That is a serious problem and it is called out again in §5.

---

## 4. Proposed Phased Plan (App Productionalization workstream)

This maps the contracted LOE milestones (M1, M3, M5, M6 — App columns) to concrete actions, informed by the audit and **re-scoped in Rev 3 for a SAR delivery**.

### Phase 0 — Pre-Kickoff Alignment (closed items + new SAR gates)
- [x] Attend Sep 16 walkthrough with Rishabh Raman.
- [x] **Which repo tree is authoritative — RESOLVED: `arc/`.** `scale/` was a POC scalability demo, not MVP-relevant; Rishabh to sync the repo and remove it. Do not spend further time on `scale/arc/`.
- [x] **Which PRD is authoritative — RESOLVED:** multiple PRD versions exist under `app_requirements/`; ignore all except the current MVP PRD (`arc/docs/MVPbridge...`). All earlier versions are POC-era.
- [x] **RBAC baseline — RESOLVED:** use the repo's persona-based RBAC as the baseline. Rishabh explicitly warned against reopening RBAC scope with the client.
- [x] **Deployment mechanism — RESOLVED: SAR**, per Daniel Sandler, Sep 18. Supersedes the Rev 2 SPCS decision. §6 rewritten.
- [x] **Target database — RESOLVED by verification:** `ARKEMA_PRICING_DB` is the intended target and **does not exist in `BR68104`**. It must be built from `arc/ddl` (which is missing its `CREATE DATABASE`).
- [x] **Account verified** — `BR68104` / `AWS_EU_WEST_1`; SAR region availability confirmed; my role holds zero privileges.
- [ ] **NEW / BLOCKING #1 — confirm `BR68104` is a paid account, not a trial.** SAR does not run on trial accounts. If this is still a trial pending CAP1, the entire SAR plan is blocked and this must go back to Daniel. **Ask today.**
- [ ] **NEW / BLOCKING #2 — get an `ACCOUNTADMIN` to run Snowsight App Development Setup** and issue the grants in `20260918_GrantsNeeded.md`. Without this, deploys land in my personal database where the app cannot be shared with anyone — so no Michael early access and no UAT.
- [ ] **NEW / BLOCKING #3 — get the SAR rewrite scoped and resourced (§6.2).** This is a change order, not a re-sequencing. Needs Brendan + Chase + Daniel aligned in writing before I write code.
- [ ] **NEW — confirm who writes the Node backend.** ~4,400 LOC of FastAPI across 93 routes is not a one-person job inside the remaining budget. Is this me alone, me plus another SA, or a scoped subset of screens?
- [ ] **Get demo-account access from Rishabh** (he agreed to provide it) so I can see a working instance and capture reference behaviour **before** the rewrite. **This is now more urgent, not less** — the demo account is the only source of truth for what the app is supposed to output (see §3.4, no known-good baseline).
- [ ] ~~Get the single consolidated deploy script from Rishabh~~ — **deprioritized.** It targets SPCS and is superseded by `snow app deploy`. Do not spend his remaining hours on it.
- [ ] **Book the deep-dive sessions Rishabh offered**, re-aimed at what survives (§2a): data model + DDL build order, and UI/persona flow + Cortex Agent wiring. **Explicitly not** the SPCS deployment path. **My action item, before his cutoff.**
- [ ] Track Chase's ML-scope email to Michael to written closure (Chase owns; I am Informed).

### Phase 1 — App Discovery (M1-03, budgeted 12h) — **re-aimed at capturing a behavioural baseline**

The purpose of Phase 1 has changed. In Rev 2 it was orientation before deploying someone else's app. In Rev 3 it is **specification capture before rewriting the service tier** — because once Rishabh is gone and the FastAPI tier is replaced, undocumented behaviour is unrecoverable.

- [ ] Pin the baseline commit/tag in `arc/`.
- [ ] Get into Rishabh's demo account and drive the working app end-to-end. **Capture outputs, not impressions:** screenshot every screen per persona and record the actual numbers rendered. This becomes the de-facto regression baseline for the rewrite. There is no other one.
- [ ] Stand up local dev environment; run the existing FastAPI app once end-to-end against a fresh DB build from `arc/ddl` (Andrew's "1-hour experiment #1" — still never done).
- [ ] **NEW — inventory all 93 route decorators**: path, method, persona gate (if any), SQL issued, stored procedures called, response shape. This inventory *is* the rewrite specification. Without it the rewrite is guesswork.
- [ ] Walk all screens (confirmed 16, not the documented 12) and confirm actual persona count (confirmed 4, not the documented 5).
- [ ] **Audit F8 GenAI Interface against F8.1–F8.7** now that Michael has confirmed it Must-have. Verify F8.7 context scoping (query scoped to selected cost simulation runs / named price scenarios, context displayed persistently, all AI responses and generated SQL scoped to it). Rishabh's in-UI assistant is beyond-PRD and does **not** run simulations. Treat any shortfall as a scope finding, not a bug to silently absorb.
- [ ] Map each screen's API calls to the production consumption layer (MART) that Platform workstream will deliver.
- [ ] Confirm write paths (scenario save, override, governance actions) and the export column set.
- [ ] Inspect real ZFSC07/customer-segment extracts once Wole lands sample CSV data (Andrew's "experiment #2") to determine if the 144× BOM fan-out is real exposure or theoretical.
- [ ] **Usability reality check.** Capture a short written list of usability risks per persona during the screen walk. Cheaper in Week 2 than in October UAT.
- [ ] **NEW — spike the SAR path end-to-end on a throwaway app before committing.** Scaffold a minimal Node/Next.js app, `snow app setup`, `snow app deploy`, confirm it reaches Snowflake and can query a table. Validates the grants, the build egress, and the toolchain in hours rather than discovering problems mid-rewrite. **Do this the moment the grants land.**

### Phase 2 — App Productionalization (M3, budgeted 64h) — **re-scoped for SAR**

The contract's M3 line items were written for an SPCS deployment of an existing app. Under SAR, M3-01 shrinks and a new, much larger line item appears. Budgeted hours below are the **contract's**, not my estimate; the gap is the point.

| LOE item | Budgeted | Action under SAR |
|---|---|---|
| ~~M3-01 SPCS Setup & Deploy~~ → **M3-01 SAR Setup & Deploy** | 8h | **Smaller than budgeted.** No image repository, no compute pool, no Dockerfile, no nginx. Work is: `snow app setup`, author `app.yml` (name, destination DB/schema, `query_warehouse`, `min_instances`/`max_instances`, env vars, secrets, EAIs), `snow app deploy`, verify endpoint, document rollback via `ALTER APPLICATION SERVICE ... UPGRADE`. **Realistically fits in 8h once grants exist.** This is the one line item SAR makes cheaper. |
| **NEW — M3-0X Backend rewrite (FastAPI → Node.js)** | **0h — not in contract** | Re-implement ~4,400 LOC / 93 routes as a Node service, driven by the Phase 1 route inventory. Includes: Snowflake connectivity, parameterized SQL (fixes blocker #2 by construction), server-side persona enforcement, real error handling, and the write paths. **This is the change order.** It is the largest single unbudgeted item in the engagement and it sits on the critical path to everything downstream. |
| **NEW — M3-0Y Frontend port to Node/Next.js** | **0h — not in contract** | Restructure the Vite/React app into the SAR project layout and rewire it to the new API surface. Lower risk than the backend (already TypeScript), but not free — 16 screens, Leaflet + Vega + Recharts all need to build under the SAR remote build. |
| M3-02 RBAC Wiring | 0h* (audit: needs 11h min) | **Unchanged as a requirement, cheaper to do well under SAR.** Signed proposal Slide 16: *"Bind 5 roles to server-side authorization. Role drives screen access. Remove browser-claimed persona. Audit events."* SAR supplies a trusted signed-in identity (§3.4), so the work is mapping identity → persona and enforcing it, not building auth. **Still budgeted at 0 hours. Still a change-order risk.** Fold into the rewrite rather than retrofitting afterwards — doing it during the rewrite is far cheaper than doing it twice. |
| M3-04 Write Path Hardening | 16h | Saved scenarios, price-increase assumptions, transaction boundaries, idempotent re-runs. **Now performed as part of the rewrite rather than as a patch** — same requirement, different sequencing. |
| M3-05 Error Handling & Observability | 12h | Stop returning empty-as-success; add loading/empty/error/unauthorized UI states. **Built in rather than retrofitted.** Observability via `SYSTEM$GET_APPLICATION_SERVICE_LOGS` / `snow app events` instead of SPCS event-table wiring. |
| F8 GenAI verification | drawn from Misc. | Confirm the Cortex Agent + semantic view build cleanly in `BR68104` (**neither `SNOWFLAKE_INTELLIGENCE` nor `ARKEMA_PRICING_DB` exists there yet**), that the agent is **not** granted to `ROLE PUBLIC` as in the POC, and that F8.7 context scoping behaves. Customer-Must scope; cannot be deferred. |
| Misc. Fixes | 28h | Contingency. **Realistically this is now absorbed by the rewrite and should not be treated as available slack.** |
| **NEW — decide owner's rights vs caller's rights** | — | A design decision with security consequences, not a config toggle. Recommendation: **owner's rights for MVP**, because persona enforcement is not yet server-side and caller's rights would add a second failure mode. Revisit post-MVP. **Note the execution role is fixed at creation and cannot be changed** — getting this wrong means dropping and recreating the service. |
| **NEW — external access for map tiles + fonts** | — | OSM tiles and Google Fonts move from `spcs_setup.sql` network rules into `app.yml` `external_access_integrations`. Also confirm SAR **build-time** egress is enabled in this account, or npm/font fetches fail during remote build and a `build_eai` is needed. |

**Concerns on Phase 2, in order of cost:**
1. **The rewrite is unbudgeted and on the critical path.** M3 is 64 contracted hours. The rewrite alone plausibly exceeds that, and M3-02 is separately budgeted at 0h. Mid-October UAT with a Sep 18 start on a from-scratch service tier is extremely tight.
2. **No regression baseline exists.** Per §3.1 item 3, none of the ~14 numbers the app shows is provably correct. So the rewrite cannot be validated as "matches the old app" *or* as "correct" — only as "matches the screenshots I captured in Phase 1." Capture them early and treat them as the contract.
3. **M3-02 at 0 hours** remains the single highest-risk line item, unchanged from Rev 2.


### Phase 3 — Integrated Testing & UAT (M5, App portion budgeted 40h) — **UAT anchored mid-October**
- [ ] Build test plan from acceptance criteria; seed test users across the (confirmed) 4–5 roles.
- [ ] End-to-end testing: app → API → consumption layer → (ML output, if ML stays in scope), with per-role access checks and FP&A calculation spot-checks.
- [ ] **NEW — regression-test the rewrite against the Phase 1 baseline.** Screen-by-screen, persona-by-persona, compare rendered numbers to the captured reference. This is the only defence against the rewrite silently changing outputs. **Budget real time for it; it is not covered by the contract's 40h, which assumed testing a deployed app rather than a rebuilt one.**
- [ ] **F8 GenAI test cases.** F8.1–F8.7 are all "Must" per Michael — including his own example question (*"what's our margin exposure in Asia if MMA hits EUR 2,000/t?"*) and F8.6 commercial-justification text generation.
- [ ] **NEW — multi-user concurrency test.** SPCS capped correct concurrency at 1; SAR plus a Node rewrite should lift that, but it must be *demonstrated* with concurrent sessions before UAT, not assumed. Tune `min_instances`/`max_instances` off the result.
- [ ] **Stand up pre-UAT early access for Michael.** **Hard prerequisite: the app must be in a standard database, not my personal database** — an Application Service in a personal DB cannot be granted to anyone. Blocked on Phase 0 item #2.
- [ ] Share access via `GRANT USAGE ON APPLICATION SERVICE` to the UAT role(s); `MONITOR` to whoever needs logs.
- [ ] Support Arkema-driven 2-week UAT; Snowflake reverse-shadows testing in parallel.
- [ ] Track UAT defects against the audit's known-issue list so nothing already identified gets "rediscovered" mid-UAT.

### Phase 4 — Launch Support (M6, budgeted 20h)
- [ ] Final production deploy at end of Week 8 via `snow app deploy`.
- [ ] Smoke test all screens across all roles.
- [ ] Verify the daily refresh ran and models scored (confirm the ML retrain task chain actually executes — audit found it's currently inert, only inserting `PENDING` log rows).
- [ ] **NEW — document the SAR operational runbook**, which differs entirely from the SPCS one the contract anticipated: deploy (`snow app deploy`), upgrade/rollback (`ALTER APPLICATION SERVICE ... UPGRADE TO VERSION`), suspend/resume, scaling (`min_instances`/`max_instances`, `auto_suspend_secs`), logs (`snow app events`, `SYSTEM$GET_APPLICATION_SERVICE_LOGS`), and sharing (`GRANT USAGE/MONITOR/OPERATE`).
- [ ] **NEW — flag two irreversibility traps in the runbook:** `UNDROP` is not supported for Application Services, and **ownership transfer is not supported** — so the owning role must be right from day one (§6.4).
- [ ] Deliver admin runbook (deploy, rollback, refresh failure, permission changes) and one KT session.
- [ ] Hand off named support owner and open issue list.

---

## 5. Key Risks (from contract + audit, consolidated)

| Risk | Source | Impact | Mitigation |
|---|---|---|---|
| **NEW (Rev 3): SAR cannot run the app's Python backend — full service-tier rewrite required, unbudgeted** | SAR limitations (Node.js only, Python "planned") + repo inspection | **Critical — schedule and budget** | Scope and price as a change order **before** writing code. Confirm who does the work. If it cannot be resourced, escalate to Daniel that mid-October UAT and SAR are not simultaneously achievable, and ask which one moves. |
| **NEW (Rev 3): rewrite regression with no known-good baseline** | §3.4 + Audit Pass 3 (no number provably correct) | **High — silent wrong answers** | Capture screen-by-screen, persona-by-persona reference outputs from Rishabh's demo account in Phase 1, **before** his cutoff. Treat those captures as the regression contract. This is the only baseline that will ever exist. |
| **NEW (Rev 3): `BR68104` may still be a trial account — SAR would be impossible** | SAR limitations + `CREATE APPLICATION SERVICE` usage notes + §1 | **Critical — binary gate** | Confirm paid status **first**, before any other SAR work. If trial, the SAR plan stops and goes back to Daniel. |
| **NEW (Rev 3): I have zero privileges; App Development Setup never run** | Account verification, Sep 18 | **Critical — everything is blocked** | `20260918_GrantsNeeded.md` is ready to send. Needs an `ACCOUNTADMIN` and a named turnaround. Identify that person today. |
| **NEW (Rev 3): personal-database deploy cannot be shared — blocks Michael's early access and UAT** | SAR limitations (no `GRANT` on PDB Application Services) | High — UAT schedule | Same fix as above: get App Development Setup run so deploys land in a standard database. Do not waste a deploy attempt into the personal DB expecting to share it later. |
| **NEW (Rev 3): execution role and ownership are irreversible** | SAR access control + limitations (no ownership transfer, `EXECUTE_AS_ROLE` immutable, no `UNDROP`) | Medium — costly to unwind | Create a dedicated `ARC_APP_PUBLISHER` role and deploy under it from the very first attempt. Do not "just test" under my AAD role. |
| **NEW (Rev 3): contract names SPCS; SAR is a platform change** | Signed proposal RACI + M3-01 "SPCS Setup & Deploy" + Delivery Assumptions ("no rearchitecture") | Medium-High commercially | Get Daniel's direction reflected in writing by Brendan/Chase, with the change-order path stated. I should not be the only record of why the platform changed. |
| Entitlement/auth is unbudgeted but is the #1 blocker | Audit F-2.01–03, M3-02 = 0h | High — cannot go live with an anonymous write path | **Partly helped by SAR** (trusted SSO identity, §3.4), but authorization is still unbudgeted work. Raise with Brendan before Week 3; fold into the rewrite rather than retrofitting. |
| "Deploy as-is" contract clause vs. numbers being provably wrong | Contract Slide 4/10/11 vs. Audit Pass 3 | High — going live with incorrect pricing/margin numbers is a business risk, not just a technical one | Get explicit written sign-off from Arkema on what "as-is" covers once they see the specific wrong-number findings. **Sharper under SAR: a rewrite forces a decision on every number, so silence is no longer neutral.** |
| Repo divergence (`arc/` vs `scale/arc/`) | Audit lineage.md, Pass 0 | ~~High~~ **CLOSED Sep 16** | Resolved: `arc/` is authoritative, `scale/` to be removed. Confirm the removal lands. |
| Deployment target DB never created by any script | Audit Pass 0 | **Confirmed, not theoretical** | **Verified Sep 18: `ARKEMA_PRICING_DB` does not exist in `BR68104`.** Add the missing `CREATE DATABASE`; verify `arc/ddl` builds from empty. Needs either `CREATE DATABASE ON ACCOUNT` or a pre-created DB I own. |
| 144× BOM fan-out with real data, no timeout/cancel | Audit Pass 4 | High once real data lands | **Unchanged by SAR.** Run "experiment #2" early; set `STATEMENT_TIMEOUT_IN_SECONDS` on the warehouse as a cheap guardrail now. If confirmed, raise as change order (13 days, per audit). |
| Synchronous blocking runs cap concurrency at 1 correct user | Audit Pass 4 | ~~Medium-High~~ **Reduced by SAR** | SAR gives managed scaling and a Node rewrite is async-native. **But it must be demonstrated, not assumed** — concurrency test in Phase 3. |
| Production data schema may deviate from mock/dev schema | Contract Slide 12 (flagged High in the SOW itself) | High | Pre-kickoff schema validation is a contractual pre-req; do not assume match. Sources confirmed **CSV** — get sample files the moment they exist. |
| No formal KT; Rishabh's editing cutoff | Prep meeting notes; not restated Sep 16 | **High and now worse** | The rewrite makes his knowledge more valuable, not less, and the cutoff unchanged. **Book the deep dives immediately** and re-aim them at what survives (§2a). Ask Brendan to state the cutoff date in writing. |
| ~~SPCS vs. SAR unresolved~~ | Rev 1 → Rev 2 → Rev 3 | **CLOSED — SAR** | Decided by Daniel Sandler, Sep 18. Reverses the Rev 2 SPCS decision. See rewritten §6. |
| ML scope contradiction across SOW / PRD / customer | Sep 16 App Review + Chase/Michael email | Medium for me, High commercially | Chase owns closure with Michael in writing. **Now matters more to me:** if ML returns, the F6 recommendation screens are additional rewrite surface. Do not build against either assumption until it is written down. |
| F8 GenAI is customer-Must but was never audited | Chase/Michael email vs. Andrew's audit coverage | Medium-High | Audit F8.1–F8.7 in Phase 1. Note **`SNOWFLAKE_INTELLIGENCE` does not exist in `BR68104`** — the agent must be built there. Do not carry over the POC's `ARC_AGENT → ROLE PUBLIC` grant. |
| App has never been used by any Arkema business user; usability doubted by its own author | Sep 16 App Review | High for UAT sign-off | Capture usability risks during the Phase 1 screen walk; push for Michael's early access. **Unchanged by the platform move, but the rewrite consumes the time that would have gone here.** Most likely cause of a failed UAT. |
| Customer platform not Snowflake-ready; S3 bucket has no readiness date | Wole's Sep 15 note | High — schedule risk | Synthetic-data-first so SAR setup, rewrite, and RBAC proceed in parallel with Wole's ingestion. Do not make Phase 2 conditional on real data. |
| InfoSec has not confirmed single-account architecture | Wole's Sep 15 note | Medium | Michael to confirm. If InfoSec forces multi-account, note SAR handles multi-environment cleanly via `app.yml` `targets` — a genuine simplification versus SPCS. |

---

## 6. Deployment Target — Decision: SAR (Snowflake App Runtime)

**Status as of Sep 18: decided in favour of SAR by Daniel Sandler.** This reverses the Rev 2 decision. The plan is now built around SAR and I am proceeding on that basis. This section records what that means concretely, what it costs, and what has to be true for it to work.

### 6.1 What the SAR delivery looks like

| Aspect | SPCS (Rev 2 plan) | **SAR (Rev 3 plan)** |
|---|---|---|
| Snowflake object | `SERVICE` | **`APPLICATION SERVICE`** |
| Manifest | `service_spec.yaml` | **`app.yml`** (written by `snow app setup`) |
| Deploy command | `deploy.sh` (build → push image → `CREATE/ALTER SERVICE`) | **`snow app deploy`** (upload source → remote build → create/alter service) |
| Container | Hand-authored `Dockerfile` + nginx + `start.sh` | **None** — Snowflake builds and serves it |
| Compute | `ARC_COMPUTE_POOL`, `INSTANCE_FAMILY`, node counts | **Managed** — `min_instances` / `max_instances` / `auto_suspend_secs` |
| Image registry | `ARKEMA_PRICING_DB.SPCS.ARC_IMAGES` | **Artifact repository, auto-provisioned by deploy** |
| Runtime language | Anything containerizable (Python was fine) | **Node.js only** — the binding constraint |
| Versioning / rollback | Pinned image tags | **Immutable build versions**; `ALTER APPLICATION SERVICE ... UPGRADE TO VERSION` |
| Egress | Network rules + EAI in `spcs_setup.sql` | **`external_access_integrations` in `app.yml`** (plus separate build-time egress) |
| Identity | App builds its own (it didn't) | **Inherits Snowflake SSO / RBAC perimeter** |
| Sharing | Service endpoint grants | `GRANT USAGE / MONITOR / OPERATE ON APPLICATION SERVICE` |
| Logs | Event table wiring | `snow app events`, `SYSTEM$GET_APPLICATION_SERVICE_LOGS` |
| Multi-env | Separate specs / scripts | **`app.yml` `targets`** with `default_target` |

**Genuine wins from this decision, stated plainly:** no container to build or maintain, no compute pool or image repository to administer, a materially more reproducible deploy than the stub `deploy.sh`, managed scaling that removes the single-correct-user ceiling, SSO identity for free (which is half of the M3-02 problem), and clean DEV/UAT/PROD separation via `targets` if InfoSec forces multi-account. On the deployment and operations axis, SAR is the better platform. That is not in dispute.

### 6.2 The cost: the Python backend cannot come along

This is the one thing that must not get lost.

**SAR deployable projects are Node.js (typically Next.js). Python support is documented as "planned", not available.** The app's service tier is FastAPI: ~4,400 LOC across 5 route modules and 3 service modules, with 93 route decorators, doing Snowflake connectivity, persona gating, simulation orchestration, BOM rollup, margin math, and Cortex Agent calls.

There is no lift-and-shift. The options are:

| Option | What it means | Assessment |
|---|---|---|
| **A. Rewrite the backend in Node.js** | Re-implement all 93 routes against the Phase 1 route inventory. SQL and stored-proc calls can be carried over; framework, routing, auth, connection handling, and error handling cannot. | **The only option that delivers SAR as asked.** Unbudgeted; largest single item in the engagement; on the critical path. **This is the plan of record.** |
| **B. Deliver a reduced SAR scope** | Rewrite only the screens needed for UAT sign-off (e.g. the F8 GenAI interface, which is customer-Must, plus the core simulation and margin screens), and defer the rest. | **Worth putting to Daniel as the realistic mid-October option.** Needs Michael's acceptance criteria to know which screens are non-negotiable — and nobody owns those yet (§7 item 14). |
| **C. Wait for SAR Python support** | Defer until Python is GA. | Not viable — no published date, and UAT is mid-October. |
| **D. SPCS now, SAR after go-live** | Deliver the contracted SPCS path for UAT, migrate to SAR as a phase 2. | **Not the direction given.** Recorded only so the option is visible if the change order is refused. |

**My recommendation, offered once and then dropped:** if mid-October UAT is immovable *and* the change order cannot be funded, Option B is the only honest way to deliver SAR on time, and it needs Michael's acceptance criteria this week to scope. If the change order *can* be funded, Option A is the right build. Either way this is Daniel's and Brendan's call, not mine — **I am proceeding on Option A until told otherwise**, and Phase 1 work (route inventory, baseline capture) is valuable under A and B alike, so it is not wasted effort while the decision is made.

### 6.3 Hard prerequisites before any SAR work

Three gates. All are binary, none are mine to clear.

1. **`BR68104` must be a paid account.** SAR is not available on trial accounts; account admin setup requires a paid account. §1 records Arkema as on-demand/trial pending CAP1. **If this is still a trial, everything in §6 is blocked** and the decision goes back to Daniel. **Verify first.**
2. **An `ACCOUNTADMIN` must run Snowsight App Development Setup** (Settings → Account → Apps → Begin Setup) or issue the equivalent SQL. Verified Sep 18: the `DEFAULT_SNOWFLAKE_APPS_*` parameters are all empty, so this has never been done.
3. **My role needs privileges.** It currently has none. Full request, with verified evidence and both the SAR and SPCS grant sets, is in **`20260918_GrantsNeeded.md`**.

### 6.4 Two irreversible choices to get right on the first deploy

Both are cheap now and expensive later.

- **Deploy under a dedicated `ARC_APP_PUBLISHER` role, not my AAD role.** For an app in a standard database the execution role **is** the creating session's primary role, and it **cannot be changed** afterwards — changing it means dropping and recreating the service. Ownership transfer is also unsupported. Deploying once "just to test" under `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` would permanently bind the app's execution identity to an Azure group.
- **Decide owner's rights vs caller's rights before the first deploy.** Recommendation: **owner's rights for MVP**, because persona authorization is not yet enforced server-side and caller's rights would add a second failure mode on top of an unsolved one. Revisit post-MVP once M3-02 is genuinely done.

Also worth noting: `UNDROP` is not supported for Application Services. A dropped service is gone.

### 6.5 Getting the decision on the record

The signed proposal names SPCS in the RACI ("Implementation of existing prototype (deployed to SPCS)"), in LOE item M3-01 ("SPCS Setup & Deploy"), and in the Delivery Assumptions slide ("no new features, integrations, or **rearchitecture** in scope"). A move to SAR that requires rewriting the service tier is a rearchitecture by that slide's own wording.

**Action:** ask Brendan to minute Daniel's direction, and ask Chase to confirm the commercial path (change order vs reduced scope per §6.2 Option B). I should not be the only written record of why the platform changed — that exposes both me and the engagement.

---

## 6a. Confirmed Decisions Register (as of Sep 18)

| Decision | Value | Decided by / where | Firmness |
|---|---|---|---|
| Authoritative repo tree | `arc/` (`scale/` to be deleted) | Rishabh, Sep 16 App Review | Firm |
| Authoritative PRD | current MVP PRD (`arc/docs/MVPbridge...`); ignore all `app_requirements/` versions | Rishabh, Sep 16 | Firm |
| **Deployment mechanism** | **SAR (Snowflake App Runtime)** — reverses the Rev 2 SPCS decision | **Daniel Sandler, Sep 18** | **Firm as direction; needs minuting by Brendan (§6.5)** |
| **Backend language** | **Node.js — forced by SAR, not chosen.** Python backend must be rewritten. | SAR platform limitation | **Firm (platform constraint)** |
| **Rewrite scope (Option A full vs B reduced)** | — | — | **OPEN — needs Daniel + Brendan + Chase (§6.2)** |
| **Rewrite resourcing / change order** | — | — | **OPEN — blocking Phase 2** |
| Execution model (owner's vs caller's rights) | **owner's rights for MVP** (recommendation) | William, Rev 3 | Proposed — **decide before first deploy, it is irreversible** |
| Deploying role | **`ARC_APP_PUBLISHER`** (new, granted to my AAD role) | William, Rev 3 | Proposed — **decide before first deploy** |
| RBAC model | repo's persona-based model as baseline; do not reopen with client | Rishabh, Sep 16 | Firm |
| GenAI / Cortex Agent + Cowork | **IN scope**, F8.1–F8.7 all "Must" | Michael THAI, written email Sep 16 | Firm |
| ML in MVP | **UNRESOLVED** — PRD says none, SOW assumed it, Michael says keep POC ML *if cheap*, F6.1 alone acceptable | Conflicting | **Open — Chase to close in writing** |
| Target database | `ARKEMA_PRICING_DB` — **confirmed absent from `BR68104`; must be built from `arc/ddl`** | Verified Sep 18 | Firm (name); **build not started** |
| Target account | **`BR68104`, region `AWS_EU_WEST_1`** | Verified Sep 18 | Firm |
| **Paid vs trial account** | — | — | **OPEN — blocking gate for SAR (§6.3)** |
| **App Development Setup** | **not run** — all `DEFAULT_SNOWFLAKE_APPS_*` params empty | Verified Sep 18 | **OPEN — blocking** |
| Snowflake account topology | single account, logical DEV/UAT/PROD | Wole + team, Sep 15 | Pending InfoSec (Michael) |
| Source file format | CSV from AWS S3 | Wole, Sep 15 | Firm; bucket not ready |
| UAT | mid-October, 2 weeks, Arkema-driven | Sep 16 App Review | Firm-ish — **at risk from the rewrite** |
| Rishabh editing cutoff | Sep 21 (from prep meeting; **not** restated Sep 16) | unconfirmed | **Confirm with Brendan — now more urgent** |

---

## 7. Questions — Status and What to Ask Next

**Closed:**
1. ~~Which tree is authoritative?~~ → **`arc/`.** `scale/` is POC-era and will be removed.
2. ~~Which database is the production target?~~ → **`ARKEMA_PRICING_DB`**, and verification on Sep 18 confirms **it does not exist in `BR68104`** — it must be built from `arc/ddl`.
8. ~~SAR vs. SPCS?~~ → **SAR**, per Daniel Sandler, Sep 18 (§6).
9. ~~What is the target account?~~ → **`BR68104`, region `AWS_EU_WEST_1`.** Under SAR there is no compute pool or image repository to own, so that half of the question is moot.

**New and blocking — ask first, in this order:**

15. **Is `BR68104` a paid account or still a trial?** SAR does not run on trial accounts. **This is the single gate that determines whether the SAR plan is executable at all.** Everything else is wasted effort until it is answered.
16. **Who is the `ACCOUNTADMIN` for `BR68104`, and what is their turnaround?** I need them to run App Development Setup and issue the grants in `20260918_GrantsNeeded.md`. My role has zero privileges; I cannot create a database, a schema, or a warehouse.
17. **How is the backend rewrite being resourced?** SAR is Node.js-only, the backend is ~4,400 LOC of Python across 93 routes, and M3 is 64 contracted hours. Is this a change order (Option A), a reduced screen set (Option B), or something else? **Needs Daniel, Brendan, and Chase aligned in writing.**
18. **Who writes it?** Me alone, me plus another SA, or a partner? This determines whether mid-October is achievable.
19. **Does mid-October UAT hold, given the rewrite?** If the answer is "UAT does not move and the change order is not funded," then §6.2 Option B is the only path and I need Michael's acceptance criteria this week to scope which screens make the cut.

**Still open, carried forward:**

3. Has this app ever been run end-to-end against a freshly built (non-synthetic) database? Andrew's audit suggests no. **More important under SAR** — if the answer is no, the rewrite has no validated behaviour to target.
4. M3-02 (entitlement enforcement) is budgeted at 0 hours but is the top blocker in the audit — how is it resourced? **SAR improves this** (trusted SSO identity, §3.4), but authorization work remains. **Note the audit findings were never raised with Rishabh.** Still needs doing.
5. Does Arkema know the pricing/margin numbers currently shown are not reliable (audit Pass 3)? Does "deploy as-is" cover shipping them unchanged? **This is now urgent rather than merely important: a rewrite forces an explicit decision on every number, so there is no longer a "leave it as it is" option.**
   - Rishabh confirmed the *"price increases of like 77%"* recommendations **were his own POC additions, not a customer requirement** — *"that was added by me during the POC. It never came from him."* That makes them prototype artifacts to correct, not contracted behaviour to preserve.
   - **In the code:** `arc/ddl/014_nlc2_agent_recommend.sql`. The elasticity gate is `if beta < -1.0:` (line 48), directly below where `beta` is clamped to a floor of exactly `-1.0` by `shrink_beta()` (line 33). The clamp's output can never be *less than* `-1.0`, so the `<` on line 48 can never be true and the ML-elasticity path (`p_unc`, line 49) never executes.
   - **In the UI:** margin/recommendation screens driven by `arc/backend/services/nlc2_service.py` (reads `ML.NLC2_ELASTICITY`, ~line 396) and `SP_BUILD_RECOMMENDATIONS` per `arc/docs/ARCHITECTURE.md`.
6. Is real ZFSC07 (BOM) data available to test the 144× fan-out before UAT? Practical ask: **can Arkema email a handful of representative CSV extracts now**, ahead of the S3 pipeline?
7. Confirm target go-live date, and the committed date for "enough data to unblock development."
10. Confirm Rishabh's editing cutoff **in writing**, and get the deep dives booked before it — re-aimed at data model + DDL build order and UI/persona flow + Cortex Agent wiring. **Not** the SPCS deploy path.
11. When do I get Rishabh's demo-account access? **Now the highest-value item he owes me**, because it is the only source of a regression baseline for the rewrite.
12. F8 GenAI: which parts of F8.1–F8.7 does the existing agent satisfy, and was F8.7 (context scoping) ever implemented?
13. Pre-UAT early access for Michael: what date, and against which data? **Blocked on App Development Setup** — a personal-database app cannot be shared.
14. Who owns writing the UAT acceptance criteria? **Now doubly urgent:** they are both the UAT gate and, under §6.2 Option B, the input that decides which screens get rewritten.

---

## 7a. Agenda for the Next App / Delivery Call

Decisions first, in descending order of what they unblock:

1. **Confirm paid-vs-trial status of `BR68104`** (Q15). If trial, stop and re-plan — nothing else on this agenda matters.
2. **Name the `ACCOUNTADMIN` and agree a turnaround** for App Development Setup and `20260918_GrantsNeeded.md` (Q16). Everything I do is blocked on this.
3. **Put the rewrite on the table explicitly** (Q17, Q18, §6.2). State the scale — Node.js only, ~4,400 LOC, 93 routes, 0 budgeted hours — and get a decision between Option A (change order) and Option B (reduced screen set). **Do not leave this call without a direction.**
4. **Minute the SAR decision** (§6.5). Daniel's direction needs to be recorded by Brendan, with the commercial path noted, so the deviation from the SOW's SPCS/no-rearchitecture language is documented by someone other than me.
5. **Raise the audit findings** — still never put to Rishabh. Lead with no server-side auth and the correctness findings, framed as "what the rewrite has to account for."
6. **Confirm Rishabh's cutoff and book the deep dives** (Q10, Q11), re-aimed at what survives the platform change (§2a).
7. **Flag the two unowned items:** UAT acceptance criteria and usability validation with a real business user.
8. **State my plan of record:** proceed synthetic-data-first; do Phase 1 (route inventory + baseline capture) immediately, since it is needed under both Option A and Option B; spike the SAR toolchain the moment grants land.

---

## 8. Coming Steps (next 5 working days)

| When | Action | Owner | Blocks |
|---|---|---|---|
| **Sep 18 (today)** | **Confirm `BR68104` paid vs trial** (Q15) | William → Brendan/Daniel | **everything SAR** |
| **Sep 18 (today)** | **Send `20260918_GrantsNeeded.md`**; identify the `ACCOUNTADMIN` and agree a turnaround | William → Brendan/Arkema IT | **everything** |
| **Sep 18 (today)** | **Escalate the rewrite scope and resourcing** (§6.2, Q17–Q19) | William → Daniel + Brendan + Chase | Phase 2 |
| Sep 18 | Ask Brendan to minute the SAR decision and the change-order path (§6.5) | William → Brendan | commercial cover |
| **Sep 18–19** | **Chase Rishabh's demo-account access — now the top KT item.** Capture screen-by-screen, per-persona reference outputs as the rewrite's only regression baseline | William | rewrite validation |
| Sep 18–19 | Book 2× deep dives with Rishabh before his cutoff; confirm the cutoff date in writing | William → Brendan/Rishabh | all KT |
| Sep 18–19 | Pin baseline commit in `arc/`; build the DB from `arc/ddl` on empty and record every gap (missing `CREATE DATABASE`, ordering failures) | William | data layer |
| **Sep 19–22** | **Inventory all 93 route decorators** — path, method, persona gate, SQL, procs, response shape. This is the rewrite specification. | William | rewrite start |
| Sep 19–22 | Walk all 16 screens × 4 personas; produce the F8.1–F8.7 gap list and the per-persona usability risk list | William | Phase 2 scoping, UAT criteria |
| **On grant arrival** | **Spike SAR end-to-end on a throwaway Node app** — `snow app setup`, `snow app deploy`, query a table. Validates grants, build egress, toolchain. | William | de-risks Phase 2 |
| Sep 18–19 | Ask for representative CSV extracts by email, independent of the S3 pipeline | William → Michael via Brendan | BOM fan-out test |
| Sep 18+ | Michael to confirm InfoSec position on account isolation; Wole needs S3 path + IAM role | Michael / Wole | ingestion, env topology |
| Ongoing | Track Chase's ML-scope email to written closure | Chase (I am Informed) | ML workstream, rewrite surface |

**Sequencing principle, revised for Rev 3:** Rishabh's remaining availability was already the scarcest resource; the rewrite makes it scarcer, because his knowledge is now the *specification* rather than just context. Spend every remaining hour of his on what the app is supposed to do and why — and none of it on the SPCS deployment path, which no longer applies.

**Second principle:** do the Phase 1 work now, before the scope decision lands. The route inventory and the behavioural baseline are required under Option A *and* Option B, so they are the only zero-regret work available while §6.2 is unresolved.

---

## 9. Open Items / My Concerns Summary

Ordered by cost. Written for Daniel and Brendan as much as for me.

- **Top concern: SAR cannot run this app's backend, and the rewrite is unbudgeted.** SAR is Node.js-only; Python is planned, not shipped. The service tier is ~4,400 LOC of FastAPI across 93 routes, M3 is 64 contracted hours, and M3-02 is separately budgeted at 0h. **I have replanned around SAR as directed and I am proceeding — but mid-October UAT, a full backend rewrite, and the existing budget cannot all three hold.** One of them has to move, and that is a decision for Daniel and Brendan, not for me to absorb quietly. §6.2 sets out the four options and my recommendation.
- **Second: there is no known-good baseline to validate the rewrite against.** The audit found that none of the ~14 numbers the app displays is provably correct. So a rewrite can be checked against neither "the old app" nor "correct" — only against outputs I capture from Rishabh's demo account before his cutoff. **This makes demo-account access the most time-critical dependency in my workstream**, and it is still not delivered.
- **Third: a binary gate is unverified.** SAR does not run on trial accounts. Arkema is recorded as on-demand/trial pending CAP1. **If `BR68104` is still a trial, the entire SAR plan is void** and we have spent the week planning something unexecutable. Ten minutes of checking retires this.
- **Fourth: I am completely blocked on privileges.** My role has zero grants, App Development Setup has never run, and any deploy today would land in a personal database where the app cannot be shared with Michael or any UAT user. `20260918_GrantsNeeded.md` is ready; it needs an `ACCOUNTADMIN` and a named turnaround.
- **Fifth, unchanged and still the likeliest cause of a failed UAT: usability.** No Arkema business user has ever used this app, the intended users are Excel-native, and its own author questioned who it serves. Nobody owns validating it. **The rewrite makes this worse, not better, because it consumes the weeks that would otherwise have gone to usability and correctness.**
- **Sixth: the contract still says SPCS.** The RACI, LOE item M3-01, and the Delivery Assumptions slide ("no rearchitecture") all name or imply SPCS. Daniel's direction needs minuting by Brendan and a commercial path from Chase. I should not be the only written record of the deviation.
- **What the SAR decision genuinely buys us, to be fair to it:** no container or compute pool to manage, a far more reproducible deploy than the stub `deploy.sh`, managed scaling that removes the one-correct-user ceiling, SSO identity that solves half of the M3-02 problem, and clean multi-environment support if InfoSec forces it. **On the operations axis this is the better platform.** The cost is entirely in the service-tier rewrite, and that cost is real.
- **Schedule concern:** my critical path now runs through four things I do not control — the paid-account answer, the `ACCOUNTADMIN`, the rewrite decision, and Rishabh's remaining hours. Mitigation is to decouple where possible: synthetic-data-first, and do the route inventory and baseline capture now since they are needed under every option.
- Everything above is sourced from the repo, the audit, the signed proposal deck, the Sep 16 transcript, Michael's email, Wole's note, current Snowflake App Runtime documentation, and read-only verification of `BR68104` on Sep 18. **No findings, numbers, or product capabilities in this plan were invented or assumed.** Where something is unverified, it is labelled as such.

---

*Local planning document. The only Snowflake activity behind Rev 3 was read-only `SHOW` / `SELECT current_*()` inspection of account `BR68104` on Sep 18, 2026. **No Snowflake objects were created, altered, or dropped.***
