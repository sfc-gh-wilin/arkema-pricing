# Arkema Pricing Cockpit — App Productionalization Project Plan

**Created:** September 15, 2026
**Last updated:** September 17, 2026 (Rev 2 — post App Review + Chase email thread + Wole platform update)
**Author:** William Lin (App Productionalization workstream owner)
**Status:** ACTIVE — updated ahead of the second App meeting (Sep 17)

**Sources reviewed:**
- App source: `arkema-pricing-ss/` (`arc/` and `scale/arc/` trees)
- Andrew Carson's Cortex Code audit: `~/Docs/Snowflake/20260904 Arkema/Andrew/Andrew - Arkema_POC_Pricing/working/audits/`
- Andrew's Slack summary and prep-call transcript (`andrew-slack-msg.md`, `Transcript_Arkema with Andrew.txt`)
- Signed engagement collateral: `ref/Arkema — Snowflake PS TDR Proposal ... August 2026.pptx` (20 slides), `ref/20260909_PrepMeeting_Summary.md`, `ref/20260909_PrepMeeting_Transcript.txt`
- **New in Rev 2:** `ref/20260916 App Review's transcript.txt` (Sep 16 App Review with Rishabh Raman), `ref/20260916_ChaseEmailAndReply.md` (Chase Growney → Michael THAI scope thread), `ref/20260915_MeetingNoteFromWole.md` (Wole Babalola platform-foundation update)

No Snowflake account was queried for this plan (local-file research only, per instruction).

---

## 0. What Changed Since Rev 1 (read this first)

Five things moved between Sep 15 and Sep 17. Three of them change the plan.

| # | Change | Source | Effect on plan |
|---|---|---|---|
| 1 | **`arc/` is the authoritative tree.** Rishabh: use `arc/` only; `scale/` was a POC scalability demonstration and is irrelevant to the MVP. He also agreed to sync the repo and remove `scale/`. | Sep 16 App Review | **Phase 0 question #1 is CLOSED.** Removes a High risk. |
| 2 | **ML scope is contradictory across three sources — still open.** Rishabh stated plainly *"there's zero machine learning"* in the current MVP PRD; the SOW was scoped **with** ML; Michael's written reply says ML lives under **F6 Margin Protection**, that BU would be happy with **F6.1 alone**, and *"if we can easily embed the ML features Rishabh had already implemented in the POC, let's keep them for now"* — otherwise open to discussion. | Sep 16 App Review + Chase/Michael email | **Still the #1 open commercial item.** Directly affects Luis/Sharanya's ML workstream, less so mine. |
| 3 | **GenAI / Cortex Agent is confirmed IN scope by the customer.** Michael: *"Cortex agent and Cowork must be included as they are key features expected by the Business"* — covered by **F8 GenAI Interface, F8.1–F8.7, all "Must."** Rishabh confirmed Cortex Agents and semantic views are already built in the code, but the in-UI assistant he added goes beyond PRD spec and **does not run simulations**. | Chase/Michael email + Sep 16 App Review | **New scope line for my workstream.** F8.7 (context scoping to a simulation run / price scenario) is a real functional requirement that I must verify exists — the audit did not cover it. |
| 4 | **UAT is mid-October.** Pre-UAT early access for Michael is TBD; Brendan to confirm after the Sep 17 team call. No Sep 21 Rishabh cutoff was restated in the Sep 16 meeting — treat Sep 21 as still-assumed until Brendan confirms otherwise. | Sep 16 App Review | Compresses Phase 1/2. See revised timeline §9. |
| 5 | **Platform foundation is starting, but the customer is not Snowflake-ready.** Single Snowflake account with logical DEV/UAT/PROD separation is the working decision (pending InfoSec confirmation from Michael by Thursday). Source files confirmed **CSV**; AWS S3 bucket still being configured, no readiness date yet. Wole will build storage integration → stage → file format once he has bucket/path + IAM role, and will load partial data early to unblock dev. | Wole's Sep 15 note | **My hard dependency.** I cannot connect the app to real data until Wole's stage lands. Plan for a synthetic-data-first deployment. |

