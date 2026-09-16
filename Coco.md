# CoCo

cortex -c snowhouse --workdir "/Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing"

**[ /bypass ]**
**[ /auto-accept-plan ]**

$sd-project-list-setup --force
$sd-display-project-menu

what is my session tag?

$sd-submit-info 2
    Arkema

Help me tag session to: "INTERNAL / DEFINED_INITIATIVE / Strategic_Initiatives"
    $sd-submit-info 27

# ASKs

- Just update local file. Do not update database objects in Snowflake account.
- Always keep codes and reports updated, clean, neat, and efficient.
- Do NOT return Hallucinations or OUTDATED info.
- Always let me know if you have any concerns.
- In file "/Users/wilin/Docs/LocalProjects/GitHub/cowork_gaps/Coco.md"
    - please help proceed with section "# Ask_001"

# 20260909

Please give me detail summary from meeting transcript below. Especailly anything related to me (William Lin) and APP (my work in this project)
  - /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing/ref/20260909_PrepMeeting_Transcript.txt
  - Create the summary file use path below:
    - /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing/ref/20260909_PrepMeeting_Summary.md
```
Your role (William Lin):

• You are responsible for deploying the app to production — that's your primary work stream.
• You confirmed directly with Brendan: the app is taken as-is, no new features, no edits.
• You start next week alongside Sharanya and Luis at ~10 hours ramp.
• A team walkthrough of the app with Rashab is scheduled for September 14.

APP (Pricing Cockpit):

• Built by Rashab (Industry team) — no KT provided, self-serve from the Git repo.
• Rashab stops editing on September 21, then hands off to your team for production deployment.
• No feature requests will be accepted after that point — Brendan will enforce this.
• Biggest current blocker: environment access is not yet provisioned. Until that's resolved, contact Michael (customer) directly or wait for Brendan's
follow-up.
```

# 20260915

Please help start research on this new client project.
Generate a detailed Project Plan with path below:
  - /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing/20260915_ProjectPlan.md
Reference:
  - Current App source code:
    - /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing-ss/
  - Findings from Andrew on this project:
    - /Users/wilin/Docs/Snowflake/20260904 Arkema/Andrew/
  - Few resources I have for now:
    - /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing/ref/
Note:
  - I feel we should push the app to SAR (Snowflake App Runtime).
  - Current App Review meeting with Rishabh Raman scheduled tomorrow morning.
Just update local file. Do not update database objects in Snowflake account.
Always keep codes and reports updated, clean, neat, and efficient.
Do NOT return Hallucinations or OUTDATED info.
Always let me know if you have any concerns.

# 20260916

File: /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing/20260915_ProjectPlan.md
Please add Glossary section
  - BOM ?
    - ZFSC07 ?
  - etc.
What does M3-02 reference to?
  - "Given M3-02 (entitlement enforcement) is budgeted at 0 hours but is the top blocker in the audit — how should this be resourced? Is a change order expected here?"
Where can I locate these numbers in the project?
  - "Does Arkema (the customer) know the specific pricing/margin numbers currently shown by the app are not reliable (audit Pass 3)? Is "deploy as-is" intended to cover shipping those numbers unchanged, or is there tolerance to fix the clearly-broken arithmetic (e.g., the beta-clamp bug that disables the ML model's influence entirely) within "productionalization" rather than "new feature"?"
Keeping doc simple to understand would be great, since I am still new to this project.

# Todo

- /Users/wilin/Docs/LocalProjects/GitHub/arkema-pricing/ref/20260915_MeetingNoteFromWole.md

# END