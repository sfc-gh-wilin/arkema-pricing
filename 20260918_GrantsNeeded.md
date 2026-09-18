# Arkema Snowflake — Grants Needed (App Productionalization, SPCS)

**Date:** September 18, 2026
**Requestor:** William Lin (Snowflake PS — App Productionalization workstream)
**Account:** `BR68104` · region `AWS_EU_WEST_1`
**My user:** `A6347057@ARKEMA.COM`
**My only role:** `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` (AAD/SCIM-provisioned; note the hyphens — **must be double-quoted** in all SQL)
**Deployment target:** **Snowpark Container Services (SPCS)** — per the signed SOW (M3-01 "SPCS Setup & Deploy") and confirmed Sep 18. *SAR was evaluated and ruled out: it runs Node.js only, and the ARC backend is Python. See `others/20260915_ProjectPlan_SAR.md`.*

> **Ask in one line:** the role above currently has **zero object privileges**. Nothing can be built or deployed until an `ACCOUNTADMIN` actions §3–§6.

> **Sending this to Arkema?** Send **§0 only**. It is self-contained and written for a non-specialist. Sections 1–10 are my internal working notes.

---

# §0. For Arkema IT — What We Need You To Do

**Who this is for:** whoever administers the Arkema Snowflake account `BR68104` (needs the `ACCOUNTADMIN` role).
**Time needed:** about 45 minutes, plus one short decision (Step 2).
**Why:** the Snowflake Professional Services team needs a workspace and permissions in your Snowflake account before we can build and deploy the Pricing Cockpit application. Right now our account has no permissions at all, so no work can start.

There are **nine steps**. Steps 3–7 are SQL you copy, paste and run; Steps 1, 2, 8 and 9 need a short answer from you. You do **not** need to understand the application or the SQL.

---

### Step 1 — Confirm the account is a paid account

Please confirm `BR68104` is a **paid** Snowflake account and not a trial account. Just a yes/no in your reply is fine.

*(Why we ask: some Snowflake features are unavailable on trial accounts. This affects what we can build.)*

---

### Step 2 — Two decisions we need from you

Please tell us your preference. If you have no preference, say so and we will use the defaults.

| # | Decision | Default we suggest |
|---|---|---|
| 2a | **Names for the new objects.** We suggest `ARKEMA_PRICING_DB` (database), `ARC_COMPUTE_WH` (warehouse), `ARC_APP_DEPLOYER` (role). If Arkema has naming standards, give us the names you want instead and we will adjust the script. | Use the suggested names |
| 2b | **Internet access for the app's maps and fonts.** The application shows a map and uses web fonts, which means it needs to reach eight public internet addresses (listed in Step 5). **Your security team should approve this list.** If they refuse, tell us — the map and fonts will not display, and we will plan around it. | Approve the eight addresses |

---

### Step 3 — Create a role for our team

This creates a dedicated role for the project, so our permissions are contained and easy to remove later.

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS ARC_APP_DEPLOYER
  COMMENT = 'Snowflake PS - builds and deploys the ARC Pricing Cockpit application';

USE ROLE SECURITYADMIN;
GRANT ROLE ARC_APP_DEPLOYER TO ROLE "EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD";
GRANT ROLE ARC_APP_DEPLOYER TO ROLE SYSADMIN;
```

*What this does: creates a project role and makes it usable by our existing Arkema login, and visible to your system administrators.*

---

### Step 4 — Create the database and the compute

This creates the workspace where the application's data will live, and the compute resource that runs its queries.

```sql
USE ROLE SYSADMIN;

CREATE DATABASE IF NOT EXISTS ARKEMA_PRICING_DB;

CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.RAW;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.ATOMIC;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.MART;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.ML;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.GOV;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.DOCS;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.SPCS;

CREATE WAREHOUSE IF NOT EXISTS ARC_COMPUTE_WH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND   = 300
  AUTO_RESUME    = TRUE
  INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 3600;

GRANT OWNERSHIP ON DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT OWNERSHIP ON ALL SCHEMAS IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE ARC_COMPUTE_WH TO ROLE ARC_APP_DEPLOYER;
```

*What this does: creates an empty database with seven sections, plus a small compute resource that switches itself off after 5 minutes of inactivity to control cost. The `STATEMENT_TIMEOUT_IN_SECONDS` line is a deliberate safety limit — please keep it. It stops a runaway query from running for hours and generating unnecessary cost.*

---

### Step 5 — Create the resources that run the application

This creates the infrastructure that hosts the application itself.

```sql
USE ROLE ACCOUNTADMIN;

-- Where the application runs
CREATE COMPUTE POOL IF NOT EXISTS ARC_COMPUTE_POOL
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_S
  AUTO_RESUME = TRUE
  AUTO_SUSPEND_SECS = 3600;

-- Where the application's software package is stored
CREATE IMAGE REPOSITORY IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_IMAGES;

-- Internet addresses the application needs (see Decision 2b)
CREATE NETWORK RULE IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_OSM_RULE
  MODE = EGRESS TYPE = HOST_PORT
  VALUE_LIST = (
    'tile.openstreetmap.org:443',
    'a.tile.openstreetmap.org:443',
    'b.tile.openstreetmap.org:443',
    'c.tile.openstreetmap.org:443',
    'unpkg.com:443',
    'cdnjs.cloudflare.com:443'
  );