**Also confirmed, and this is the most important non-technical finding:** **no Arkema business user has ever seen or used this app.** Rishabh openly doubted its usability — *"who is it useful for? Maybe Michael"* — and noted the real end users are Excel users, not analysts. The app is, in his words, *"all UI"*: *"all the PRD is UI. There's nothing else in it."* A functionally correct deployment can still fail UAT on usability. Flag this now, not in October.

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
| Key near-term event | **Second App meeting — today, Sep 17.** First App Review held Sep 16 (see §0). Rishabh's editing cutoff was stated as **Sep 21** in the Sep 9 prep meeting but was *not* restated on Sep 16 — confirm with Brendan. No formal KT either way. |
| Target UAT | **Mid-October**, 2 weeks, Arkema-driven (confirmed Sep 16) |
| Budgeted LOE (App stream) | M1‑03 App Discovery: 12h · M3 App Productionalization: 64h · M5 Testing/UAT (App portion): 40h · M6 Launch Support: 20h → **~136h app-specific**, inside a ~140h total App SA allocation over 8 weeks (Slide 9: App SA 140h across weeks 2–9) |
| Independent effort audit | Andrew Carson ran a 5-pass Cortex Code audit against the repo (no live Snowflake). App-tier + service-tier findings alone total **145.5 person-days** of remediation work if done exhaustively; a compressed "fastest path" is **49–54 days**, which alone consumes 77–84% of the full 64-day/three-workstream engagement budget. |

**Bottom line entering today's meeting:** the contracted scope ("deploy as-is") and the audited state of the code (numbers are wrong, no auth, blocking single-user runtime) are still in tension — and critically, **none of the audit findings were put to Rishabh on Sep 16.** Doing so today is the highest-value item on the agenda (§7a). See §5 Risks and §7 Questions.

---

## 2. Current App Architecture (as documented in `arkema-pricing-ss/`)

| Tier | Stack | Notes |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite + Tailwind CSS, Leaflet (map), Vega-Lite/Vega/Recharts | 16 screens exist (docs say 12); persona switcher is a client-side `<select>` |
| Backend | FastAPI (Python), 44 routes | Runs via `snow` CLI locally, or containerized for SPCS in production |
| Data layer | Snowflake DB `ARKEMA_PRICING_DB` — **now the target**, since `scale/` (and with it `ARKEMA_PRICING_DB_SCALE`) is out of scope as of Sep 16; schemas RAW → ATOMIC → MART → ML → GOV → DOCS → SPCS. Note `ARKEMA_PRICING_DB` still has no `CREATE DATABASE` in the scripts — fix under M3-01. |
| AI layer | Cortex Analyst (semantic model), Cortex Search, Cortex Agent (`ARC_AGENT`, currently granted to `ROLE PUBLIC`) |
| **Deployment target per contract & repo** | **Snowpark Container Services (SPCS)** — confirmed by `arc/deploy/service_spec.yaml`, `arc/deploy/deploy.sh` (build → push to image repo → `CREATE/ALTER SERVICE`), and the signed proposal's RACI ("Implementation of existing prototype (deployed to SPCS)", M3-01 "SPCS Setup & Deploy") |
| Repo structure | Historically two divergent trees: `arc/` (root, newer feature DDL: F11 PI tracking, lineage/scenario, price-ref dedup, an MVP cockpit variant) and `scale/arc/` (performance variant targeting 50K materials × 150K customers × 24mo, adds clustering/search-optimization/dynamic tables). **Resolved Sep 16: `arc/` is authoritative; `scale/` was a POC scalability demonstration and will be removed from the repo by Rishabh.** |
| Features (per current MVP PRD) | F1 Ingestion/DQ · F2 Simulation engine · F3 BOM rollup/health score/lineage · F4 Customer margin impact wizard · F5 Margin rollup · F6 Recommendations (**where ML lives, per Michael — F6.1 alone acceptable for MVP**) · F7 Reporting · F8 Ask ARC / GenAI Interface (**F8.1–F8.7 all "Must", customer-confirmed**) · F9 Price Index Evolution · F10 Performance Bridge · F11 Price Increase Tracking · F12 Component Price Coverage · F13 Procurement History · F14 Multi-Currency. Multiple older PRD versions exist under `app_requirements/` — **ignore them all**; only the current MVP PRD (`arc/docs/MVPbridge...`) counts. |

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

