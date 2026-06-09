# LinkedIn Outreach CRM — Setup & Änderungen

Kurze Anleitung. Ziel: eine Salesperson kann dem Ablauf folgen, ein n8n-Admin richtet das Backend in ~15 Min ein.

> **Neu in v4 — wer bist du (SDR) + Tags.** Nach Eingabe des API-Keys fragt das CRM **„Wer bist du?“**
> (feste Namensliste mit Flagge). Jeder Kontakt bekommt eine **SDR-Spalte** (von wem ist der Kontakt —
> kann zu mehreren Personen gehören) und **frei vergebbare Tags**. Beides ist oben **filterbar** und
> wird in Google Sheets gespeichert. **Du musst dafür 2 neue Spalten im Sheet anlegen** (siehe §3) und
> die **Extension neu laden** (v2.4, siehe §4). Im n8n-Flow ist alles schon enthalten — **kein neuer
> Webhook**, `sdr`/`tags` reisen auf `list`/`sync`/`import` mit.

---

## 1. Was kann das CRM (`index.html`)

- **Kopfzeile** mit allen Aktionen + 👤 Identitäts-Chip + ⚙️ Einstellungen + 🔒 Sperren; aktiver Tab hellblau. Auf dem Handy → **☰ Burger-Menü**.
- **Wer bist du? (SDR):** Nach dem Entsperren wählst du **einmal** deinen Namen (Liste mit 🇳🇱/🇩🇪-Flagge). Der Chip oben zeigt deine Identität — Klick zum Wechseln. Neue Kontakte, die du anlegst oder scrapest, werden **dir** zugewiesen.
- **Von wem ist der Kontakt:** Auf jeder Karte stehen die **SDR-Chips** (Besitzer) — ein Kontakt kann zu mehreren gehören. Im Bearbeiten-Dialog hakst du die zuständigen Personen an. Oben filterst du per **SDR-Dropdown** (alle / nicht zugewiesen / einzelne Person) — so siehst und bearbeitest du auch die Kontakte anderer.
- **Tags:** frei vergebbare Labels pro Kontakt (zusätzlich zum automatischen Direkt-/Partner-Typ), kommagetrennt im Dialog, als Chips auf der Karte, filterbar per **Tag-Dropdown**.
- **Favicon** = HoorayHR-Logo (`favicon.svg`).
- **Zwei Ansichten für Leads** (Umschalter oben rechts):
  - **☰ Liste** — die klassische Karten-Liste mit Nachrichtenverlauf, Notizen, Aktionen.
  - **▦ Board** — Pipedrive-artiges Kanban: eine Spalte pro Status, **Karten per Drag & Drop** zwischen den Stufen verschieben (ändert den Status). Doppelklick auf eine Karte = bearbeiten.
- **Nachricht senden = Dialog**: Vorlage wählen (`{voornaam}`/`{bedrijf}` werden gefüllt), bearbeiten, **✨ mit KI generieren** (`ai-message`), kopieren, „Als gesendet markieren“. Verlauf landet auf der Karte.
- **Pipedrive** pro Lead:
  - **➕ Pipedrive** öffnet ein schönes Bestätigungs-Popup, das **genau zeigt, was angelegt wird** (Organisation, Person, Deal, Notiz) — und beim Öffnen automatisch prüft, ob die Person/Firma **schon in Pipedrive existiert**.
  - **🔍** prüft direkt, ob ein Lead schon in Pipedrive ist (Toast-Ergebnis).
  - Status → „In Pipedrive“ öffnet dasselbe Popup, falls noch kein Deal existiert.
- **Vorlagen** liegen in einem eigenen Sheet-Tab und werden über denselben Webhook geladen/gespeichert (`list-templates` / `save-templates`).

---

## 2. n8n — Workflow aktualisieren (PFLICHT)

Datei: **`n8n-crm-webhook-updated.json`** — dein aktueller Workflow + der neue **`pipedrive-check`**-Zweig. Alles auf **demselben Webhook** (`56bc510a-…`), nur neue `action`-Werte. **Kein Custom-Code-Node.**

1. n8n → **Workflows → Import from File** → `n8n-crm-webhook-updated.json` (oder die `PDcheck:`-Nodes + die neue Switch-Regel manuell übernehmen).
2. Credentials prüfen: Google Sheets, Pipedrive, OpenAI.
3. **Wichtig für `pipedrive-check`:** die zwei `PDcheck: search …`-Nodes sind **HTTP-Request-Nodes** auf die Pipedrive-Such-API. Sie nutzen *Predefined Credential Type → Pipedrive API*. Öffne beide Nodes einmal und wähle die **„Pipedrive account“**-Credential aus (falls n8n sie nach dem Import nicht automatisch zuordnet).
4. Workflow **aktivieren**.