CREATE NETWORK RULE IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_FONTS_RULE
  MODE = EGRESS TYPE = HOST_PORT
  VALUE_LIST = ('fonts.googleapis.com:443','fonts.gstatic.com:443');

CREATE EXTERNAL ACCESS INTEGRATION IF NOT EXISTS ARC_EXTERNAL_ACCESS
  ALLOWED_NETWORK_RULES = (
    ARKEMA_PRICING_DB.SPCS.ARC_OSM_RULE,
    ARKEMA_PRICING_DB.SPCS.ARC_FONTS_RULE
  )
  ENABLED = TRUE;
```

*What this does: creates the small server the application runs on (which also switches itself off when idle), a private store for the application software, and a strictly limited allow-list of the only internet addresses the application may reach. It cannot reach anything else.*

**If your security team rejects the internet addresses in Decision 2b:** skip the last three commands in this step and tell us. Everything else still works.

---

### Step 6 — Give our project role permission to use them

```sql
USE ROLE ACCOUNTADMIN;

GRANT USAGE ON DATABASE ARKEMA_PRICING_DB             TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON SCHEMA   ARKEMA_PRICING_DB.SPCS        TO ROLE ARC_APP_DEPLOYER;
GRANT CREATE SERVICE ON SCHEMA ARKEMA_PRICING_DB.SPCS TO ROLE ARC_APP_DEPLOYER;
GRANT CREATE STAGE   ON SCHEMA ARKEMA_PRICING_DB.SPCS TO ROLE ARC_APP_DEPLOYER;

GRANT USAGE, MONITOR, OPERATE ON COMPUTE POOL ARC_COMPUTE_POOL TO ROLE ARC_APP_DEPLOYER;
GRANT READ, WRITE ON IMAGE REPOSITORY ARKEMA_PRICING_DB.SPCS.ARC_IMAGES TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON INTEGRATION ARC_EXTERNAL_ACCESS TO ROLE ARC_APP_DEPLOYER;
```

*What this does: allows our project role to deploy and run the application using only the resources created above. It does not grant access to any other data in your Snowflake account.*

---

### Step 7 — Create the space for the application's AI features

The application includes an AI assistant ("Ask ARC") that answers pricing questions in plain language. It is built on Snowflake's own AI features — Cortex Agents, Cortex Analyst and Cortex Search — so it needs one more small database and a few permissions.

```sql
USE ROLE SYSADMIN;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA   IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

USE ROLE ACCOUNTADMIN;

GRANT USAGE        ON DATABASE SNOWFLAKE_INTELLIGENCE        TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE        ON SCHEMA   SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE ARC_APP_DEPLOYER;
GRANT CREATE AGENT ON SCHEMA   SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE ARC_APP_DEPLOYER;

GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA ARKEMA_PRICING_DB.DOCS TO ROLE ARC_APP_DEPLOYER;
```

*What this does: creates the standard Snowflake location where AI assistants live, and lets us build the application's AI assistant and its document search there. Nothing is created that can reach outside `ARKEMA_PRICING_DB`.*

---

### Step 8 — Check two AI settings

Snowflake normally switches these on for everyone by default. **Some organisations turn them off for security reasons.** We need to know which is the case here, because the AI assistant will not work without them — and we would rather check than ask you for permissions you have already granted.

Please run this and include the result in your reply:

```sql
SHOW GRANTS TO ROLE PUBLIC;
```

In the output, look for these two lines:

| Look for | What it means |
|---|---|
| `USE AI FUNCTIONS` on `ACCOUNT` | Permission to call Snowflake's AI features |
| `SNOWFLAKE.CORTEX_USER` (a database role) | Permission to use Snowflake Cortex |

- **If both are present** — nothing more to do. Skip the rest of this step.
- **If either is missing** — someone has deliberately restricted AI usage in your account. Please run the matching line(s) below, **or** tell us it was restricted on purpose and we will raise it with your security team before going further:

```sql
USE ROLE ACCOUNTADMIN;

-- Only if "USE AI FUNCTIONS" was missing:
GRANT USE AI FUNCTIONS ON ACCOUNT TO ROLE ARC_APP_DEPLOYER;