### Phase 0 — Pre-Kickoff Alignment (now → Sep 17 second App meeting)
- [x] Attend Sep 16 walkthrough with Rishabh Raman.
- [x] **Which repo tree is authoritative — RESOLVED: `arc/`.** `scale/` was a POC scalability demo, not MVP-relevant; Rishabh to sync the repo and remove it. Do not spend further time on `scale/arc/`.
- [x] **Which PRD is authoritative — RESOLVED:** multiple PRD versions exist under `app_requirements/`; ignore all except the current MVP PRD (`arc/docs/MVPbridge...`). All earlier versions are POC-era.
- [x] **RBAC baseline — RESOLVED:** use the repo's persona-based RBAC as the baseline. Rishabh explicitly warned against reopening RBAC scope with the client.
- [ ] **Which database is the real deployment target** (`ARKEMA_PRICING_DB` vs `ARKEMA_PRICING_DB_SCALE`) — still open, but now largely answered by implication: with `scale/` dropped, `ARKEMA_PRICING_DB` is the target. **Confirm explicitly today** and confirm it is reproducible from `arc/ddl` alone.
- [x] **SPCS vs SAR — effectively resolved in favour of SPCS** for the contracted delivery (see revised §6). Rishabh confirmed the app runs on SPCS in his demo account. Services' own deployment target was not *formally* ratified — get that on the record today.
- [ ] **Get demo-account access from Rishabh** (he agreed to provide it) so I can see a working instance before touching deployment.
- [ ] **Get the single deploy script** Rishabh agreed to provide, replacing the stub `deploy.sh`.
- [ ] **Book the deep-dive sessions Rishabh offered.** He was explicit that 30 minutes cannot cover this app — *"let's meet to do a deep dive in a smaller cadence."* This is my only substitute for formal KT and it must happen before his cutoff. **My action item.**
- [ ] Track Chase's ML-scope email to Michael to written closure (Chase owns; I am Informed — it does not block my deployment path).

