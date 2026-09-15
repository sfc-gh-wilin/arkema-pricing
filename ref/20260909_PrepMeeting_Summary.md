# Arkema Prep Meeting Summary — September 9, 2026

## Attendees
- **Brendan Owens** (SDM / Lead, Snowflake)
- **William Lin** (Snowflake) — joined late
- **Wole Babalola** (Snowflake — Platform stream)
- **Sharanya Krishnamurthi** (Snowflake — ML stream)
- **Luis Villavicencio** (Snowflake — ML stream)

---

## Overall Project Context

- **Customer:** Arkema — not yet a contracted customer; currently on an on-demand/trial account pending CAP1 signature.
- **Goal:** Deploy a **Pricing Cockpit / Pricing Agent app** built by Rashab (Snowflake Industry team) to Arkema's production environment. This app is their internal pitch to leadership for a larger Snowflake contract centered around SAP.
- **Background:** PHData was originally going to deploy the app but backed out. Snowflake Services stepped in last-minute.
- **Brendan** is acting as temporary SDM; a full-time SCM will be assigned to manage customer coordination going forward.

---

## Three Work Streams

| Stream | Owner | Description |
|---|---|---|
| **Platform Foundation** | Wole Babalola | Security setup, RBAC/roles, data ingestion from S3, account provisioning |
| **ML** | Sharanya Krishnamurthi + Luis Villavicencio (50/50 split) | Validate, configure, and run ML features within the app |
| **App Deployment** | **William Lin** | Deploy the app to the production environment |

---

## William Lin — Role & Key Points

- **Primary responsibility: App deployment to production.**
- William joined late (~17:33) and confirmed his understanding directly with Brendan:
  > *"What I capture is: we just leave the app as it is, no addition to it."* — William Lin
  > *"Correct."* — Brendan Owens
- **William will be in charge of deploying the app** once Rashab's work is handed off and access is granted.
- William asked Brendan for the meeting transcript at the end of the call (confirmed he would receive it).
- William, Sharanya, and Luis all **start next week** (week of ~Sep 14), ramping at ~10 hours. Wole is already working this week on platform foundation.
- A **kick-off/walkthrough with Rashab is scheduled for September 14** to review the app.

---

## APP (Pricing Cockpit / Pricing Agent) — Key Points

- Built by **Rashab** (Snowflake Industry team). Rashab is currently out and returns next week.
- **No new feature requests will be accepted.** Any ask from the account team is an automatic no. Brendan will reinforce this.
- **Rashab stops editing the app on September 21.** After that, the app is handed off to the Services team for production deployment.
- There is **no formal Knowledge Transfer (KT)**. The team must self-serve from the Git repo, which contains artifacts (RBAC definitions, access roles, etc.).
- The app should be **taken as-is** — no edits to the app itself, no customer input on app content. Only the platform/RBAC setup involves customer input.
- Next week's focus: **get access, explore the repo, understand what's in the app**, and prepare for the Sep 14 walkthrough with Rashab.

---

## Platform / Access Status (Wole's Update)

- Morning calls covered: SSO/SCIM setup and S3 bucket configuration for data ingestion.
- Michael (customer contact) indicated **no additional data transformation is expected** — ingest data as-is from AWS.
- Account structure (single vs. multiple accounts for dev/prod separation) is still being discussed; Michael will provide guidance.
- **Access to the environment is not yet provisioned.** Blocker for the whole team.
  - ETA: unknown. Brendan will follow up.
  - Escalation path: ping **Michael** directly if access is needed urgently.
  - Wole will send a follow-up email to Michael (cc Brendan) regarding account setup.

## RBAC Scope

- Contracted for **5 business personas (user roles)** — not 5 technical/functional roles.
- Functional roles can be created as needed to support those 5 personas.
- RBAC definition should align with what is already in the repo, with confirmation from Michael on whether to replicate exactly or adjust.

---

## Action Items

| Owner | Action |
|---|---|
| **Brendan** | Share meeting transcript with all attendees |
| **Brendan** | Send follow-up on environment access ETA |
| **Brendan** | Coordinate Sep 14 app walkthrough with Rashab and full team |
| **Wole** | Send follow-up email to Michael (cc Brendan) re: account setup |
| **William** | Get environment access when provisioned; begin app deployment prep next week |
| **Sharanya / Luis** | Get access; explore repo and ML components; align on task split end of next week |
| **All** | Join Sep 14 walkthrough with Rashab to review the app |

---

## Key Dates

| Date | Event |
|---|---|
| Week of Sep 14 | William, Sharanya, Luis officially start (~10 hrs ramp) |
| **Sep 14** | App walkthrough session with Rashab |
| **Sep 21** | Rashab stops editing; app handed off to Services team |
| TBD | Production deployment (William) |