-- Only if "SNOWFLAKE.CORTEX_USER" was missing:
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE ARC_APP_DEPLOYER;
```

*The same output also tells us about one non-AI permission we need, so this single command covers both checks.*

---

### Step 9 — Send us the results

Please reply with:

1. The output of `SHOW GRANTS TO ROLE PUBLIC;` from Step 8.
2. The output of `SHOW GRANTS TO ROLE ARC_APP_DEPLOYER;` (run this now — it confirms Steps 3–7 worked).
3. Your answer to Step 1 (paid or trial account).
4. Your answers to the Step 2 decisions.

```sql
SHOW GRANTS TO ROLE ARC_APP_DEPLOYER;
```

A screenshot or copy-paste of each is fine.

---

### What this does NOT give us

So the review is straightforward:

- **No access to any other Arkema data.** Only the new, empty `ARKEMA_PRICING_DB`.
- **No administrator rights.** We are not asking for `ACCOUNTADMIN` or `SECURITYADMIN`.
- **No ability to grant permissions to anyone else.**
- **No unrestricted internet access.** Only the eight specific addresses listed in Step 5.
- **Nothing granted to all users.** No permissions are given to Snowflake's `PUBLIC` role.
- **No customer data leaves Snowflake through the AI features.** Cortex Agents, Cortex Analyst and Cortex Search all run inside your Snowflake account. The AI assistant reads only the pricing data in `ARKEMA_PRICING_DB`.

Everything above can be removed later by dropping the `ARC_APP_DEPLOYER` role and the objects created in Steps 4, 5 and 7.

---

### Any questions

Contact **William Lin** (Snowflake Professional Services, App Productionalization workstream) or **Brendan Owens** (Service Delivery Manager). Happy to walk through this on a call if that is easier than working from the document.

---

*End of client-facing section. Everything below is Snowflake PS internal working notes.*

---

## 1. What I verified in the account (read-only, Sep 18)

No objects were created, altered, or dropped. Evidence from `SHOW` commands run as my role:

| Check | Result |
|---|---|
| `SHOW GRANTS TO ROLE "EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` | **0 rows** — the role holds no privileges on any object |
| `CURRENT_AVAILABLE_ROLES()` | `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"`, `PUBLIC` — nothing else |
| `SHOW DATABASES` | Only `SNOWFLAKE` (shared app DB) and my personal database `USER$A6347057@ARKEMA.COM`. **`ARKEMA_PRICING_DB` does not exist.** |
| `SHOW WAREHOUSES` | Only `SYSTEM$STREAMLIT_NOTEBOOK_WH` (owned by `ACCOUNTADMIN`). **No `ARC_COMPUTE_WH`.** |
| `SHOW EXTERNAL ACCESS INTEGRATIONS` | 0 rows visible — **no `ARC_EXTERNAL_ACCESS`** |
| `SHOW ROLES` | `ACCOUNTADMIN`, `SECURITYADMIN`, `SYSADMIN`, `USERADMIN`, `PUBLIC`, plus AAD-provisioned roles. **No `ARC_*` role exists.** |
| `SHOW GRANTS TO USER "A6347057@ARKEMA.COM"` | My role was granted by `AAD_PROVISIONER` — i.e. it is managed by the Entra ID / SCIM pipeline, not by this project |

**Net:** the account is effectively empty from my perspective. Every object the app needs — database, schema, warehouse, compute pool, image repository, external access integration, **Cortex Search services, semantic-model stage, and Cortex Agents** — has to be created, and I cannot create any of them.

### 1a. What the AI layer actually is (corrected after reading the repo)

I checked `arkema-pricing-ss/arc/cortex/` and `arc/ddl/` directly rather than relying on the architecture doc. Three things differ from what I assumed in my first draft:

| Component | What it actually is | Where |
|---|---|---|
| **Semantic model** | **A YAML file on an internal stage — not a native `SEMANTIC VIEW` object.** Referenced as `@ARKEMA_PRICING_DB.MART.SEMANTIC_MODELS/arc_v2_semantic_model.yaml`. The stage is created by `ddl/001_database.sql` line 62. | `cortex/arc_semantic_model.yaml`, `cortex/arc_v2_semantic_model.yaml` |
| **Cortex Search** | Two services over market-note documents: `ARKEMA_PRICING_DB.DOCS.ARC_MARKET_INTEL` (v1) and `ARKEMA_PRICING_DB.DOCS.ARC_V2_MARKET_INTEL` (v2). Both refresh on `TARGET_LAG = '1 day'` using `ARC_COMPUTE_WH`. | `cortex/deploy_cortex.sql`, `cortex/deploy_agent_v2.sql` |
| **Cortex Agents** | `ARC_AGENT` (v1) and `ARC_V2_AGENT` (v2), both in `SNOWFLAKE_INTELLIGENCE.AGENTS`. v1 was created through the Snowsight UI from `arc_agent.json`; v2 has real `CREATE AGENT` DDL. | `cortex/deploy_agent.sql`, `cortex/deploy_agent_v2.sql`, `cortex/arc_agent.json` |

**This correction matters for the grants.** My earlier draft asked for `GRANT SELECT ON SEMANTIC VIEW` — that privilege does not apply here, because there is no semantic view object. What is actually needed is `READ` on the stage holding the YAML. §6 below is corrected.

**Two further things the agent spec reveals**, both of which add grants I had missed:

1. **The agent calls stored procedures as tools** — `ARKEMA_PRICING_DB.MART.SP_NLC2_AGENT_SIMULATE` (`run_simulation`) and `ARKEMA_PRICING_DB.ML.SP_NLC2_AGENT_RECOMMEND` (`run_recommendations`). So `USAGE ON PROCEDURE` is needed in **`MART` and `ML`**, not just `ATOMIC` and `GOV` as I originally wrote.
2. **The app reaches the agent over the REST API**, not SQL — `arc_agent_client.py` line 36 posts to `/api/v2/databases/{db}/schemas/{schema}/agents/{name}:run`. Functionally this still needs `USAGE ON AGENT`, but it means the agent call is an authenticated HTTPS call from inside the container to the Snowflake host, which is worth knowing when debugging.