### Alle `action`-Werte am Webhook
| action | tut |
|---|---|
| `list` / `sync` | Leads lesen / schreiben (Google Sheets) — inkl. **`sdr`** (Besitzer) + **`tags`**, als JSON-Array gespeichert |
| `import` | gescrapte Leads von der Extension, Abgleich per LinkedIn-URL — neue Kontakte erben den **`sdr`** des Scrapers; bei Firmenwechsel bleiben **Besitzer + Tags erhalten** |
| `pipedrive-add` | Organisation + Person + Deal + Notiz in Pipedrive anlegen |
| `pipedrive-check` | **NEU** — prüft per Pipedrive-Suche, ob Person/Firma schon existiert. Antwort: `{ ok, personExists, orgExists, persons[], organizations[] }` |
| `ai-message` | KI-Nachrichtenentwurf (OpenAI) |
| `list-templates` / `save-templates` | Vorlagen lesen / schreiben (Tab „Templates“) |

So sieht der Request für die Prüfung aus (CRM **und** Extension senden das):
```json
{ "action": "pipedrive-check", "apiKey": "…", "lead": { "name": "Anna de Vries", "newCompany": "Globex", "url": "https://linkedin.com/in/…" } }
```

---

## 3. Google-Sheet — erforderliche Änderungen

Spreadsheet: `1M3Jkv4pAnWFVKk-XH7e4bKET2qms3-c3YAgELYKrYwc`

- **Tab `Templates`** (für die Vorlagen-Bibliothek), Header in Zeile 1: `id | name | body | deleted`.
- **Spalte `pipedrive` im `Leads`-Tab** — ist in deinem aktuellen Workflow bereits in den Set-Nodes (`List/Sync/Import`) vorhanden. Stelle nur sicher, dass im Leads-Tab eine Spalte mit dem Header **`pipedrive`** existiert, damit „In Pipedrive ✓“ über Sitzungen erhalten bleibt.
- **NEU (v4) — zwei Spalten im `Leads`-Tab anlegen (PFLICHT):** schreibe in Zeile 1 zwei neue Header rechts dazu:
  **`sdr`** und **`tags`**. Beide bleiben in den Zellen leer bzw. enthalten ein JSON-Array, z. B.
  `sdr` = `["Raphael","Theo"]`, `tags` = `["priority"]`. Ohne diese Header **verwerfen** die Google-Sheets-Nodes
  Besitzer und Tags stillschweigend (sie schreiben nur in vorhandene Spalten). Reihenfolge egal — der Abgleich läuft über den Spalten-Namen.
  → Die komplette empfohlene Header-Zeile lautet damit:
  `id | name | title | oldCompany | newCompany | size | url | status | manualType | notes | messagesSent | pipedrive | sdr | tags`

Für `pipedrive-check` ist **keine** Sheet-Änderung nötig (es fragt nur Pipedrive ab).

---

## 4. Chrome-Extension (v2.4)

`chrome-extension.zip` neu gepackt.
- **NEU (v2.4) — Besitzer-Zuweisung:** Die Extension übernimmt (wie den API-Key) auch **deine im CRM gewählte Identität** über die Bridge und schreibt sie beim Scrapen in das `sdr`-Feld jedes Kontakts. So gehört ein gescrapter Kontakt automatisch **dir**. Voraussetzung: Du hast das CRM mindestens einmal geöffnet, entsperrt **und deinen Namen gewählt** (sonst gehen die Kontakte ohne Besitzer rein und können später im CRM zugewiesen werden).
- **Pipedrive-Badge** (seit v2.3): Auf einem LinkedIn-Profil prüft das Popup automatisch via `pipedrive-check`: 🟢 „Noch nicht in Pipedrive“ / ⚠️ „Bereits in Pipedrive: Person/Firma“.

**Wichtig:** Lade die Extension nach diesem Update **einmal neu** (`chrome://extensions` → ↻), sonst greift die Besitzer-Zuweisung nicht.

---

## 5. Schnelltest

1. CRM öffnen → entsperren → **„Wer bist du?“** → deinen Namen wählen → Chip oben zeigt z. B. „🇩🇪 Raphael“.
2. Leads laden. Auf einer Karte stehen **SDR-Chip(s)** + ggf. **Tag-Chips**.
3. Lead bearbeiten (✏️) → unter „Zugewiesen an (SDR)“ mehrere Personen anhaken, **Tags** kommagetrennt eintragen → Speichern → **💾 Speichern** (→ landet im Sheet als `sdr`/`tags`).
4. Oben per **SDR-Dropdown** auf eine Kollegin filtern → nur deren Kontakte; per **Tag-Dropdown** auf einen Tag filtern. Tag-Chip auf einer Karte anklicken = direkt nach dem Tag filtern.
5. Im Google Sheet die Spalten **`sdr`** / **`tags`** prüfen → enthalten JSON-Arrays (z. B. `["Raphael"]`).
6. **▦ Board** wählen → Karten ziehen → Status ändert sich; SDR/Tag-Chips bleiben sichtbar.
7. Lead → **➕ Pipedrive** → Popup zeigt Org/Person/Deal/Notiz + Prüf-Banner → „Create in Pipedrive“.
8. Extension neu geladen → auf einem LinkedIn-Profil scrapen → Kontakt erscheint im CRM **mit dir als SDR**.
9. Browser-Tab zeigt das HoorayHR-Favicon.
