# CLAUDE.md — LinkedIn Outreach CRM (HoorayHR)

> Context & handover doc for any future LLM/dev. Read this fully before changing anything.
> Companion doc for the human operator: [SETUP.md](SETUP.md) (Deutsch, einrichtungsfokussiert).

---

## 0. TL;DR

A tiny, self-contained **internal sales CRM for HoorayHR** that turns "people who just
changed jobs on LinkedIn" into outreach leads and pushes the good ones into Pipedrive.

Three parts, **one shared backend**:
1. **`index.html`** — the whole CRM web app in a single file (no build, no framework). Served at `https://crm.internal-tool.online`.
2. **`chrome-extension.zip`** — a Chrome MV3 extension that scrapes leads from LinkedIn / Sales Navigator and checks Pipedrive.
3. **n8n workflow** (`n8n-crm-webhook-updated.json`) fronting a **Google Sheet** — one webhook, branched by an `action` field.

The user (Raphael, CRO at HoorayHR, German-speaking) drives this. **Primary goal: it must be
dead simple for a non-technical salesperson to follow.** When in doubt, optimise for that.

---

## 1. Intent & product vision (what it is and should be)

**The job-to-be-done:** A salesperson finds people who recently switched jobs (great moment to
pitch HR software), captures them, sends a warm personal LinkedIn message, tracks the reply, and
when it's promising, creates a deal in Pipedrive — without double-entry and without leaving a
familiar, simple UI.

**The loop the product supports:**
1. **Grow** — On Sales Navigator (a saved search) or a normal LinkedIn profile, the Chrome
   extension scrapes the person and POSTs them to n8n → Google Sheet. Re-scraping is safe:
   unchanged people stay, a detected company change resets them to `new`.
2. **Message** — In the CRM, open a lead, pick a template (placeholders auto-filled) or generate
   a draft with AI, copy it into LinkedIn, mark it sent. The sent text is logged on the lead.
3. **Qualify & hand off** — Move the lead through stages (list dropdown or Kanban drag). When it's
   a real opportunity, push it to Pipedrive (org + person + deal + note) — after a duplicate check.

**Design principles the user cares about (honour these):**
- **Simplicity for salespeople** over feature richness. Few clicks, clear words, no jargon.
- **Trilingual**: English, Deutsch, Nederlands. *Every* user-facing string lives in the `I18N`
  dict (CRM) / `EXT_I18N` (extension) in **all three languages**. Never hardcode UI text.
- **One n8n webhook, many `action`s.** Do NOT spin up extra webhooks. New capability = new
  `action` value routed by the Switch node. (The user was explicit about this.)
- **No custom-code nodes in n8n.** Use built-in nodes (Set, Filter, Aggregate, SplitOut, IF,
  Switch, Merge, HTTP Request, Google Sheets, Pipedrive, OpenAI) and **expressions**. The user
  asked explicitly to avoid Code nodes. (Earlier drafts used Code nodes; they were replaced.)
- **No backend secrets in the page.** The n8n API key lives only in `sessionStorage`, mirrored to
  the extension via a localStorage "bridge". Everything else lives in Google Sheets / n8n.
- **Self-contained `index.html`.** No bundler, no npm, no external JS/CSS. Plain vanilla JS + inline CSS.

---

## 2. Architecture & data flow

```
 Chrome Extension (LinkedIn tabs)          CRM web app  (crm.internal-tool.online / index.html)
 ┌───────────────────────────┐             ┌─────────────────────────────────────────────┐
 │ popup.js  scrape + check   │             │ gate (enter n8n key) → leads (list|board) /  │
 │ background.js  badges/alarm│             │ messages (templates+analytics) / help        │
 │ bridge.js  reads key on CRM│             └───────────────┬───────────────────────────────┘
 └─────────────┬──────────────┘                             │
   action: import / pipedrive-check         action: list / sync / pipedrive-add /
   X-API-Key + apiKey in body               pipedrive-check / ai-message / list-templates /
               │                             save-templates    (X-API-Key + apiKey in body)
               ▼                                             ▼
   ┌──────────────────────────────────────────────────────────────────────────────┐
   │  ONE n8n webhook  56bc510a-38d4-410f-ab5c-41ca63e825f8                          │
   │  Switch "Route by action3" → one branch per action (+ CORS preflight fallback) │
   └───────┬──────────────────────────┬───────────────────────────┬─────────────────┘
           ▼                          ▼                           ▼
     Google Sheet              Pipedrive API                 OpenAI (chat)
     (Leads + Templates tabs)  (org/person/deal/note +       gpt-5.4-mini
                                search for dedupe)
```