### Phase 1 — App Discovery (M1-03, budgeted 12h)
- [ ] Pin the baseline commit/tag to deploy from `arc/` (tree question now closed).
- [ ] Get into Rishabh's demo account and drive the working app end-to-end **before** rebuilding anything locally — cheapest possible orientation and it did not exist as an option in Rev 1.
- [ ] Stand up local dev environment; run the app once end-to-end locally against a fresh DB build from `arc/ddl` (Andrew's "1-hour experiment #1" — never done yet).
- [ ] Walk all screens (confirmed 16, not the documented 12) and confirm actual persona count (confirmed 4, not the documented 5). Rishabh describes ~14 sections/layers driven entirely by persona selection.
- [ ] **New: audit F8 GenAI Interface against F8.1–F8.7** now that Michael has confirmed it Must-have. Specifically verify F8.7 context scoping (query scoped to selected cost simulation runs / named price scenarios, context displayed persistently, all AI responses and generated SQL scoped to it). Rishabh's in-UI assistant is beyond-PRD and does **not** run simulations — so the gap between "an agent exists" and "F8.1–F8.7 satisfied" is unmeasured. Treat any shortfall as a scope finding, not a bug to silently absorb.
- [ ] Map each screen's API calls to the confirmed production consumption layer (MART) that Platform workstream will deliver.
- [ ] Confirm write paths (scenario save, override, governance actions) and the export column set.
- [ ] Inspect real ZFSC07/customer-segment extracts once Wole lands sample CSV data (Andrew's "experiment #2") to determine if the 144× BOM fan-out is real exposure or theoretical.
- [ ] **New: usability reality check.** Since no business user has ever used this app and Rishabh doubts its fit for Excel-native users, capture a short written list of usability risks per persona during the screen walk. This protects the UAT window — it is far cheaper to surface these in Week 2 than to discover them in mid-October UAT.

### Phase 2 — App Productionalization (M3, budgeted 64h)
Per contract LOE line items, re-sequenced by the audit's chokepoint analysis where it doesn't expand scope:

| LOE item | Budgeted | Action |
|---|---|---|
| M3-01 SPCS Setup & Deploy | 8h | Image repo, compute pool, service spec, pinned image, repeatable deploy + rollback script. Rishabh agreed to hand over **one consolidated deploy script** — start from that rather than the stub `deploy.sh`. |
| M3-02 RBAC Wiring | 0h* (audit: needs 11h min) | **What "M3-02" is:** LOE line item #2 under Milestone 3 (App Productionalization) in the signed proposal deck (Slide 16, "LOE Breakdown M3"). Its description there: *"Bind 5 roles to server-side authorization. Role drives screen access. Remove browser-claimed persona. Audit events."* This is the exact fix for the audit's #1 blocker (§3.1, item 1: persona is just a client header today, e.g. `App.tsx`, `arc/backend/api/main.py` `PERSONA_PERMS`). **Flag as change-order risk — budgeted at 0 hours in the contract.** Keep the repo's persona model as the baseline (per Rishabh) — the work is enforcing it server-side, not redesigning it. |
| M3-04 Write Path Hardening | 16h | Saved scenarios, price-increase assumptions, transaction boundaries, idempotent re-runs |
| M3-05 Error Handling & Observability | 12h | Stop returning empty-as-success on failed queries; add loading/empty/error/unauthorized UI states; parameterize the highest-risk SQL call sites |
| **New — F8 GenAI verification** | drawn from Misc. | Confirm the Cortex Agent + semantic view deploy cleanly into the target account (agent is currently granted to `ROLE PUBLIC` — must be fixed as part of M3-02, it is the same defect class), and that F8.7 context scoping behaves. Now customer-Must scope, so it cannot be deferred. |
| Misc. Fixes | 28h | Contingency bucket — prioritize by risk, not by convenience. Now also absorbing F8 verification above. |

**Concern:** M3-02 (entitlement enforcement) is the #1 blocker in the audit and is budgeted at 0 hours. This is the single highest-risk line item in the entire App workstream and needs explicit sign-off from Brendan/Rishabh on how it will be resourced.

### Phase 3 — Integrated Testing & UAT (M5, App portion budgeted 40h) — **UAT now anchored mid-October**
- [ ] Build test plan from acceptance criteria; seed test users across the (confirmed) 4–5 roles.
- [ ] End-to-end testing: app → API → consumption layer → (ML output, if ML stays in scope), with per-role access checks and FP&A calculation spot-checks against known-good numbers.
- [ ] **New: F8 GenAI test cases.** F8.1–F8.7 are all "Must" per Michael, so they need explicit UAT cases — including Michael's own example question (*"what's our margin exposure in Asia if MMA hits EUR 2,000/t?"*) and F8.6 commercial-justification text generation.
- [ ] **New: stand up pre-UAT early access for Michael.** Brendan is confirming timing after the Sep 17 call. Because Michael is the only person who has engaged with this app, his early access is the de-facto first usability test — treat it as a milestone, not a courtesy.
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
| Repo divergence (`arc/` vs `scale/arc/`) | Audit lineage.md, Pass 0 | ~~High~~ **CLOSED Sep 16** | Resolved: `arc/` is authoritative, `scale/` to be removed from the repo. Confirm the removal actually lands. |
| Deployment target DB never created by any script | Audit Pass 0 | Medium (was High) | With `scale/` dropped, `ARKEMA_PRICING_DB` is the target. Verify `arc/ddl` builds it from empty; fix the missing `CREATE DATABASE` as part of M3-01. |
| 144× BOM fan-out with real data, no timeout/cancel | Audit Pass 4 | High once real data lands | Run "experiment #2" early; if confirmed, raise as explicit change order (13 days, per audit) rather than absorbing silently |
| Synchronous blocking runs cap concurrency at 1 correct user | Audit Pass 4 | Medium-High for multi-user UAT | Decide interim mitigation (serialize + queue) vs. full async (9.5 days, change-order candidate) before UAT starts |
| Production data schema may deviate from mock/dev schema | Contract Slide 12 (flagged High in the SOW itself) | High | Pre-kickoff schema validation is a contractual pre-req; do not assume match. Wole confirmed sources are **CSV** — get sample files the moment they exist. |
| No formal KT; Rishabh's editing cutoff | Prep meeting notes; not restated Sep 16 | High | Rishabh offered smaller-cadence deep dives — **book them immediately**. Sep 21 was never re-confirmed in the Sep 16 meeting; ask Brendan to state the cutoff date in writing so I can plan against it. |
| ~~SPCS vs. SAR unresolved~~ | Rev 1 note | **Effectively closed — SPCS** | See revised §6. Do not start SAR work. |
| **New: ML scope contradiction across SOW / PRD / customer** | Sep 16 App Review + Chase/Michael email | Medium for me, High commercially | Chase owns closure with Michael in writing. My deployment path is unaffected either way; but if ML returns, F6 recommendation screens need retesting and the beta-clamp bug (§7 item 5) becomes live scope again. Do not build against either assumption until it is written down. |
| **New: F8 GenAI is customer-Must but was never audited** | Chase/Michael email vs. Andrew's audit coverage | Medium-High | Audit F8.1–F8.7 in Phase 1. The Cortex Agent is currently granted to `ROLE PUBLIC`, which is the same auth defect class as M3-02 — fix together. |
| **New: app has never been used by any Arkema business user; usability doubted by its own author** | Sep 16 App Review | High for UAT sign-off | Capture usability risks during the Phase 1 screen walk; push for Michael's pre-UAT early access as the first real user test. This is the most likely cause of a failed UAT, and it is not a defect the audit can find. |
| **New: customer platform not Snowflake-ready; S3 bucket has no readiness date** | Wole's Sep 15 note | High — schedule risk | Plan a synthetic-data-first deployment so SPCS/deploy/RBAC work proceeds in parallel with Wole's ingestion. Do not make my Phase 2 start conditional on real data. |
| **New: InfoSec has not confirmed single-account architecture** | Wole's Sep 15 note | Medium | Michael to confirm by Thursday. If InfoSec forces multi-account, DEV/UAT/PROD promotion for the app changes and M3-01 needs re-estimating. |

---

## 6. SPCS vs. SAR (Snowflake App Runtime) — Decision: SPCS

**Status as of Sep 17: closed in favour of SPCS for this engagement.** You had noted a preference for SAR; here is why the plan does not pursue it, and what would have to change for it to.

- The **signed TDR proposal** scopes deployment explicitly to **SPCS**: the RACI table lists "Implementation of existing prototype (**deployed to SPCS**)," and LOE item M3-01 is literally "**SPCS** Setup & Deploy." The Delivery Assumptions slide also states *"App and ML deployed as-is; no new features, integrations, or **rearchitecture** in scope."*
- The **existing app repo is already built for SPCS**: Docker image, `service_spec.yaml`, compute pool, and image-repo scripts all assume SPCS. There is no SAR-oriented code (`app.yml`, Next.js/App-Runtime scaffolding) anywhere in the repo.
- **Rishabh confirmed on Sep 16 that the app runs on SPCS in his demo account** — so SPCS is a demonstrated, working path, not a theoretical one. Services' own target account deployment was not formally ratified in that meeting; get it minuted today.
- Moving to SAR would be a genuine re-platform: different deployment manifest, different runtime model for the FastAPI backend + React frontend, and a change to what "deploy as-is" contractually means. Per the SOW's own assumptions, this reads as a **rearchitecture** and would need a **change order**, not something to fold into the current 64-hour App budget. Against a mid-October UAT and an app that no user has yet touched, spending budget on re-platforming instead of on auth, correctness, and usability would be the wrong trade.
- **If you still want SAR,** the honest framing is a *post-MVP* item: raise it as a follow-on/phase-2 discussion with Brendan and Chase after go-live, not as a substitution inside this engagement. I have not built any part of this plan around a SAR migration.

---

## 6a. Confirmed Decisions Register (as of Sep 17)

| Decision | Value | Decided by / where | Firmness |
|---|---|---|---|
| Authoritative repo tree | `arc/` (`scale/` to be deleted) | Rishabh, Sep 16 App Review | Firm |
| Authoritative PRD | current MVP PRD (`arc/docs/MVPbridge...`); ignore all `app_requirements/` versions | Rishabh, Sep 16 | Firm |
| Deployment mechanism | **SPCS** | SOW + repo + Rishabh's working demo | Firm (Services target account not yet minuted) |
| RBAC model | repo's persona-based model as baseline; do not reopen with client | Rishabh, Sep 16 | Firm |
| GenAI / Cortex Agent + Cowork | **IN scope**, F8.1–F8.7 all "Must" | Michael THAI, written email Sep 16 | Firm |
| ML in MVP | **UNRESOLVED** — PRD says none, SOW assumed it, Michael says keep POC ML *if cheap*, F6.1 alone acceptable | Conflicting | **Open — Chase to close in writing** |
| Target database | `ARKEMA_PRICING_DB` (by implication) | inferred, not stated | **Confirm today** |
| Snowflake account topology | single account, logical DEV/UAT/PROD | Wole + team, Sep 15 | Pending InfoSec (Michael, by Thursday) |
| Source file format | CSV from AWS S3 | Wole, Sep 15 | Firm; bucket not ready |
| UAT | mid-October, 2 weeks, Arkema-driven | Sep 16 App Review | Firm-ish |
| Rishabh editing cutoff | Sep 21 (from prep meeting; **not** restated Sep 16) | unconfirmed | **Confirm with Brendan** |

---

## 7. Questions — Status After Sep 16, and What to Ask Today

**Answered on Sep 16:**
1. ~~Which tree is authoritative?~~ → **`arc/`.** `scale/` is POC-era and will be removed.
2. Which database is the real production target? → Not stated outright; `ARKEMA_PRICING_DB` by implication once `scale/` is dropped. **Still needs an explicit yes.**
8. ~~SAR vs. SPCS?~~ → **SPCS** (see §6).

**Still open — carry into today's meeting:**

3. Has this app ever been run end-to-end against a freshly built (non-synthetic) database? Andrew's audit suggests no, and nothing on Sep 16 contradicted that. Ask directly, because if the answer is no, the first clean rebuild is a discovery risk I need budgeted time for.
4. Given M3-02 (entitlement enforcement) is budgeted at 0 hours but is the top blocker in the audit — how should this be resourced? Is a change order expected? **Note: none of the audit findings (auth, wrong numbers, concurrency, BOM fan-out) were raised with Rishabh on Sep 16.** That was a gap in that meeting and is the single most important thing to correct today.
5. Does Arkema know the specific pricing/margin numbers currently shown are not reliable (audit Pass 3)? Does "deploy as-is" cover shipping them unchanged, or is there tolerance to fix clearly-broken arithmetic under "productionalization" rather than "new feature"?
   - Useful new context: Rishabh confirmed the eye-catching *"price increases of like 77%"* recommendations **were his own POC additions, not a customer requirement** — *"that was added by me during the POC. It never came from him."* That makes it much easier to argue these are prototype artifacts to correct, not contracted behaviour to preserve.
   - **Where to find this in the code:** `arkema-pricing-ss/arc/ddl/014_nlc2_agent_recommend.sql`. The elasticity gate is `if beta < -1.0:` (line 48), right below where `beta` is clamped to a floor of exactly `-1.0` by the `shrink_beta()` function (line 33). Because the clamp's output can never be *less than* `-1.0`, the `<` condition on line 48 can never be true, so the code path that uses the ML elasticity (`p_unc`, line 49) never executes.
   - **Where the "not reliable" numbers surface in the UI:** the margin/recommendation screens driven by `arc/backend/services/nlc2_service.py` (reads `ML.NLC2_ELASTICITY`, line ~396) and the stored procedure `SP_BUILD_RECOMMENDATIONS` referenced in `arc/docs/ARCHITECTURE.md`.
6. Is real ZFSC07 (BOM) data available yet to test the 144× fan-out before UAT? Wole's note says the S3 bucket is still being configured with no date — so the practical question is: **can Arkema hand over a handful of representative CSV extracts by email now**, ahead of the pipeline?
7. Confirm target go-live date. UAT is mid-October; the App workstream cannot connect to production data until Wole's storage integration + stage land. What is the committed date for "enough data to unblock development"?

**New questions for today:**

9. What exactly is the deployment target *account* for Services — Rishabh's demo account, the provisioned Arkema trial account, or a Snowflake-internal sandbox? And who owns the compute pool / image repo there?
10. Confirm Rishabh's editing cutoff date in writing, and get the deep-dive sessions on the calendar before it. I need at least two: one on data model + DDL build order, one on UI/persona flow and the Cortex Agent wiring.
11. When can I get Rishabh's demo-account access? (He agreed; not yet delivered.)
12. F8 GenAI: which parts of F8.1–F8.7 does the existing agent actually satisfy, and was F8.7 (context scoping to a simulation run / price scenario) ever implemented? Rishabh's in-UI assistant is beyond-PRD and does not run simulations — so what does it do, and what is missing?
13. Pre-UAT early access for Michael: what date is Brendan committing to, and against which data (synthetic or real)?
14. Who owns writing the UAT acceptance criteria, given no business user has ever used the app? If nobody has, this needs an owner today.

---

## 7a. Agenda for Today's App Meeting (Sep 17)

Fifteen minutes of decisions, then everything else. Suggested order:

1. **Minute the closed decisions** (§6a) so they stop being reopened — tree, PRD, RBAC baseline, SPCS.
2. **Raise the audit findings with Rishabh — this is the priority.** They were not discussed on Sep 16. Lead with the two that change the plan: no server-side auth (fails open to a write-capable persona), and the correctness findings. Frame as "here is what Services will hit in deployment," not as a critique of the prototype.
3. **Confirm target account + target database**, and who owns compute pool / image repo.
4. **Book the deep-dive sessions and confirm Rishabh's cutoff date.** Hardest deadline in my workstream — after the cutoff, the repo is my only source.
5. **Collect deliverables Rishabh owes:** demo-account access, consolidated deploy script, `scale/` removal.
6. **Flag the two things nobody owns yet:** UAT acceptance criteria, and usability validation with a real business user.
7. **State my plan of record:** proceed synthetic-data-first on SPCS so deployment, RBAC, and error-handling work is not blocked by S3 readiness.

---

## 8. Coming Steps (next 5 working days)

| When | Action | Owner | Blocks |
|---|---|---|---|
| Today, Sep 17 | Run the agenda above; get target account/DB confirmed and audit findings on the record | William | Phase 1 start |
| Today, Sep 17 | Request demo-account access + consolidated deploy script from Rishabh | William | orientation |
| Today, Sep 17 | Book 2× deep-dive sessions with Rishabh before his cutoff | William | all KT |
| Sep 17–18 | Ask Brendan to confirm in writing: Rishabh cutoff date, pre-UAT early-access date, and how M3-02 (0h) will be resourced | William → Brendan | change-order path |
| Sep 18 | Michael to confirm InfoSec position on account isolation (committed "by Thursday"); Wole needs S3 path + IAM role | Michael / Wole | ingestion, env topology |
| Sep 18–19 | Pin baseline commit in `arc/`; build the DB from `arc/ddl` on empty and record every gap (missing `CREATE DATABASE`, ordering failures) | William | M3-01 |
| Sep 18–19 | Ask for a handful of representative CSV extracts by email, independent of the S3 pipeline | William → Michael via Brendan | BOM fan-out test |
| Sep 19–22 | Walk all 16 screens × 4 personas; produce the F8.1–F8.7 gap list and the per-persona usability risk list | William | Phase 2 scoping, UAT criteria |
| Ongoing | Track Chase's ML-scope email to written closure | Chase (I am Informed) | Luis/Sharanya workstream |

**Sequencing principle for the next two weeks:** treat Rishabh's remaining availability as the scarcest resource in the engagement — spend it on knowledge that cannot be recovered from the repo (why decisions were made, what is mocked, what the UI intends), not on things I can read myself.

---

## 9. Open Items / My Concerns Summary

- **Biggest concern, unchanged and now more urgent:** the contract's "deploy as-is, no code modifications" framing does not accommodate the audit's top findings (no auth, wrong numbers, blocking runtime). **These findings were not raised in the Sep 16 meeting at all** — so as of now, the delivery team is still planning around an app whose known blockers have not been acknowledged by its author or priced by the SDM. Surface today, in writing, before committing to mid-October UAT.
- **New top-3 concern: usability.** No Arkema business user has ever used this app, the intended users are Excel-native, and Rishabh himself questioned who it serves. Nothing in the SOW, the audit, or the plan currently owns validating that the app is usable — and that is the most likely reason UAT fails in October. Needs an owner and needs Michael in front of the app early.
- **Scope drift is now bidirectional:** ML came *out* (per PRD) while GenAI/Cortex was reaffirmed *in* (per Michael), and the SOW matches neither exactly. Every scope conversation should reference the current MVP PRD plus Michael's written F8 confirmation, not the SOW alone.
- **Schedule concern:** my critical path runs through work I do not control — Wole's S3 ingestion (no bucket date), Michael's InfoSec confirmation, and Rishabh's remaining hours. Mitigation is to decouple: synthetic-data-first deployment so Phase 2 is never idle waiting on data.
- **Resolved since Rev 1, and worth banking:** repo divergence, authoritative PRD, RBAC baseline, and SPCS-vs-SAR are all now settled. That is four High/Medium risks retired in one meeting.
- Everything above is sourced directly from the repo, the audit, the signed proposal deck, the Sep 16 transcript, Michael's email, and Wole's note — no numbers or findings in this plan were invented.

---

*This is a local planning document only. No Snowflake database objects were created, altered, or queried in producing this plan.*
