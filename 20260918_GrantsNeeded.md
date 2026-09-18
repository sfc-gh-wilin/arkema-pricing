# Arkema Snowflake — Grants Needed (App Productionalization)

**Date:** September 18, 2026
**Requestor:** William Lin (Snowflake PS — App Productionalization workstream)
**Account:** `BR68104` · region `AWS_EU_WEST_1` · org region group AWS EU West 1
**My user:** `A6347057@ARKEMA.COM`
**My only role:** `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` (AAD/SCIM-provisioned; note the hyphens — **must be double-quoted** in all SQL)
**Target:** deploy the ARC / Pricing Cockpit app to this account, on **Snowflake App Runtime (SAR)**

> **Ask in one line:** the role above currently has **zero object privileges**. Nothing can be deployed until an `ACCOUNTADMIN` runs §3 and §4.

---

## 1. What I verified in the account (read-only, Sep 18)

No objects were created, altered, or dropped. Evidence from `SHOW` commands run as my role:

| Check | Result |
|---|---|
| `SHOW GRANTS TO ROLE "EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` | **0 rows** — the role holds no privileges on any object |
| `CURRENT_AVAILABLE_ROLES()` | `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"`, `PUBLIC` — nothing else |
| `SHOW DATABASES` | Only `SNOWFLAKE` (shared app DB) and my personal database `USER$A6347057@ARKEMA.COM`. **`ARKEMA_PRICING_DB` does not exist / is not visible.** |
| `SHOW WAREHOUSES` | Only `SYSTEM$STREAMLIT_NOTEBOOK_WH` (owned by `ACCOUNTADMIN`). **No `ARC_COMPUTE_WH`.** |
| `SHOW APPLICATION SERVICES IN ACCOUNT` | 0 rows |
| `SHOW EXTERNAL ACCESS INTEGRATIONS` | 0 rows visible |
| `SHOW PARAMETERS LIKE '%APPS%' IN ACCOUNT` | `DEFAULT_SNOWFLAKE_APPS_DESTINATION_DATABASE`, `..._DESTINATION_SCHEMA`, `..._QUERY_WAREHOUSE`, `..._SERVICE_COMPUTE_POOL` are **all empty** → **Snowsight App Development Setup has never been run in this account** |

**Consequence of the last row:** until setup runs, `snow app setup` / `snow app deploy` resolve to my **personal database** (`USER$A6347057@ARKEMA.COM`). Apps deployed there work, but **cannot be shared with any other role** — `GRANT USAGE ON APPLICATION SERVICE` is not supported in a personal database. That makes a personal-DB deploy useless for UAT.

---

## 2. Read this before granting — three hard platform facts about SAR

These are current documented SAR limitations, not opinions. They change the scope of the conversion, so they belong in the same document as the grant request.

1. **SAR runs Node.js only.** Deployable projects are Node.js (typically Next.js). **Python support is "planned", not available.** The existing app is a **FastAPI/Python backend (~4,400 LOC, 93 route decorators across 5 modules) + a Vite/React frontend**. Converting to SAR therefore means **re-implementing the entire backend in Node/Next.js** — it is not a repackaging exercise. There is no lift-and-shift path for the Python tier today.
2. **SAR is not available on trial accounts.** The project plan records Arkema as an on-demand/trial account pending CAP1. `BR68104` must be a **paid** account before any SAR deploy, and account admin setup itself lists a paid account as a prerequisite. **This needs confirming before the grants below are worth requesting.**
3. **SAR gives you no compute pool or image repository to manage.** There is no `compute_resource`/compute-pool field in `app.yml`; scaling is `min_instances`/`max_instances` and the artifact repository is provisioned by `snow app deploy`. So **none** of the existing `deploy/spcs_setup.sql` grants (`USAGE ON COMPUTE POOL`, `READ ON IMAGE REPOSITORY`) apply to SAR. Standard SPCS commands (`CREATE SERVICE`, `ALTER SERVICE`) do not work on Application Services.

> Also note the repo's contracted path is **SPCS**, and the signed SOW scopes deployment to SPCS ("M3-01 SPCS Setup & Deploy"). §8 of the current app already exists for SPCS (Dockerfile, `service_spec.yaml`, compute pool, image repo). A SAR move is a re-platform. **§7 of this doc lists the SPCS grant set as well**, so whichever path is chosen, the request to Arkema IT is ready.

