---
name: crm-internal-tool
description: The HoorayHR LinkedIn Outreach CRM project + its hard constraints
metadata: 
  node_type: memory
  type: project
  originSessionId: 7dcc0289-8d59-41ec-af7f-e26a2e3b9b91
---

LinkedIn Outreach CRM for HoorayHR: single-file `index.html` web app + Chrome extension + one n8n webhook fronting a Google Sheet, plus Pipedrive + OpenAI. Captures recent job-changers from LinkedIn → outreach → Pipedrive. Full context lives in the repo's `CLAUDE.md` (read it first).

**Hard constraints Raphael set (do not violate):**
- One n8n webhook only; new capability = new `action` value routed by the Switch (never a second webhook).
- No custom-code nodes in n8n — built-in nodes + expressions only.
- Self-contained `index.html` (no build/framework); every UI string trilingual (EN/DE/NL).
- n8n API key stays in sessionStorage + the localStorage "bridge" to the extension; no other secrets in the page.

See [[user-raphael]].