---

## 2. Role design — please create a dedicated deploy role first

Everything else grants to this role. Do **not** attach these privileges directly to `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"`.

Two reasons, both practical:
- That role is provisioned by `AAD_PROVISIONER`, so it is owned by the Entra ID/SCIM pipeline. Binding the app's identity to an Azure group name is the wrong long-term coupling.
- Objects created during deploy (stage, service, image contents) are owned by the creating role. Ownership should land somewhere intentional and project-scoped.

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS ARC_APP_DEPLOYER
  COMMENT = 'Builds and deploys the ARC Pricing Cockpit SPCS service';

USE ROLE SECURITYADMIN;
-- Reachable from my existing AAD role, so SCIM churn does not break my access
GRANT ROLE ARC_APP_DEPLOYER TO ROLE "EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD";
GRANT ROLE ARC_APP_DEPLOYER TO ROLE SYSADMIN;  -- standard hierarchy hygiene
```

I will deploy with `ARC_APP_DEPLOYER` as the **primary** role and secondary roles disabled, so all created objects and the service's owner's-rights token resolve to it.

---

## 3. Database, schema and warehouse

`ARKEMA_PRICING_DB` does not exist in `BR68104` and has to be created before `arc/ddl` can be run.

> **Correction (Sep 18).** An earlier version of this document, and `20260915_ProjectPlan.md` §2, stated that the repo has no `CREATE DATABASE` for `ARKEMA_PRICING_DB`. **That is wrong** — `ddl/001_database.sql` contains `CREATE DATABASE IF NOT EXISTS ARKEMA_PRICING_DB`, along with all seven schemas, the warehouse, three stages, two file formats, the image repository and five RBAC roles. The audit's "no `CREATE DATABASE`" finding was about **`ARKEMA_PRICING_DB_SCALE`** — the `scale/` tree, which is out of scope as of Sep 16. The DDL *is* reproducible from empty. See `20260918_Architecture.md` §9.
>
> **Practical effect:** the SQL below overlaps with what `ddl/001_database.sql` already does. Both are idempotent (`IF NOT EXISTS`), so running the script afterwards is safe — but if Arkema prefers, they can create only the database and grant me ownership, and I will run `arc/ddl` for the rest.

```sql
USE ROLE SYSADMIN;   -- or ACCOUNTADMIN

CREATE DATABASE IF NOT EXISTS ARKEMA_PRICING_DB;

-- Schemas per arc/ddl and arc/docs/ARCHITECTURE.md
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.RAW;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.ATOMIC;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.MART;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.ML;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.GOV;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.DOCS;
CREATE SCHEMA IF NOT EXISTS ARKEMA_PRICING_DB.SPCS;

-- Warehouse the app and the DDL build both use
CREATE WAREHOUSE IF NOT EXISTS ARC_COMPUTE_WH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND   = 300
  AUTO_RESUME    = TRUE
  INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 3600;   -- deliberate; see note below

-- Hand the database to the project role
GRANT OWNERSHIP ON DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT OWNERSHIP ON ALL SCHEMAS IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE ARC_COMPUTE_WH TO ROLE ARC_APP_DEPLOYER;
```

**The `STATEMENT_TIMEOUT_IN_SECONDS` is intentional, not boilerplate.** Andrew's audit found a BOM fan-out that can multiply scans **up to 144×** on real ZFSC07 data, with no statement timeout and no cancel path — a runaway simulation does not fail, it burns credits for hours. A warehouse-level timeout is the cheapest available guardrail and costs nothing to set now. Please do not drop it.

**Alternative if handing me ownership is not acceptable:** grant `CREATE DATABASE ON ACCOUNT TO ROLE ARC_APP_DEPLOYER` and I will create and own it myself. Either works; ownership of the database is what matters, because `arc/ddl` creates schemas, tables, views, stored procedures, streams, and tasks throughout it.

---

## 4. SPCS infrastructure

Three account-level pieces: a compute pool, an image repository, and an external access integration for the frontend's map tiles and fonts.

```sql
USE ROLE ACCOUNTADMIN;

-- 4a. Compute pool (account-level object)
CREATE COMPUTE POOL IF NOT EXISTS ARC_COMPUTE_POOL
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_S
  AUTO_RESUME = TRUE
  AUTO_SUSPEND_SECS = 3600;

-- 4b. Image repository (schema-level)
CREATE IMAGE REPOSITORY IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_IMAGES;

-- 4c. Egress: OpenStreetMap tiles + Google Fonts, per arc/deploy/network_rules.sql
CREATE NETWORK RULE IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_OSM_RULE
  MODE = EGRESS TYPE = HOST_PORT
  VALUE_LIST = (
    'tile.openstreetmap.org:443',
    'a.tile.openstreetmap.org:443',
    'b.tile.openstreetmap.org:443',
    'c.tile.openstreetmap.org:443',
    'unpkg.com:443',
    'cdnjs.cloudflare.com:443'
  );

CREATE NETWORK RULE IF NOT EXISTS ARKEMA_PRICING_DB.SPCS.ARC_FONTS_RULE
  MODE = EGRESS TYPE = HOST_PORT
  VALUE_LIST = ('fonts.googleapis.com:443','fonts.gstatic.com:443');