---

## 3. Preferred path — one Snowsight action by ACCOUNTADMIN

This is the supported, lowest-effort route and it issues the correct grants automatically.

**Ask the Arkema Snowflake account admin to:**

1. In Snowsight: **profile menu (lower-left) → Settings → Account → Apps → Begin Setup**.
2. Under **"What roles will be making apps?"**, select **`ARC_APP_PUBLISHER`** (see §5 — please create this role rather than selecting the AAD role directly).
3. Under **Resources**, choose **Quick start** (creates `SNOWFLAKE_APPS` database, `PUBLIC` schema, `SNOWFLAKE_APPS_QUERY_WH` warehouse) **or Custom** if Arkema naming standards require specific names. If Custom, tell me the three names so I can pin them in `app.yml`.
4. **Preview SQL → Execute Setup.**

That sets the `DEFAULT_SNOWFLAKE_APPS_*` account parameters and grants the chosen role what it needs. Nothing in §4 is then required.

---

## 4. Fallback path — explicit SQL if Snowsight setup is not permitted

Run as `ACCOUNTADMIN` (or `SYSADMIN`/`SECURITYADMIN` as noted). Substitute the real database/schema/warehouse if Arkema standards differ from the defaults.

```sql
-- ---------------------------------------------------------------
-- 4a. Deployment destination (run as SYSADMIN or ACCOUNTADMIN)
-- ---------------------------------------------------------------
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_APPS;
CREATE SCHEMA   IF NOT EXISTS SNOWFLAKE_APPS.PUBLIC;

CREATE WAREHOUSE IF NOT EXISTS SNOWFLAKE_APPS_QUERY_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND   = 60
  AUTO_RESUME    = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- ---------------------------------------------------------------
-- 4b. Account defaults so `snow app setup` stops resolving to my
--     personal database (run as ACCOUNTADMIN)
-- ---------------------------------------------------------------
ALTER ACCOUNT SET DEFAULT_SNOWFLAKE_APPS_DESTINATION_DATABASE = 'SNOWFLAKE_APPS';
ALTER ACCOUNT SET DEFAULT_SNOWFLAKE_APPS_DESTINATION_SCHEMA   = 'PUBLIC';
ALTER ACCOUNT SET DEFAULT_SNOWFLAKE_APPS_QUERY_WAREHOUSE      = 'SNOWFLAKE_APPS_QUERY_WH';

-- ---------------------------------------------------------------
-- 4c. Deploy privileges for the publisher role
-- ---------------------------------------------------------------
GRANT USAGE ON DATABASE SNOWFLAKE_APPS                        TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON SCHEMA   SNOWFLAKE_APPS.PUBLIC                 TO ROLE ARC_APP_PUBLISHER;
GRANT CREATE APPLICATION SERVICE  ON SCHEMA SNOWFLAKE_APPS.PUBLIC TO ROLE ARC_APP_PUBLISHER;
GRANT CREATE ARTIFACT REPOSITORY  ON SCHEMA SNOWFLAKE_APPS.PUBLIC TO ROLE ARC_APP_PUBLISHER;
GRANT CREATE STAGE                ON SCHEMA SNOWFLAKE_APPS.PUBLIC TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON WAREHOUSE SNOWFLAKE_APPS_QUERY_WH              TO ROLE ARC_APP_PUBLISHER;
```

| Privilege | On | Why I need it |
|---|---|---|
| `USAGE` | database, schema | Resolve and address the deployment target |
| `CREATE APPLICATION SERVICE` | schema | Create the app service during `snow app deploy` |
| `CREATE ARTIFACT REPOSITORY` | schema | `snow app deploy` provisions one repository per app |
| `CREATE STAGE` | schema | Per-app code stage, owned by the deploying role |
| `USAGE` | warehouse | Run the SQL the deploy pipeline issues |

**Also please confirm (do not blindly grant):**
- `BIND SERVICE ENDPOINT ON ACCOUNT` — needed to expose a public app endpoint. It is granted to `PUBLIC` by default; if Arkema security has revoked it from `PUBLIC`, `ARC_APP_PUBLISHER` needs it explicitly.
- Whether a **feature policy** blocks `APPLICATION_SERVICES` / `ARTIFACT_REPOSITORIES` in this account or on personal databases. If one exists, the grants above will not be sufficient.

