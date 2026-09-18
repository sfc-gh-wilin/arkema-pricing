# Arkema Snowflake — Grants Needed (App Productionalization, SPCS)

**Date:** September 18, 2026
**Requestor:** William Lin (Snowflake PS — App Productionalization workstream)
**Account:** `BR68104` · region `AWS_EU_WEST_1`
**My user:** `A6347057@ARKEMA.COM`
**My only role:** `"EUNFG-AZURE-APP-ACCESS-SNOWFLAKE-ADMIN-PROD"` (AAD/SCIM-provisioned; note the hyphens — **must be double-quoted** in all SQL)
**Deployment target:** **Snowpark Container Services (SPCS)** — per the signed SOW (M3-01 "SPCS Setup & Deploy") and confirmed Sep 18. *SAR was evaluated and ruled out: it runs Node.js only, and the ARC backend is Python. See `others/20260915_ProjectPlan_SAR.md`.*

> **Ask in one line:** the role above currently has **zero object privileges**. Nothing can be built or deployed until an `ACCOUNTADMIN` actions §3–§6.

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

**Net:** the account is effectively empty from my perspective. Every object the app needs — database, schema, warehouse, compute pool, image repository, external access integration — has to be created, and I cannot create any of them.

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

`ARKEMA_PRICING_DB` does not exist. It has to be created before `arc/ddl` can be run, and **the repo has no `CREATE DATABASE` statement for it** — a known gap I will fix under M3-01.

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

-- Simulation / recommendation stored procedures (SP_SIMULATE_MOVC, SP_BUILD_RECOMMENDATIONS, ...)
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.ATOMIC TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.ATOMIC TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON ALL PROCEDURES    IN SCHEMA ARKEMA_PRICING_DB.GOV    TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA ARKEMA_PRICING_DB.GOV    TO ROLE ARC_APP_DEPLOYER;
```

### Cortex Agent (F8 GenAI — customer-confirmed "Must")

`service_spec.yaml` points the app at `SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_AGENT`. **Verified Sep 18: `SNOWFLAKE_INTELLIGENCE` does not exist in `BR68104`** — the agent and its semantic view have to be created here, not just granted.

```sql
-- After the agent and semantic view are created:
GRANT USAGE ON DATABASE SNOWFLAKE_INTELLIGENCE        TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON SCHEMA   SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE ARC_APP_DEPLOYER;
GRANT USAGE ON AGENT    SNOWFLAKE_INTELLIGENCE.AGENTS.ARC_AGENT TO ROLE ARC_APP_DEPLOYER;
GRANT SELECT ON SEMANTIC VIEW ARKEMA_PRICING_DB.MART.<SEMANTIC_VIEW> TO ROLE ARC_APP_DEPLOYER;
```

> **Do not reproduce the POC's `GRANT ... TO ROLE PUBLIC` on the agent.** In the prototype, `ARC_AGENT` was granted to `PUBLIC`. That is the same authorization defect class as the audit's #1 blocker and must not be carried into this account.

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
4. **Egress hosts need an InfoSec decision, not just a grant.** §4c opens six public CDN hosts for map tiles and fonts. If Arkema InfoSec refuses, the map and font behaviour changes and that becomes a UI finding for UAT. Better to know in September.
5. **The POC's `PUBLIC` grants must not be carried over.** `ARC_AGENT` was granted to `ROLE PUBLIC` in the prototype. Getting this scoped correctly at grant time is far cheaper than retrofitting it during UAT.
6. **The service spec has no `serviceRoles:` block**, so as written there is no way to grant anyone endpoint access — Michael's early access and UAT sharing both depend on a spec change, not a grant. Small fix, but it has to happen before §7 is usable, and it is the kind of thing that surfaces on the morning of a UAT kickoff if nobody looks for it now.
7. **Paid vs trial account is still unconfirmed.** This does not block SPCS — but it was the blocker for SAR, and it likely affects other platform features, so it is worth answering while we have an admin's attention.
8. **§7 is deliberately provisional.** I would rather confirm service-role syntax against a real deployed service than guess at it. It is not on the critical path for §3–§6.

---

*Read-only verification only. No Snowflake objects were created, altered, or dropped in producing this document.*
