# LinkedIn Outreach CRM — Setup & Änderungen (v2)

Kurze Anleitung. Ziel: eine Salesperson kann dem Ablauf folgen, ein n8n-Admin richtet das Backend in ~15 Min ein.

---

## 1. Was ist neu im CRM (`index.html`)

- **Keine Seitenleiste mehr** — alles oben in der Kopfzeile (Tabs + Aktionen + ⚙️ Einstellungen + 🔒 Sperren). Aktiver Tab ist hellblau, Kopfzeile bleibt auf allen Tabs gleich.
- **Responsive / Handy** — auf schmalen Bildschirmen klappt die Kopfzeile in ein **☰ Burger-Menü** ein.
- **Keine Initialen-Kreise mehr** auf den Lead-Karten (spart Platz).
- **Nachrichtenverlauf auf der Karte** — gesendete Nachrichten (Datum · Vorlage · Text) und Notizen werden direkt auf der Lead-Karte angezeigt.
- **➕ Pipedrive auf jeder Karte** — legt Organisation + Person + Deal + Notiz an (über `pipedrive-add`). Danach „In Pipedrive ✓“.
- **Status → „In Pipedrive“** → fragt „Deal jetzt anlegen?“, falls noch nicht vorhanden.
- **„Nachricht gesendet“ = Dialog** — Vorlage wählen (`{voornaam}`/`{bedrijf}` werden gefüllt), bearbeiten, **✨ mit KI generieren** (über `ai-message`), kopieren, „Als gesendet markieren“. Der Text wird in `messagesSent[]` gespeichert → die Analyse im Tab *Nachrichten* nutzt echte Daten.
- **Vorlagen werden in der DB gespeichert** — über **denselben Webhook**, zwei neue Aktionen `list-templates` / `save-templates`, in ein eigenes Sheet-Tab. Anlegen/Umbenennen/Löschen wird sofort persistiert (Löschen = Soft-Delete über `deleted`-Spalte).

> **Wichtig:** Es wird **kein neuer Webhook** verwendet — alles läuft über den bestehenden Webhook `56bc510a-…`, nur mit zwei zusätzlichen `action`-Werten (genau wie `list`, `sync`, `import`, `pipedrive-add`, `ai-message`).

---

## 2. n8n — Workflow aktualisieren (PFLICHT)

Datei: **`n8n-crm-webhook-updated.json`** — das ist dein bestehender Workflow + die zwei neuen Template-Zweige.

1. n8n → **Workflows → Import from File** → `n8n-crm-webhook-updated.json`.
   - Entweder als neuen Workflow importieren und alten deaktivieren, **oder** die zwei neuen Zweige (`Tpl: …`-Nodes + die zwei neuen Switch-Regeln) manuell in den bestehenden Workflow übernehmen.
2. Credentials prüfen: Google Sheets, Pipedrive, OpenAI (httpHeaderAuth-Key) müssen wie gehabt verbunden sein.
3. In den zwei neuen Google-Sheets-Nodes **„Tpl: read sheet“** und **„Tpl: write sheet“** das **Templates-Tab auswählen** (siehe Sheet-Änderungen unten). Der Sheet-Verweis lässt sich beim Import nicht automatisch setzen, weil n8n die Tab-Liste neu lädt.
4. Workflow **aktivieren**.

Neue `action`-Werte am bestehenden Webhook:
| action | wann | tut |
|---|---|---|
| `list-templates` | beim Entsperren / Verversen | liest Vorlagen aus dem `Templates`-Tab |
| `save-templates` | beim Bearbeiten / Speichern | schreibt Vorlagen zurück (inkl. Soft-Delete) |

---

## 3. Google-Sheet — erforderliche Änderungen

Spreadsheet: `1M3Jkv4pAnWFVKk-XH7e4bKET2qms3-c3YAgELYKrYwc`

### a) Neues Tab `Templates` (PFLICHT für die Vorlagen-Bibliothek)
Lege ein zweites Tab namens **`Templates`** an, mit diesen Spalten-Headern in Zeile 1:

```
id | name | body | deleted
```

Beim ersten Mal ist es leer → das CRM nutzt seine 3 Standard-Vorlagen, und sobald du im Tab *Nachrichten* etwas änderst (oder „Speichern“ drückst), werden sie hier hineingeschrieben.

### b) Spalte `pipedrive` im `Leads`-Tab (empfohlen)
Damit „In Pipedrive ✓“ auch **nach einem Reload / über Sitzungen hinweg** erhalten bleibt:

1. Im `Leads`-Tab eine neue Spalte mit Header **`pipedrive`** ergänzen (am Ende reicht).
2. Im Workflow zwei Set-Nodes um ein Feld erweitern (Typ *String*):
   - **„List: shape rows“** → neues Feld `pipedrive` = `={{ $json.pipedrive }}`
   - **„Sync: shape row“** → neues Feld `pipedrive` = `={{ $json.pipedrive }}`

Ohne (b) funktioniert alles — das „In Pipedrive“-Häkchen merkt sich dann nur innerhalb der aktuellen Sitzung.

---

## 4. Chrome-Extension

Unverändert und korrekt: sendet beim `import` weiterhin `name, title, newCompany, oldCompany, size, linkedinUrl, source` — genau das, was der `Import`-Zweig erwartet (`url` wird aus `linkedinUrl` gemappt). Keine neue Version nötig.

---

## 5. Schnelltest

1. CRM öffnen → mit n8n-Key entsperren → Leads + Vorlagen laden (Vorlagen kommen aus dem `Templates`-Tab).
2. Lead-Karte → **➕ Pipedrive** → Deal in Pipedrive prüfen, Karte zeigt „In Pipedrive ✓“.
3. Lead → **Nachricht gesendet** → Vorlage + „✨ Generieren“ testen → „Als gesendet markieren“ → Nachricht erscheint im Verlauf auf der Karte.
4. Tab **Nachrichten** → Konversionszahlen prüfen, eine Vorlage umbenennen → im `Templates`-Tab erscheint die Änderung.
5. Fenster schmal ziehen / Handy → **☰**-Menü erscheint und funktioniert.