---

## 5. Role design — please create a dedicated publisher role

Do **not** attach these privileges directly to `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"`. That role is provisioned by `AAD_PROVISIONER` (visible in `SHOW GRANTS TO USER`), so it is owned by the Entra ID/SCIM pipeline, and its name binds the app's identity to an Azure group rather than to this project.

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS ARC_APP_PUBLISHER
  COMMENT = 'Deploys and owns the ARC Pricing Cockpit Application Service';

USE ROLE SECURITYADMIN;
-- Reachable from my existing AAD role, so SCIM churn does not break my access
GRANT ROLE ARC_APP_PUBLISHER TO ROLE "EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD";
GRANT ROLE ARC_APP_PUBLISHER TO ROLE SYSADMIN;  -- standard hierarchy hygiene
```

Two reasons this matters, both consequential and hard to undo:
- **The execution role is fixed at creation.** For an app in a standard database, the execution role **is the creating session's primary role**, and it **cannot be changed later** — changing it requires dropping and recreating the service. Deploying under the AAD role would permanently bind the app's execution identity to an Azure group.
- Objects created during deploy (stage, artifact repository, service) are owned by the deploying role, so ownership lands somewhere intentional.

I will deploy with `ARC_APP_PUBLISHER` as **primary** role with secondary roles disabled.

---

## 6. Data-access grants for the running app

The app must read the ARC data layer. **`ARKEMA_PRICING_DB` does not exist in this account yet** — it has to be built from `arc/ddl` (and the repo is missing its `CREATE DATABASE`, a known gap). So treat this section as the grant set to apply *once the data layer lands*, and substitute the final database name.

Pick one execution model. I recommend **owner's rights** for MVP, because the app's persona model is currently a client-supplied header with no server-side enforcement (audit blocker #1) — caller's rights would not fix that and adds a second failure mode.

### 6a. Owner's rights (recommended for MVP)

```sql
SET APP_DB = 'ARKEMA_PRICING_DB';  -- confirm final name

GRANT USAGE  ON DATABASE ARKEMA_PRICING_DB                        TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE  ON ALL SCHEMAS IN DATABASE ARKEMA_PRICING_DB         TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE  ON FUTURE SCHEMAS IN DATABASE ARKEMA_PRICING_DB      TO ROLE ARC_APP_PUBLISHER;
GRANT SELECT ON ALL TABLES    IN DATABASE ARKEMA_PRICING_DB       TO ROLE ARC_APP_PUBLISHER;
GRANT SELECT ON FUTURE TABLES IN DATABASE ARKEMA_PRICING_DB       TO ROLE ARC_APP_PUBLISHER;
GRANT SELECT ON ALL VIEWS     IN DATABASE ARKEMA_PRICING_DB       TO ROLE ARC_APP_PUBLISHER;
GRANT SELECT ON FUTURE VIEWS  IN DATABASE ARKEMA_PRICING_DB       TO ROLE ARC_APP_PUBLISHER;

-- Write paths: scenario save, overrides, governance actions
GRANT INSERT, UPDATE, DELETE ON ALL TABLES    IN SCHEMA ARKEMA_PRICING_DB.GOV TO ROLE ARC_APP_PUBLISHER;
GRANT INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA ARKEMA_PRICING_DB.GOV TO ROLE ARC_APP_PUBLISHER;