CREATE EXTERNAL ACCESS INTEGRATION IF NOT EXISTS ARC_EXTERNAL_ACCESS
  ALLOWED_NETWORK_RULES = (
    ARKEMA_PRICING_DB.SPCS.ARC_OSM_RULE,
    ARKEMA_PRICING_DB.SPCS.ARC_FONTS_RULE
  )
  ENABLED = TRUE;
```

**Please review the egress host list with Arkema InfoSec before creating it.** These are public CDNs the prototype's frontend calls for map tiles and web fonts. If InfoSec will not allow them, tell me — the map and font loading degrade, which is a UI change I would rather surface now than have discovered in UAT.

> Note: the repo's `arc/deploy/spcs_setup.sql` uses `CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION`. I have used `IF NOT EXISTS` above so a re-run cannot silently clobber an integration someone else is depending on.

---

## 5. Grants for the deploy role

This is the core request. Privileges are per the documented `CREATE SERVICE` access-control requirements plus what is needed to push an image and operate the service.

```sql
USE ROLE ACCOUNTADMIN;

-- Database / schema
GRANT USAGE ON DATABASE ARKEMA_PRICING_DB            TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON SCHEMA   ARKEMA_PRICING_DB.SPCS       TO ROLE ARC_APP_DEPLOYER;
GRANT CREATE SERVICE ON SCHEMA ARKEMA_PRICING_DB.SPCS TO ROLE ARC_APP_DEPLOYER;
GRANT CREATE STAGE   ON SCHEMA ARKEMA_PRICING_DB.SPCS TO ROLE ARC_APP_DEPLOYER;

-- Compute pool
GRANT USAGE, MONITOR, OPERATE ON COMPUTE POOL ARC_COMPUTE_POOL TO ROLE ARC_APP_DEPLOYER;

-- Image repository: READ to create the service, WRITE to push the built image
GRANT READ, WRITE ON IMAGE REPOSITORY ARKEMA_PRICING_DB.SPCS.ARC_IMAGES TO ROLE ARC_APP_DEPLOYER;

-- External access integration
GRANT USAGE ON INTEGRATION ARC_EXTERNAL_ACCESS TO ROLE ARC_APP_DEPLOYER;

-- Public endpoint — CHECK BEFORE GRANTING, see note below
-- GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE ARC_APP_DEPLOYER;

-- Warehouse (repeated from §3 for completeness)
GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE ARC_COMPUTE_WH TO ROLE ARC_APP_DEPLOYER;
```

### `BIND SERVICE ENDPOINT` — check, don't assume

This one changed recently and is easy to get wrong in both directions. Per Snowflake behaviour-change bundle **BCR 2321** (rollout began **May 18, 2026**), `BIND SERVICE ENDPOINT ON ACCOUNT` is **automatically granted to the `PUBLIC` role**. Since `PUBLIC` is granted to every user and role, `ARC_APP_DEPLOYER` most likely already has it and **no grant is needed**.

However, the same note states administrators who want to limit the feature may revoke it from `PUBLIC`. Arkema InfoSec may well have done exactly that. So:

```sql
-- Check whether PUBLIC still holds it
SHOW GRANTS TO ROLE PUBLIC;   -- look for BIND SERVICE ENDPOINT on ACCOUNT

-- Only if it has been revoked from PUBLIC:
GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE ARC_APP_DEPLOYER;
```

**Why it matters either way:** `service_spec.yaml` sets `public: true`, so the service needs this privilege to expose its endpoint. And if the **owning role later loses** it, the public endpoint stops being accessible — so this is a privilege to keep, not one to grant temporarily during deployment.

I could not fully determine from my read-only session whether `PUBLIC` still holds it in `BR68104`. **Please check as part of actioning this request** rather than granting blind.

### Why each one

| Privilege | Object | Purpose |
|---|---|---|
| `CREATE SERVICE` | schema | Create the SPCS service — documented `CREATE SERVICE` requirement |
| `USAGE` | compute pool | Run the service on the pool — documented `CREATE SERVICE` requirement |
| `READ` | image repository | Read the image referenced by the service spec — documented `CREATE SERVICE` requirement |
| `WRITE` | image repository | **Push** the locally built image via `docker push`. Not part of `CREATE SERVICE`, but required by the build step in `arc/deploy/deploy.sh` |
| `BIND SERVICE ENDPOINT` | account | Required to create a service exposing a **public** endpoint (`service_spec.yaml` sets `public: true`). **Likely already held via `PUBLIC` since BCR 2321 — check first, see the note above.** |
| `USAGE` | external access integration | Attach `ARC_EXTERNAL_ACCESS` so the frontend can reach map tiles and fonts |
| `CREATE STAGE` | schema | Internal stage to hold the service specification file |
| `READ` | stage | Read the spec at service-create time. Implicit here since the deploy role owns the stage it creates; only needed explicitly if someone else's stage is used |
| `MONITOR`, `OPERATE` | compute pool | Describe the pool, and suspend/resume it during deployment and troubleshooting |
| `USAGE`, `OPERATE`, `MONITOR` | warehouse | Run the DDL build and the app's queries; resume/suspend as needed |

---

## 6. Data and AI access for the running service

The service runs with owner's rights, as the role that created it (`ARC_APP_DEPLOYER`). If §3 grants ownership of `ARKEMA_PRICING_DB` to that role, most of this is already covered by ownership — but state it explicitly so nothing is assumed:

```sql
GRANT USAGE  ON ALL SCHEMAS    IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE  ON FUTURE SCHEMAS IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT SELECT ON ALL TABLES     IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT SELECT ON FUTURE TABLES  IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT SELECT ON ALL VIEWS      IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;
GRANT SELECT ON FUTURE VIEWS   IN DATABASE ARKEMA_PRICING_DB TO ROLE ARC_APP_DEPLOYER;