**The bridge (single-sign-on between CRM and extension):**
- On unlock the CRM writes `li_crm_bridge_key` + `li_crm_bridge_env` to its own `localStorage`.
  Once the user picks their identity it also writes `li_crm_bridge_user` (the chosen SDR id).
- The extension content script `bridge.js` runs on the CRM page, polls that localStorage every 2s,
  and mirrors it into `chrome.storage.local` (`n8nKeyFromCRM`, `n8nEnvFromCRM`, `n8nUserFromCRM`).
- So the user types the key once (in the CRM) and picks their name once; the extension reuses both.
  The extension stamps that user onto every scraped lead's `sdr` array. Locking the CRM clears all three.

---

## 3. Files in this repo

| File | Role |
|---|---|
| `index.html` | The entire CRM (HTML + CSS + JS, ~1k lines). The main deliverable. |
| `favicon.svg` | HoorayHR "h!" logo, used as favicon. Hand-drawn SVG recreation (no binary asset). |
| `chrome-extension.zip` | Packed MV3 extension (manifest v2.4). Unzip to edit, re-zip to ship. |
| `n8n-crm-webhook-updated.json` | The full n8n workflow to **Import from File**. This is the source of truth for the backend contract. |
| `SETUP.md` | German operator setup guide (sheet columns, n8n import, test steps). |
| `CLAUDE.md` | This file. |