-- Simulation / recommendation stored procedures
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.ATOMIC TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.ATOMIC TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.GOV    TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.GOV    TO ROLE ARC_APP_PUBLISHER;
```

### 6b. Cortex Agent / semantic view (F8 GenAI — customer-confirmed "Must")

The app calls a Cortex Agent (`ARC_AGENT`, per `service_spec.yaml`, in `SNOWFLAKE_INTELLIGENCE.AGENTS`). Neither that database nor the agent exists in `BR68104` yet.

```sql
GRANT USAGE ON DATABASE SNOWFLAKE_INTELLIGENCE          TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON SCHEMA   SNOWFLAKE_INTELLIGENCE.AGENTS   TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE ON AGENT    SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_AGENT TO ROLE ARC_APP_PUBLISHER;
-- Semantic view backing Cortex Analyst (confirm final name)
GRANT SELECT ON SEMANTIC VIEW ARKEMA_PRICING_DB.MART.<SEMANTIC_VIEW> TO ROLE ARC_APP_PUBLISHER;
```

> **Do not reproduce the repo's `GRANT ... TO ROLE PUBLIC` on the agent.** The POC granted `ARC_AGENT` to `PUBLIC`; that is the same auth defect class as audit blocker #1 and must not be carried into this account.

### 6c. If caller's rights is chosen instead

Requires `ACCOUNTADMIN` or `MANAGE CALLER GRANTS`. Grant to the **execution role**, not to end users. Without the warehouse grant, every caller's-rights query fails even when table grants are correct.

```sql
GRANT CALLER DATA READ  ON DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_PUBLISHER;
GRANT CALLER DATA WRITE ON DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_PUBLISHER;
GRANT CALLER USAGE ON WAREHOUSE SNOWFLAKE_APPS_QUERY_WH TO ROLE ARC_APP_PUBLISHER;

SHOW CALLER GRANTS TO ROLE ARC_APP_PUBLISHER;  -- verify
```

### 6d. External egress (map tiles + fonts)

The frontend loads OpenStreetMap tiles and Google Fonts. Under SAR these go in `app.yml` → `external_access_integrations`, and the deploying role needs `USAGE` on each integration. Reuse the host list already in `deploy/network_rules.sql`.

```sql
-- ACCOUNTADMIN creates the network rules + integration, then:
GRANT USAGE ON INTEGRATION ARC_EXTERNAL_ACCESS TO ROLE ARC_APP_PUBLISHER;
```

Separately: SAR **remote builds** have scoped outbound access by default. If Arkema has disabled that, npm/Google-Fonts fetches during build will fail and I will need a `build_eai` integration granted as well. **Please confirm whether default build egress is enabled.**

---

## 7. If the decision stays SPCS (contracted path)

Same request, different object set. Included so one conversation with Arkema IT covers both.

```sql
-- Infra (ACCOUNTADMIN)
CREATE COMPUTE POOL IF NOT EXISTS ARC_COMPUTE_POOL
  MIN_NODES = 1 MAX_NODES = 1 INSTANCE_FAMILY = CPU_X64_S
  AUTO_RESUME = TRUE AUTO_SUSPEND_SECS = 3600;
CREATE IMAGE REPOSITORY IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_IMAGES;

-- Grants to ARC_APP_PUBLISHER
GRANT USAGE  ON DATABASE ARKEMA_PRICING_DB          TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE  ON SCHEMA   ARKEMA_PRICING_DB.SPCS     TO ROLE ARC_APP_PUBLISHER;
GRANT CREATE SERVICE ON SCHEMA ARKEMA_PRICING_DB.SPCS TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE, MONITOR, OPERATE ON COMPUTE POOL ARC_COMPUTE_POOL TO ROLE ARC_APP_PUBLISHER;
GRANT READ, WRITE ON IMAGE REPOSITORY ARKEMA_PRICING_DB.SPCS.ARC_IMAGES TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE  ON INTEGRATION ARC_EXTERNAL_ACCESS     TO ROLE ARC_APP_PUBLISHER;
GRANT BIND SERVICE ENDPOINT ON ACCOUNT              TO ROLE ARC_APP_PUBLISHER;
GRANT USAGE  ON WAREHOUSE ARC_COMPUTE_WH            TO ROLE ARC_APP_PUBLISHER;
```
Plus everything in §6a–6d (the data-layer grants are identical; `WRITE` on the image repository is needed to push the built image, which the SAR path does not require).

Note `deploy/spcs_setup.sql` currently grants all of this to `SYSADMIN` and runs `USE DATABASE ARKEMA_PRICING_DB` before that database exists. **Do not run it as-is in the Arkema account.**

---

## 8. Warehouse for building the ARC data layer

Independent of app runtime: to build `ARKEMA_PRICING_DB` from `arc/ddl` and to run simulations, I need a warehouse I can actually use. Today I can see only `SYSTEM$STREAMLIT_NOTEBOOK_WH`.

```sql
CREATE WAREHOUSE IF NOT EXISTS ARC_COMPUTE_WH
  WAREHOUSE_SIZE = 'SMALL' AUTO_SUSPEND = 300 AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 3600;  -- see concern below

GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE ARC_COMPUTE_WH TO ROLE ARC_APP_PUBLISHER;
```

If I am to build the data layer myself rather than hand DDL to Wole, I additionally need either `CREATE DATABASE ON ACCOUNT`, or ownership of a pre-created `ARKEMA_PRICING_DB`:

```sql
-- Option A (preferred): admin creates it, I own it
CREATE DATABASE IF NOT EXISTS ARKEMA_PRICING_DB;
GRANT OWNERSHIP ON DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_PUBLISHER;

-- Option B: I create it
GRANT CREATE DATABASE ON ACCOUNT TO ROLE ARC_APP_PUBLISHER;
```

**The `STATEMENT_TIMEOUT_IN_SECONDS` above is deliberate.** The audit found a BOM fan-out that can multiply scans up to 144× on real ZFSC07 data with no timeout and no cancel path — a runaway simulation burns credits for hours. A warehouse-level timeout is the cheapest guardrail and costs nothing to set now.

---

## 9. How to verify the grants landed (I will run these)

```sql
USE ROLE ARC_APP_PUBLISHER;
SHOW GRANTS TO ROLE ARC_APP_PUBLISHER;
SHOW PARAMETERS LIKE '%SNOWFLAKE_APPS%' IN ACCOUNT;   -- destination should no longer be empty
SHOW DATABASES;                                        -- SNOWFLAKE_APPS should appear
SHOW WAREHOUSES;
```
Then, from the repo: `snow app setup --app-name=arc --dry-run` — it must resolve to `SNOWFLAKE_APPS.PUBLIC`, **not** `USER$A6347057@ARKEMA.COM`.

---

## 10. Concerns I want on the record

Ordered by how much they can cost.

1. **SAR cannot run this app today.** SAR is Node.js-only; Python is planned, not shipped. The app is a ~4,400-LOC FastAPI backend with 93 route decorators. A SAR conversion is a **full backend rewrite**, not a repackaging, and it is not covered by the SOW's "deploy as-is, no rearchitecture" language. This is a change-order conversation, and against a mid-October UAT I do not think the budget supports it. **I need an explicit decision from Brendan/Chase before I spend hours on SAR grants.** My own Rev 2 plan (§6) closed this in favour of SPCS for exactly these reasons.
2. **Paid-account prerequisite is unconfirmed.** SAR is unavailable on trial accounts and account admin setup requires a paid account. The plan records Arkema as on-demand/trial pending CAP1. If `BR68104` is still a trial, **§3 and §4 cannot be executed at all** and the SAR question is moot until CAP1 closes. This should be checked first, before anyone is asked for grants.
3. **My role has zero privileges and it is SCIM-managed.** I cannot create a database, a warehouse, or a schema. Every single item in this document is blocked on someone else. Please also confirm who the `ACCOUNTADMIN` holder is and what their turnaround is — my critical path now runs entirely through them.
4. **Personal-database deploys are a dead end for UAT.** With the account defaults empty, any deploy I attempt today lands in my personal database, where `GRANT USAGE ON APPLICATION SERVICE` is not supported. Michael's pre-UAT early access is impossible without §3.
5. **Don't carry the POC's `PUBLIC` grants into this account.** `ARC_AGENT` was granted to `ROLE PUBLIC` in the POC, and `deploy/spcs_setup.sql` grants broadly to `SYSADMIN`. Both should be replaced by the scoped `ARC_APP_PUBLISHER` model here. Getting this right at grant time is far cheaper than retrofitting it during UAT.
6. **Execution role is immutable.** Whatever role first creates the Application Service becomes its execution identity permanently. Deploying once "just to test" under the wrong role means a drop and recreate. This is why §5 comes before any deploy attempt.
7. **Data-layer grants (§6) are currently unactionable.** `ARKEMA_PRICING_DB` does not exist in this account, and the repo has no `CREATE DATABASE` for it. §6 is a forward-looking list; it needs the final database/schema names from Wole's platform workstream before it can be sent to IT as a concrete request.

---

*Read-only verification only. No Snowflake objects were created, altered, or dropped in producing this document.*