-- Write paths: scenario save, overrides, governance actions
GRANT INSERT, UPDATE, DELETE ON ALL TABLES    IN SCHEMA ARKEMA_PRICING_DB.GOV TO ROLE ARC_APP_DEPLOYER;
GRANT INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA ARKEMA_PRICING_DB.GOV TO ROLE ARC_APP_DEPLOYER;

-- Simulation / recommendation stored procedures.
-- ATOMIC + GOV: SP_SIMULATE_MOVC, governance writes, etc.
-- MART + ML: the two procedures the Cortex Agent calls as tools (see §1a)
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.ATOMIC TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.ATOMIC TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.GOV    TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.GOV    TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.MART   TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.MART   TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.ML     TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.ML     TO ROLE ARC_APP_DEPLOYER;
```

---

## 6a. AI layer: Cortex Agent, Cortex Search, semantic model (F8 GenAI — customer-confirmed "Must")

Read §1a first — the AI layer is not shaped the way the architecture doc implies. Four separate things need grants.

### (a) Account-level Cortex access — check, don't assume

Calling Cortex functions needs **both** an account-level privilege **and** a database role:
- `USE AI FUNCTIONS` on the account, **and**
- the `SNOWFLAKE.CORTEX_USER` database role (or `SNOWFLAKE.AI_FUNCTIONS_USER`).

**Both are granted to `PUBLIC` by default**, so `ARC_APP_DEPLOYER` most likely already has them and **no grant is needed**. But an administrator can revoke either from `PUBLIC`, and in a security-conscious account that is a realistic possibility. Same pattern as `BIND SERVICE ENDPOINT` in §5 — **check first**:

```sql
SHOW GRANTS TO ROLE PUBLIC;   -- look for: USE AI FUNCTIONS on ACCOUNT,
                              -- and DATABASE ROLE SNOWFLAKE.CORTEX_USER

-- ONLY if either has been revoked from PUBLIC (run as ACCOUNTADMIN):
GRANT USE AI FUNCTIONS ON ACCOUNT TO ROLE ARC_APP_DEPLOYER;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE ARC_APP_DEPLOYER;
```

> `SNOWFLAKE.CORTEX_USER` **cannot be granted directly to a user** — it must go to an account role, which is what we are doing.

**This is not optional plumbing.** Creating a Cortex Search service itself requires `CORTEX_USER` (or `CORTEX_EMBED_USER`), because the service calls the embedding functions to build its index. If this is missing, `CREATE CORTEX SEARCH SERVICE` fails outright — it is not a runtime-only concern.

### (b) The semantic model — a stage, not a semantic view

`ddl/001_database.sql` creates the stage; the YAML is uploaded to it by the deploy script.

```sql
-- Owned by ARC_APP_DEPLOYER if §3 grants database ownership; stated explicitly regardless
GRANT READ  ON STAGE ARKEMA_PRICING_DB.MART.SEMANTIC_MODELS TO ROLE ARC_APP_DEPLOYER;
GRANT WRITE ON STAGE ARKEMA_PRICING_DB.MART.SEMANTIC_MODELS TO ROLE ARC_APP_DEPLOYER;  -- to upload the YAML
```

`WRITE` is needed because the deploy step pushes `arc_v2_semantic_model.yaml` to the stage (`snow stage copy`). `READ` is what Cortex Analyst uses at query time.

### (c) Cortex Search services

Two services over the market-note documents in `DOCS`. Creating them needs `CREATE CORTEX SEARCH SERVICE` on the schema, `SELECT` on the source table, and `USAGE` on the refresh warehouse.

```sql
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA ARKEMA_PRICING_DB.DOCS TO ROLE ARC_APP_DEPLOYER;

-- Source table the services index (already covered by the SELECT grants above,
-- repeated here so the dependency is obvious)
GRANT SELECT ON TABLE ARKEMA_PRICING_DB.DOCS.MARKET_NOTES TO ROLE ARC_APP_DEPLOYER;

-- Refresh warehouse — already granted in §3
-- GRANT USAGE ON WAREHOUSE ARC_COMPUTE_WH TO ROLE ARC_APP_DEPLOYER;

