Wole Babalola
to Brendan, Chase, Sharanya, Luis, me, Daniel, Rishabh

Hi Brendan,

Quick update from today’s Arkema Snowflake platform discussion.

We reviewed the account strategy using the key decision areas: security/compliance, governance, access and administration, operating model, data collaboration, environment management, and cost/operations. Based on the requirements discussed, a single Snowflake account currently appears suitable, with DEV, UAT, and PROD logically separated within the account. This remains subject to confirmation from the Security/InfoSec team, who could not attend today’s session. Michael will follow up on that and let us know by Thursday.

We also discussed data movement and provisioning between logical environments.  Regardless of whether the final architecture uses one or multiple Snowflake accounts, we agreed that we can begin establishing the platform foundation now using the account already provisioned. This includes the database/medallion structure, warehouses, RBAC/access roles, functional roles/service identities, and the initial onboarding structure for the technical/ML teams.

For ingestion, the source files were confirmed as CSV.  The team is currently configuring the AWS-side S3 bucket and will provide a timeline for when the bucket is ready for Snowflake access. Once available, I will need the applicable S3 bucket/path and IAM role information to create the Snowflake storage integration, followed by the stage and CSV file-format configuration. We can initially bring in enough data to unblock development while the fully automated ingestion pipeline is built in parallel.

I also committed to providing a high-level Snowflake architecture diagram showing the proposed medallion architecture and environment structure. This should help the team visualize the recommended platform design, particularly since Snowflake is relatively new to them.

Immediate next steps:
- Michael needs to confirm the Security/InfoSec position on account isolation by Thursday and provide the S3 readiness timeline/details; in parallel, I will begin defining the DEV platform foundation, provide the architecture diagram, and prepare the initial ingestion setup so we can bring data into the environment and unblock the development team while the broader ingestion pipeline is being established.

One additional observation from the discussion is that some level of Snowflake enablement/knowledge transfer will likely be beneficial as the project progresses, particularly as the team begins using Snowflake capabilities and establishing its longer-term operating model.

Please let me know if you have questions or clarifications.

Warm Regards,