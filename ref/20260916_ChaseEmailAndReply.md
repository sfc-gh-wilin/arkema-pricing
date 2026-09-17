# Chase Growney

to THAI, Brendan, Luis, Rishabh, Wole, me

Hi Michael,

Following up on a few items as the services team gets ramped up on deployment.

While reviewing the current PRD and the Git repo, we discussed that
the machine learning component is no longer included in the MVP spec
based on the PRD. I wanted to confirm whether ML was intentionally
removed from the MVP scope, or if that was an oversight. Rishabh built
the pricing cockpit to align with the updated PRD, removing the ML
component. We need to confirm this because the SOW was scoped with ML
in mind, and it also affects Luis's area of focus on the AI/ML side.

Separately, the what-if simulation engine that ran using Snowflake
Intelligence (Cortex agents) during the POC also appears to have been
removed from the current PRD as well. That component simplified the
user experience quite a bit by letting analysts interact
conversationally rather than navigating through multiple UI layers.
Should we keep that component in scope for the MVP as well?

On the deployment side, things are moving forward. The services team
is getting oriented on the repo and data ingestion, and Brendan will
follow up shortly with an updated timeline, including when you can get
early access to a test environment ahead of formal UAT.

Let me know your thoughts on the ML and what-if engine pieces when you
get a chance.

Best,
Chase

## THAI Michael
to Chase, brendan.owens, Luis, Rishabh, Wole, me, TASSAERT, LAO

Hi Chase
Thanks for calling these out.

1. ML:
ML features are covered in F6 - Margin Protection recommendation.
More specifically we were looking to prioritize recommendation by volume or margin consideration (F6.2/F6.3) as well as taking into consideration ABC classifcation (F6.7 bargaining power) and even in a future state past negotiations outcomes captured in the Price management module (F11).
When it comes to the MVP scope, BU recently mentioned they would already be happy with F6.1, while F6.2, F6.3 could be achieved manually be filtering or sorting elements in the UI, which slightly decrease the priority for this feature.
Bottom line, if we can easily embed the ML features Rishabh had already implemented in the POC, let's keep them for now. If it adds significant amount of complexity and effort and could compromise the delivery of the MVP, I am open for dicussion.

2. Cortex agent and Cowork must be included as they are key features expected by the Business. They are covered in F8 - GenAI Interface. Please let me know if you need further clairification.

**F8: GenAI Interface**
Description: Natural language query and agentic analysis capabilities for intuitive exploration and strategic recommendations. Queries can be scoped to the user's full data access, a specific cost simulation run (F4.12), or a specific price increase scenario (F4.15).
Functional Requirements:

ID      Requirement
        Priority
F8.1    NLQ interface for ad-hoc queries ("what's our margin exposure in Asia if MMA hits EUR 2,000/t?")
        Must
F8.2    Text-to-SQL translation for flexible analysis
        Must
F8.3    Agentic analysis to identify patterns and anomalies
        Must
F8.4    AI-generated pricing strategy recommendations
        Must
F8.5    Contextual explanations of margin drivers
        Must
F8.6    Generate commercial justification text per price increase recommendation, e.g. "FP price to increase by X% — key raw contributors [Raw A, Raw B] account for X% of product variable cost; current market trend: +Y% over 3 months"
        Must
F8.7    Context selection: by default the GenAI interface queries against all data the authenticated user has access to; user can optionally narrow context to one or more specific cost simulation runs (F4.12) or to one or more named price increase scenarios (F4.15); selected context is displayed persistently in the query panel; all AI responses and generated SQL are scoped to the selected context
        Must

Best regards
Michael THAI
Global Head of Data & Architecture
Head of Information Systems
Asia Pacific outside China
iTeam / Information Technology Services
P +81-(0)3-5251-9498
M +81-(0)80-1298-8830