-- After creation, for any role that needs to query them (including the agent's caller)
GRANT USAGE ON CORTEX SEARCH SERVICE ARKEMA_PRICING_DB.DOCS.ARC_MARKET_INTEL    TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON CORTEX SEARCH SERVICE ARKEMA_PRICING_DB.DOCS.ARC_V2_MARKET_INTEL TO ROLE ARC_APP_DEPLOYER;
```

**Cost note worth flagging now:** both services are defined with `TARGET_LAG = '1 day'`, so they will refresh daily against `ARC_COMPUTE_WH` whether or not anyone uses the app. Small for the current document volume, but it is a standing cost that starts the moment they are created.

### (d) The Cortex Agents

`SNOWFLAKE_INTELLIGENCE` **does not exist in `BR68104`** (verified Sep 18). It has to be created, along with its `AGENTS` schema, before either agent can be deployed.

```sql
USE ROLE SYSADMIN;   -- or ACCOUNTADMIN
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA   IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

USE ROLE ACCOUNTADMIN;
GRANT USAGE        ON DATABASE SNOWFLAKE_INTELLIGENCE        TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE        ON SCHEMA   SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE ARC_APP_DEPLOYER;
GRANT CREATE AGENT ON SCHEMA   SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE ARC_APP_DEPLOYER;

-- After the agents are created, for the app (and later the UAT role) to invoke them
GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_AGENT    TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_V2_AGENT TO ROLE ARC_APP_DEPLOYER;
```

`CREATE AGENT` additionally requires that the creating role hold `USAGE` on the Cortex Search services referenced in the agent spec, and `USAGE` on the database, schema and tables behind the semantic model — all covered by (b), (c) and §6 above.

> **Which agent is live matters.** `service_spec.yaml` sets `ARC_AGENT_NAME: ARC_AGENT` (v1), but v1 was created by hand in the Snowsight UI and has no reproducible DDL, while `ARC_V2_AGENT` does (`cortex/deploy_agent_v2.sql`) and is the one aligned with the NLC2 pilot. **This needs deciding in Phase 1** — I would rather deploy the version that can be rebuilt from the repo. Flagging it as an open item, not assuming it.

### (e) Do not carry over the POC's `PUBLIC` grants

`cortex/deploy_agent_v2.sql` ends with:

```sql
GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_V2_AGENT TO ROLE PUBLIC;
```

**Do not run that line in the Arkema account.** `PUBLIC` is granted to every user, so this would let anyone in the account query the pricing agent — and through its tools, run simulations and the ML recommendation engine against real margin data. It is the same authorization defect class as the audit's #1 blocker. Grant the agent to a named role instead.

`arc_agent.json` / `deploy_agent.sql` should be checked for the same pattern before v1 is deployed.

---

## 7. Post-deploy: sharing the app for UAT

Needed once the service is running, for Michael's pre-UAT early access and for the UAT users. Grant to roles, not to individuals.

**Important finding, and it needs a code change rather than a grant.** In SPCS, access to a service's endpoint is granted through a **service role**, and service roles are **defined in the service specification** — they are not created automatically. The current `arc/deploy/service_spec.yaml` declares an endpoint but **has no `serviceRoles:` block at all**, so there is presently no service role to grant. Adding one is a small spec change I will make under M3-01; recording it here so the sequencing is clear.

Once the spec declares a service role, the grant takes this form — note the `service-name!service-role-name` syntax:

```sql
-- Run as the service OWNER (only the owner can grant a service role)
USE ROLE ARC_APP_DEPLOYER;

-- Confirm the actual service role name first
SHOW ROLES IN SERVICE ARKEMA_PRICING_DB.SPCS.ARC_SERVICE;

-- Then grant it (substitute the real names from the output above)
GRANT SERVICE ROLE ARC_SERVICE!<service_role_name> TO ROLE <ARC_UAT_ROLE>;

-- UAT role also needs to resolve the service's container objects
GRANT USAGE ON DATABASE ARKEMA_PRICING_DB      TO ROLE <ARC_UAT_ROLE>;
GRANT USAGE ON SCHEMA   ARKEMA_PRICING_DB.SPCS TO ROLE <ARC_UAT_ROLE>;

-- Optional: let a support role view logs / operate the service
GRANT MONITOR ON SERVICE ARKEMA_PRICING_DB.SPCS.ARC_SERVICE TO ROLE <ARC_SUPPORT_ROLE>;
GRANT OPERATE ON SERVICE ARKEMA_PRICING_DB.SPCS.ARC_SERVICE TO ROLE <ARC_SUPPORT_ROLE>;
```

**§7 is deliberately provisional and is not on the critical path.** Three things have to settle before it can be written exactly: the service spec needs a `serviceRoles:` block, the service has to exist so `SHOW ROLES IN SERVICE` can confirm the real names, and the UAT role name needs deciding with Wole, who owns the RBAC/SSO workstream. I would rather confirm this against a running service than commit to syntax for objects that do not yet exist. **Please do not let §7 hold up §3–§6, which are what I need now.**

---

## 8. What is NOT needed (so nobody over-grants)

Worth stating, because a request this size invites scope creep in the grant review:

- **No `ACCOUNTADMIN` or `SECURITYADMIN` for me.** Everything above is a scoped grant to a project role.
- **No `CREATE DATABASE ON ACCOUNT`** if you use the §3 Option A path (admin creates, grants me ownership).
- **No `MANAGE GRANTS`.** I do not need to grant privileges to others. The one exception is `GRANT SERVICE ROLE` in §7, which requires **`OWNERSHIP` of the service** — and I will have that automatically as the role that created it, so it still needs no extra privilege.
- **No caller's-rights / `GRANT CALLER` setup.** That is a SAR concept and is not in play for SPCS.
- **No `IMPORTED PRIVILEGES` on `SNOWFLAKE`** unless we later need `ACCOUNT_USAGE` for cost or access auditing — not required for deployment.
- **Nothing granted to `PUBLIC`**, anywhere.

---

## 9. How to verify the grants landed (I will run these)

```sql
USE ROLE ARC_APP_DEPLOYER;
SHOW GRANTS TO ROLE ARC_APP_DEPLOYER;
SHOW DATABASES LIKE 'ARKEMA_PRICING_DB';
SHOW WAREHOUSES LIKE 'ARC_COMPUTE_WH';
SHOW COMPUTE POOLS LIKE 'ARC_COMPUTE_POOL';
SHOW IMAGE REPOSITORIES IN SCHEMA ARKEMA_PRICING_DB.SPCS;
SHOW EXTERNAL ACCESS INTEGRATIONS LIKE 'ARC_EXTERNAL_ACCESS';
```

Then the real smoke test: build the DB from `arc/ddl` on empty, `docker push` to `ARC_IMAGES`, and `CREATE SERVICE`. I will report back on what breaks.

---

## 10. Concerns I want on the record

Ordered by how much they can cost.

1. **I am 100% blocked.** My role has zero privileges. I cannot create a database, a schema, or a warehouse, and `ARKEMA_PRICING_DB` does not exist. **Every item in the App workstream plan — Phase 1 discovery included — is waiting on this document being actioned.** Please name the `ACCOUNTADMIN` who owns it and give me a turnaround date; my critical path now runs entirely through them.
2. **`arc/deploy/spcs_setup.sql` must not be run as-is.** It runs `USE DATABASE ARKEMA_PRICING_DB` before that database exists, grants everything to `SYSADMIN` rather than a project role, uses `CREATE OR REPLACE` on the external access integration, and `GRANT SELECT ON ALL TABLES` on schemas whose tables do not exist yet so the grants would silently apply to nothing. §3–§6 above supersede it. I will fix the script under M3-01.
3. **The BOM fan-out has no guardrail today.** Up to 144× scan multiplication on real ZFSC07 data, no statement timeout, no cancel path. The `STATEMENT_TIMEOUT_IN_SECONDS = 3600` in §3 is the cheap mitigation. It is worth setting before any real data lands, not after the first runaway bill.
4. **Egress hosts need an InfoSec decision, not just a grant.** §4c opens eight public CDN hosts for map tiles and fonts. If Arkema InfoSec refuses, the map and font behaviour changes and that becomes a UI finding for UAT. Better to know in September.
5. **The POC's `PUBLIC` grants must not be carried over.** `cortex/deploy_agent_v2.sql` ends with `GRANT USAGE ON AGENT ... ARC_V2_AGENT TO ROLE PUBLIC`. In the Arkema account that would let **any user** query the pricing agent and, through its `run_simulation` and `run_recommendations` tools, execute simulations and the ML pricing engine against real margin data. Same defect class as the audit's #1 blocker. Getting this scoped at grant time is far cheaper than retrofitting during UAT. See §6a(e).
6. **`ARC_AGENT` (v1) has no reproducible DDL.** It was created by hand in the Snowsight UI from `arc_agent.json`, and `service_spec.yaml` points the app at it. `ARC_V2_AGENT` does have DDL and is the NLC2-pilot-aligned version. **Which one we deploy needs deciding in Phase 1** — if it is v1, we have an unreproducible object in the critical path, which is the same class of problem as the audit's finding #6 about the deployment not being rebuildable from the repo.
7. **The service spec has no `serviceRoles:` block**, so as written there is no way to grant anyone endpoint access — Michael's early access and UAT sharing both depend on a spec change, not a grant. Small fix, but it has to happen before §7 is usable, and it is the kind of thing that surfaces on the morning of a UAT kickoff if nobody looks for it now.
8. **Two Cortex Search services will start costing money the day they are created.** Both are defined with `TARGET_LAG = '1 day'` against `ARC_COMPUTE_WH`, so they refresh daily whether or not anyone opens the app. Small at current document volumes, but it is a standing cost and worth a conscious decision rather than a surprise on the first bill.
9. **AI access may be deliberately restricted in this account.** `USE AI FUNCTIONS` and `SNOWFLAKE.CORTEX_USER` are granted to `PUBLIC` by default, but a security-conscious organisation may have revoked them. If Arkema has, **F8 GenAI — which Michael confirmed in writing as a "Must" — cannot work without a policy decision from their security team.** That is a scope conversation, not a grant, and I would rather discover it in September. Step 8 of §0 is designed to surface it.
10. **Paid vs trial account is still unconfirmed.** This does not block SPCS — but it was the blocker for SAR, and it likely affects other platform features, so it is worth answering while we have an admin's attention.
11. **§7 is deliberately provisional.** I would rather confirm service-role syntax against a real deployed service than guess at it. It is not on the critical path for §3–§6.

---

*Read-only verification only. No Snowflake objects were created, altered, or dropped in producing this document.*