There is **no build system**. To preview locally: serve the folder with any static server and
open `index.html`. (A throwaway Node static server on a spare port works; don't commit it.)

---

## 4. The CRM (`index.html`)

Single file. Structure: `<style>` (CSS variables + components) → markup (gate, app shell, topbar,
banner, views, modal/toast containers) → one big `<script>`.

### 4.1 Auth / gate
- Page is empty until the user enters the **n8n API key** in the gate and clicks unlock.
- `unlock()` calls `action:list`; on success the key is stored in `sessionStorage` (`li_crm_n8n_key`)
  and **published to the bridge** (`publishBridge()`). 401/403 → "invalid key".
- Key + cache live only for the tab session. `lockAndReload()` clears everything + bridge.
- Every request sends the key **both** as header `X-API-Key` and body field `apiKey`.

### 4.2 Storage keys
- `sessionStorage`: `li_crm_n8n_key` (KEY_SS), `li_crm_cache_v1` (leads cache), `li_crm_templates_v1`.
- `localStorage`: `li_crm_env` (prod|test), `li_crm_lang` (en|de|nl), `li_crm_viewmode` (list|board),
  `li_crm_user` (USER_LS — the current salesperson's identity / SDR id),
  bridge `li_crm_bridge_key` / `li_crm_bridge_env` / `li_crm_bridge_user`.

### 4.3 Views (topbar tabs; header is identical on every tab, active tab highlighted blue)
- **Leads** — stat tiles (counts by status, click to filter), lead-type pills (all/direct/partner),
  search, an **SDR filter** dropdown (whose contacts: all / unassigned / each team member) and a
  **Tag filter** dropdown (all / untagged / each tag in use), and a **List/Board toggle**:
  - **List**: detailed cards (name, lead-type badge, source tag, title, old→new company, size,
    notes, **message history**, actions).
  - **Board (Kanban, Pipedrive-style)**: one column per status in `STATUS_ORDER`, compact cards,
    **HTML5 drag-and-drop** between columns updates the lead's status. Dropping onto "In Pipedrive"
    opens the Pipedrive modal. Drag-and-drop is desktop-oriented (no touch DnD).
- **Messages** — manage templates (name + body, `{voornaam}`/`{bedrijf}` placeholders) and
  per-template conversion analytics computed from each lead's `messagesSent[]`.
- **Help & Process** — explains statuses, lead types, sources, data flow, extension install, and a
  table of n8n actions (live vs planned). Update this table when actions change.

### 4.4 Domain model
**Lead object** (see `normalizeLeads()`):
```
{ id, name, title, oldCompany, newCompany, size, url,
  status,            // one of STATUS_ORDER
  notes, manualType, // manualType: 'direct'|'partner'|null (overrides auto-detection)
  source,            // 'salesnav' | 'profile' | 'manual' | ''
  pipedrive,         // '' if not pushed; else dealId (or 'yes')
  sdr,               // string[]  — owners (USERS ids). A contact can belong to several people.
  tags,              // string[]  — free manual labels for grouping/filtering (besides lead type)
  messagesSent: [ { templateId, text, sentAt(ISO) } ] }
```
- **Statuses** (`STATUS_ORDER`): `new → benaderen (to approach) → messaged → lead → in-pipedrive`,
  plus `unqualified`, `not-interested`. Status class/label via `statusClass()`/`statusLabel()`.
- **Lead type** (`detectLeadType()`): title containing interim/freelance/zzp/ai → `partner`, else
  `direct`. Manually togglable (`manualType`). This is separate from `tags`.

**Team / identity (SDR).** `USERS` is a **hardcoded** array at the top of the script:
`{id, flag, label?}`. Members: 🇳🇱 Theo, Simon, Ramon, Ichelle, Nicolette, Other NL · 🇩🇪 Vivien,
Wanda, Raphael, Lucas, Other DE. The `id` is what is stored in a lead's `sdr` array and in the
sheet (the two "Other" rows use ids `Other NL` / `Other DE` but display simply as "Other" + flag).
After unlocking, `maybePromptIdentity()` forces an identity pick if `li_crm_user` is unset
(`openIdentityModal(true)`); the choice shows as a chip in the topbar and can be changed anytime.
New leads added/scraped default their `sdr` to `[currentUser]`. `sdr` is edited via checkboxes and
`tags` via a comma-separated input, both in the add/edit modal; both render as chips on list + board
cards. Filters: `activeSdr` / `activeTag` (special values `all`, `__none`).

**Template object**: `{ id, name, body, deleted }` (`deleted:true` = soft-deleted, filtered out on read).

### 4.5 Persistence model
- Edits mutate in-memory `leads`/`templates`, set `dirty=true` (amber dot + beforeunload warning),
  and update the session cache.
- **Save** (`syncToN8n`) → `action:sync` with `{leads}` then `persistTemplates()` (`action:save-templates`).
- **Refresh** (`reloadFromN8n`) → `action:list` then `loadTemplates()` (`action:list-templates`).
- Templates also persist immediately on each template edit/add/delete (so the library is durable).

### 4.6 Pipedrive in the CRM
- `addToPipedrive(id)` → `openPipedriveModal(id)`: a styled modal showing **exactly what will be
  created** (Organisation = newCompany, Person = name·title, Deal = "name — company", Note = job-change
  context). On open it runs `pipedrive-check` and shows a banner (already-exists warning / not-found / fail).
- `doPipedriveAdd(id)` → `action:pipedrive-add`; stores returned `dealId` in `lead.pipedrive`, sets status `in-pipedrive`.
- `checkLeadInPipedrive(id)` (🔍 on each not-yet-pushed card) → `action:pipedrive-check`, toasts result.
- Setting status to `in-pipedrive` (dropdown or board drop) without an existing deal opens the modal.

### 4.7 AI message drafting
- In the compose dialog, `composeGenerate()` → `action:ai-message` with `{system, prompt, lead}`.
- `AI_SYSTEM_PROMPT` constant holds the default system prompt (HoorayHR tone, short, warm, reply only
  with the message text, in the UI language). User can add an extra instruction per draft.

### 4.8 i18n
- `I18N = { en, de, nl }`. `t(key, vars)` looks up the current `lang`, falls back to `en`, then to the
  key itself, and replaces `{var}` placeholders from `vars`. **When you add a string, add all 3 langs.**
- `lang` persisted in `localStorage`. `applyI18n()` re-renders `[data-i18n]` nodes + tooltips.

### 4.9 Environment toggle
- prod vs test webhook (`N8N_URLS`). Choice persisted and pushed to the extension via the bridge.

---

## 5. n8n backend contract (source of truth: `n8n-crm-webhook-updated.json`)

- **Webhook** (single): `POST https://n8n.dev.hoorayhr.io/webhook/56bc510a-38d4-410f-ab5c-41ca63e825f8`
  (test variant: `/webhook-test/56bc510a-...`). Header auth + permissive CORS (`*`, allows `X-API-Key`).
- **Switch "Route by action3"** branches on `{{ $json.body.action }}`; fallback output → CORS preflight (204).
- All responses include the CORS headers.

### Actions
| action | request body (besides `action`,`apiKey`) | does | responds |
|---|---|---|---|
| `list` | — | read all leads from Leads sheet (incl. `sdr`/`tags`, JSON-parsed to arrays) | `{ leads:[...], templates:[] }` (templates always empty here; see `list-templates`) |
| `sync` | `leads:[...]` (each lead incl. `sdr:[]`, `tags:[]`) | append/update leads by `id`; `sdr`/`tags` stored as JSON strings | `{ ok:true }` |
| `import` | `leads:[...]` (from extension; may include `sdr:[user]`) | match existing by LinkedIn URL; add new (owner = incoming `sdr`) / reset status on company change **but preserve owner+tags** / skip unchanged | `{ ok:true }` |
| `pipedrive-add` | `lead:{name,title,oldCompany,newCompany,url,leadType,notes}` | create Organisation → Person → Deal → Note in Pipedrive | `{ ok:true, dealId, personId, orgId }` |
| `pipedrive-check` | `lead:{name,newCompany,url}` | Pipedrive person+org **search** (HTTP Request nodes, predefined Pipedrive credential) | `{ ok:true, personExists, orgExists, persons:[names], organizations:[names] }` |
| `ai-message` | `system, prompt, lead` | OpenAI chat (gpt-5.4-mini), lead JSON appended as context | `{ ok:true, text }` |
| `list-templates` | — | read Templates sheet, drop soft-deleted | `{ templates:[{id,name,body}] }` |
| `save-templates` | `templates:[{id,name,body,deleted}]` | append/update templates by `id` (incl. `deleted` flag) | `{ ok:true }` |

**`sdr` + `tags` (no separate webhook — they ride on `list`/`sync`/`import`):** both are JSON-array
strings in the sheet, arrays in the app. Nodes that carry them (all built-in Set/Sheets, no Code):
- `List: shape rows` — parses: `sdr = {{ $json.sdr ? JSON.parse($json.sdr) : [] }}` (same for `tags`).
- `Sync: shape row` — stringifies: `sdr = {{ JSON.stringify($json.sdr || []) }}` (same for `tags`).
- `Import: namespace existing` — keeps the sheet copy as `existingSdr` / `existingTags`.
- `Import: build updated row` (company changed) — sets `sdr = {{ $json.existingSdr }}`,
  `tags = {{ $json.existingTags }}` → **owner/tags survive a job-change reset**.
- `Import: build new row` — `sdr = {{ JSON.stringify($json.sdr || []) }}` (from the extension), `tags` idem.
- The three Google-Sheets **write** nodes use `autoMapInputData`, so they only write `sdr`/`tags` if
  those **header columns exist in the Leads sheet** (operator must add them — see §Google Sheet).

**Branch notes (for future edits):**
- Templates branches are pure built-in nodes: SplitOut → Set → Filter → (Aggregate) → Google Sheets / Respond. No Code.
- `Tpl: read sheet` has **Always Output Data** on so the Respond fires even when the sheet is empty.
- `pipedrive-check` chain: `PDcheck: prep` (Set: name/company/url with non-empty defaults so Pipedrive
  search never 400s) → `PDcheck: search person` (HTTP GET `/v1/persons/search`) → `PDcheck: search org`
  (HTTP GET `/v1/organizations/search`) → `PDcheck: respond` (builds the JSON via expression, reading
  `$('PDcheck: search person').item.json.data.items` etc.). HTTP nodes use
  `authentication: predefinedCredentialType`, `nodeCredentialType: pipedriveApi`, and `onError:
  continueRegularOutput` so a failed search still returns a result. **After import the operator may
  need to re-select the Pipedrive credential on these two HTTP nodes.**
- When adding the next action: add a Switch rule (keep rule order = output index order), wire the new
  branch's start node into the Switch output that sits *before* the CORS fallback (push CORS to the end).

### Google Sheet
- Spreadsheet `1M3Jkv4pAnWFVKk-XH7e4bKET2qms3-c3YAgELYKrYwc`.
- **Leads** tab (gid=0). Columns: `id, name, title, oldCompany, newCompany, size, url, status,
  manualType, notes, messagesSent, pipedrive, sdr, tags`. (`messagesSent`, `sdr` and `tags` are each
  stored as a JSON string, e.g. `sdr` = `["Raphael","Theo"]`, `tags` = `["priority"]`.)
  **The `sdr` and `tags` headers must exist** or the Sheets write nodes silently drop them.
- **Templates** tab (gid=42773857). Columns: `id, name, body, deleted`.

### Credentials (names as referenced in the workflow; secrets are NOT in the repo)
- Header auth: **`1-HoorayHR-BEST-AI-CRM!`** (id `IrKETCwb3KOAfewH`) — validates the API key.
- Google Sheets OAuth2: **`Google Sheets account`** (id `iVlN5hQV9AyjBIuu`).
- Pipedrive API: **`Pipedrive account`** (id `8tz75zYbcct9QACj`).
- OpenAI: **`Raphael OpenAI HoorayHR API Key`** (id `m0r83jO43Q53ftsy`), model `gpt-5.4-mini`.

---

## 6. Chrome extension (inside `chrome-extension.zip`, manifest v2.4)

MV3. Files: `manifest.json`, `popup.html`, `popup.js`, `background.js`, `bridge.js`, `i18n.js`.
Permissions: `activeTab, scripting, tabs, storage, alarms, notifications`. Hosts: `linkedin.com`,
`crm.internal-tool.online`. Uses the **same `N8N_URLS`** as the CRM.

- **popup.js** — detects the active tab:
  - Sales Navigator search/list → "Scrape this page" (`scrapeSalesNavDOM`) → preview → POST `action:import`.
  - LinkedIn / Sales Nav **profile** → "Add this profile" (`scrapeProfileDOM`) → auto-POST `action:import`;
    AND **auto-runs `pipedrive-check`** (`autoPipedriveCheck`) showing a badge (in PD / not in PD).
  - Key: prefers the CRM-bridged key, else a manual key from settings (`effectiveKey()`).
  - **Owner stamping (v2.4):** reads the bridged identity (`crmUser` ← `n8nUserFromCRM`) and, on
    `import`, maps each scraped lead to `{...lead, sdr:[crmUser]}` so the scraper owns the contact.
    If no identity is bridged yet, leads go in with empty `sdr` (assignable later in the CRM).
- **background.js** — UX nudging only: orange badge dot on profile pages, red "!" if no scrape in >7
  days, a Friday 10:00 reminder notification. No data handling.
- **bridge.js** — mirrors the CRM's localStorage key/env **and the chosen SDR** (`li_crm_bridge_user`
  → `n8nUserFromCRM`) into `chrome.storage.local` (see §2).
- **i18n.js** — `EXT_I18N = {en,de,nl}`, `extT()`; popup uses `L(key, vars)` with `{var}` replacement.

**DOM scraping is brittle** (LinkedIn markup changes). Selectors live in `scrapeSalesNavDOM` /
`scrapeProfileDOM`. If scraping breaks, fix selectors there.

To edit: unzip → edit → `node -e "new Function(fs.readFileSync('popup.js'))"` to lint → bump
`manifest.json` version → re-zip the 6 files. Operator reloads at `chrome://extensions`.

---

## 7. How to work on this (conventions & checks)

- **Editing `index.html`:** surgical edits; match the terse existing style. After edits, validate JS:
  `node -e "const h=require('fs').readFileSync('index.html','utf8');new Function(h.match(/<script>([\s\S]*)<\/script>/)[1])"`.
- **i18n:** any new user-facing string → add to `en`, `de`, `nl` (CRM `I18N`, extension `EXT_I18N`).
- **n8n:** never add a Code node. Build the full workflow programmatically when extending (read the
  current JSON, push nodes + a Switch rule + connections, re-serialise) to avoid hand-edit errors.
  Validate with `JSON.parse`. Keep prod webhook path unchanged.
- **Preview/verify:** serve statically and use a headless browser; seed demo leads by calling
  `normalizeLeads([...])` + `showApp()`/`render()` to bypass the gate (no live key needed). Note the
  served file can be browser-cached — disable cache or use a fresh port.
- **Reversibility:** the user expects you to test and then report honestly what works / what they must
  still do in n8n or the Sheet.

---

## 8. Known caveats / open risks (tell the user, don't silently ignore)

- **API key = full access.** Anyone with the key can read/write the whole sheet via the webhook. It's
  also written to `localStorage` (bridge) where any script on the CRM origin could read it. Fine for an
  internal MVP; revisit if this ever goes wider.
- **`pipedrive-check` credential injection:** the two HTTP Request nodes rely on the predefined Pipedrive
  credential adding the `api_token`. Verify on first run; fallback is to switch them to the native
  Pipedrive node's "Search" operation.
- **`pipedrive-add` has no built-in idempotency** beyond the optional pre-check — clicking twice can
  create duplicate deals. The check + the `pipedrive` flag mitigate this in the UI.
- **`size` modal field** uses a fixed dropdown; sheet stores it as text.
- **Drag-and-drop board** is HTML5 DnD (desktop). Touch devices fall back to the dropdown in list view.
- **Templates soft-delete:** a deleted template keeps a row with `deleted=true` (never hard-deleted by
  the app). `list-templates` filters it out.
- **Pipedrive search result shape** is assumed `data.items[].item.{id,name}`; if the Pipedrive API
  changes, adjust the `PDcheck: respond` expression.

---

## 9. Change history (so you know how we got here)

- **v1** — Base CRM: gate, leads list, statuses, templates (in-memory only), env toggle, trilingual,
  help/walkthrough. Extension: scrape Sales Nav + profiles → `import`. Backend had `list`/`sync`/`import`
  and (unused-by-UI) `pipedrive-add`/`ai-message`.
- **v2** — Removed left sidebar (everything in the topbar) + responsive **burger menu**; removed the
  initials avatars; added **message history** on lead cards; wired **Pipedrive add** and **AI drafting**
  into the UI; made **"message sent"** record the actual text into `messagesSent[]`; moved **templates
  into the DB** via `list-templates`/`save-templates` on the **same** webhook (built with no-code nodes);
  added the `pipedrive` column.
- **v3** — **Favicon** (HoorayHR logo); **Kanban board** view with drag-to-move; **nicer
  Pipedrive modal** showing exactly what gets created (fixed the `{name}` placeholder by adding `vars`
  support to `t()`); **`pipedrive-check`** duplicate detection as a new no-code action (HTTP Request
  search) used by both the CRM (🔍 + auto-check in the add modal) and the extension (auto-badge on
  LinkedIn profiles, manifest bumped to v2.3).
- **v4 (current)** — **User management / SDR ownership**: hardcoded `USERS` team, post-unlock
  **identity picker** + topbar chip, new `sdr` (owner) array column — a contact can belong to several
  people; identity is bridged to the extension so scraped contacts auto-assign to the scraper.
  **Manual tags**: free `tags` array column, chips on cards, comma input in the modal. Two new
  **filter dropdowns** (by SDR, by tag). `sdr`+`tags` ride on the existing `list`/`sync`/`import`
  actions (no new webhook), stored as JSON strings; import preserves owner+tags on a job-change reset.
  Two new **Leads sheet columns** (`sdr`, `tags`); extension manifest bumped to v2.4.

### Ideas explicitly marked "planned" (in the Help tab, not built)
- `enrich` — enrich a lead (company size, sector, email) from an external source after import.

---

## 10. The operator's preferences (how Raphael wants it)

- Communicates in **German**; keep summaries in German, code/comments can stay English.
- Wants **maximum simplicity for salespeople** — that beats elegance or feature count.
- **Same single n8n webhook**, new capabilities as new `action`s — never new webhooks.
- **No custom-code nodes in n8n.**
- Wants the assistant to **actually test** changes (preview/headless) and then report clearly what
  works and what he still has to do manually in n8n / the Google Sheet.
- Appreciates the n8n flow being importable and the Sheet changes spelled out explicitly.
