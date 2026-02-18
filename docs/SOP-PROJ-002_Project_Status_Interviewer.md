**id: "SOP-PROJ-002" title: "Automatisierter Projektstatus-Interviewer (Slack + ElevenLabs)" version: "0.1.0-DRAFT" owner: "Ellen Egyptien" tech_lead: "Andreas" compliance_lead: "Jonas (eHex)" last_updated: "2026-02-20" tags: ["Project Management", "Reporting", "Interactive", "n8n"]**

#

# **1. Executive Summary (Für Menschen)**

**Ziel:**  
Wöchentliche Projektstatus-Updates nicht mehr manuell in Confluence einsammeln, sondern projektverantwortliche Personen automatisiert per Slack erinnern und per Voice-Interview (ElevenLabs) strukturiert durch den Status führen.

**Kern-Logik:**  
Der Workflow schickt einmal pro Woche eine Sammel-Nachricht in einen definierten Slack-Channel. Pro Projekt werden die zugehörigen Project Owner (Lookup über n8n DataTable) mit einem `@` erwähnt und erhalten einen Button, der ein ElevenLabs-Interview startet. Die Antworten (Current Things, Next Steps, Help Needed) werden anschließend als strukturierter Text in ein Google Doc geschrieben und mit der Projekt- und Kalenderwochen-Information in einer DataTable versioniert abgelegt. Nach dem Interview haben die Project Owner Zeit, die Google Docs bei Bedarf zu bearbeiten. Kurz vor dem Projektstatus-Termin werden alle Statusberichte der aktuellen Kalenderwoche zu einem kombinierten Markdown-Dokument zusammengeführt und ins GitHub-Repo geschrieben (eine Datei pro Woche, Single Source of Truth).

**Output:**  
- Pro Projekt und Kalenderwoche ein Google-Doc-Statusbericht (`Statusbericht_{ProjectName}_{YYYY}_KW{WW}`) im hinterlegten Drive-Ordner.  
- Ein Eintrag in der DataTable `project_status_overview` (inkl. `project_id`, `project_name`, `kw`, `year`, `document_id`, `drive_folder_id`).  
- Transparente Slack-Historie im gemeinsamen Project-Owner-Channel.  
- Wöchentlich eine kombinierte Markdown-Datei im Repo (z. B. `reports/combined/{YYYY}-KW{WW}-project-status.md`) mit allen Projektstatus der aktuellen KW.

**Security Level:**  
Medium–High (interne Projekt- und Personeninformationen; PII-Check empfohlen, insbesondere bei Freitext im Interview).

**Artifacts / Quellen (Repo):**  
- n8n-Workflow: `Project Status Interviewer - voice.json`  
- n8n-Workflow: `Project Status Consolidation.json` (wöchentliche Konsolidierung Google-Docs → Markdown → GitHub)  
- ElevenLabs-Agent (Konfiguration, Prompt, Tools): `elevenlabs_agent.json`

#

# **2. Agent Configuration (Für n8n / Maschinen)**

**Trigger:**  
- **Type:** `ScheduleTrigger` (wöchentlich)  
- **Schedule:** Montag, 15:00 Uhr (KW-basiert via `$now.toFormat('WW')` und `yyyy`)  
- **Target Audience:** alle aktiven Projekte aus der DataTable `project_status_project_owners` (ID: `CHXaFl7Pv82RthV4`), gruppiert nach `project_id` / `project_name`.

**Model / Voice Settings:**  
- **Voice Agent:** ElevenLabs Agent (Export: `elevenlabs_agent.json`)  
  - Agent-ID: `agent_1501kgkzbqnzfyys53p1820hq3a9`  
  - Name: `project-status-interviewer`  
  - Branch-ID: `agtbrch_3301kgkzbseafrx8erkqvbbg2wn7`  
  - Start-URL im Slack-Button:  
    `https://elevenlabs.io/app/talk-to?agent_id=agent_1501kgkzbqnzfyys53p1820hq3a9&branch_id=agtbrch_3301kgkzbseafrx8erkqvbbg2wn7&var_project_id={{ project_id }}`  
  - **LLM im Agent:** `gemini-2.5-flash` (Temperature 0).  
  - **TTS:** Modell `eleven_turbo_v2`, Voice-ID `cjVigY5qzO86Huf0OWal`; Stability 0.5, Similarity Boost 0.8.  
  - **ASR:** High Quality, Provider ElevenLabs, Format PCM 16 kHz.  
  - **Gespräch:** Max. Dauer 600 s, erste Nachricht: *"Hi, ready for a quick check-in on this week's project progress?"*  
- **LLM für Kontext-Aufbereitung (n8n):** `Google Gemini Chat Model` im Node „Report fetcher“ zur Zusammenfassung des letzten Berichts für ElevenLabs.

**ElevenLabs Agent – Tools (Webhooks zu n8n):**  
Alle Tools rufen die n8n-Cloud-Instanz `https://verkstedt.app.n8n.cloud/webhook/…` auf. Die Variable `project_id` wird beim Start des Interviews aus der URL (`var_project_id`) an den Agenten übergeben.

| Tool | n8n-Pfad | Zweck |
|------|-----------|--------|
| `verify_slot_is_empty` | `POST /webhook/verify_slot_is_empty` | Prüft, ob für aktuelle KW/Jahr bereits ein Status existiert. Body: `verify_slot_request`, `project_id`. |
| `fetch_last_report` | `POST /webhook/fetch-last-report` | Holt und fasst den letzten Projektbericht (Google Doc). Body: `last_report_request`, `project_id`. |
| `submit_status_report` | `POST /webhook/submit-status-report` | Speichert Interview-Ergebnis (Google Doc + DataTable). Body: `submit_status_request`, `activities`, `next_steps`, `support_needed`, `short_summary`, `project_id`, `slack_ts`. |
| `end_call` | (System) | Beendet das Gespräch vom Agenten aus. |

**Data Connections (Context):**  
- **Read:**  
  - n8n DataTable `project_status_project_owners` (Projekt-Owner-Lookup).  
  - n8n DataTable `project_status_overview` (historische Statusberichte).  
  - Google Docs (letzter Statusbericht pro Projekt via `document_id`).  
- **Write:**  
  - Neues Google Doc pro Statuslauf im projektbezogenen Drive-Ordner (`drive_folder_id`).  
  - Neue Zeilen in `project_status_overview` (pro Projekt, KW, Jahr).  
- **n8n-Nodes:**  
  - `Get a document in Google Docs` (Lesen des vorherigen Berichts).  
  - `Create a document` & `Update a document` (Anlage des neuen Berichts).  
  - `dataTable`-Nodes für Lookup & Insert.

#

# **3. The Process Workflow**

## **Schritt 1: Wöchentlicher Trigger & Projektauswahl**

**System Action (n8n):**
1. `Schedule Trigger` feuert einmal pro Woche (Montag 15:00).  
2. Node `Get row(s)` liest alle Zeilen der DataTable `project_status_project_owners`.  
3. Node `Code in JavaScript` gruppiert diese Zeilen nach `project_name` / `project_id` und baut pro Projekt eine kombinierte Owner-Liste mit Slack-IDs (`combined_owners` wie `<@U123>, <@U456>`).  
4. Node `Edit Fields` ergänzt:  
   - `slack_channel_id` (konfigurierter Channel)  
   - `current_kw = $now.toFormat('WW')`  
   - `year = $now.toFormat('yyyy')`.

## **Schritt 2: Slack-Benachrichtigung an Project Owner**

**System Action (n8n):**
1. Node `request for status message` (Slack) sendet pro Projekt eine Block-Message in den definierten Channel:  
   - Text: Bitte um Status-Update für `project_name`, KW + Jahr, inkl. `combined_owners` (Project Owner werden direkt getaggt).  
   - Button **"📝 Status abgeben"** mit Link zu ElevenLabs:  
     - enthält URL-Parameter `var_project_id={{ project_id }}`.
2. Die Slack-Nachricht ist damit der zentrale Einstiegspunkt, über den die Project Owner das Voice-Interview starten.

## **Schritt 3: ElevenLabs-Interview (Voice Agent)**

**Referenz:** Konversationslogik und Tools sind im Agent-Prompt in `elevenlabs_agent.json` (Key `conversation_config.agent.prompt.prompt`) definiert.

**Ablauf im Agenten (verkürzt):**
1. **Einstieg:** Erste Nachricht: *"Hi, ready for a quick check-in on this week's project progress?"*  
2. **Gatekeeper:** Agent ruft sofort `verify_slot_is_empty` auf (mit `project_id` aus Start-URL).  
   - **Falls Slot belegt (`slot_empty` = false):** Agent sagt, dass für diese Woche bereits ein Bericht vorliegt, und beendet mit `end_call`.  
   - **Falls Slot frei:** Weiter zu 3.  
3. **Kontext:** Agent sagt z. B. *"Let me pull up the last summary for context..."* und ruft `fetch_last_report` auf.  
4. **Interview:** Mit Rückgabe des letzten Berichts: Brücke (*"I see that last week the focus was on [Summary]. What are the current activities happening now?"*), dann:  
   - **Current activities** (Woran wird gearbeitet?)  
   - **Next steps** (Geplante nächste Schritte)  
   - **Support/refinement needed** (Hilfe oder Verfeinerung nötig?)  
5. **Abschluss:** Agent fasst zusammen, fragt ob der Bericht gespeichert werden soll, ruft `submit_status_report` mit `activities`, `next_steps`, `support_needed`, `short_summary`, `project_id`, `slack_ts` auf, bestätigt dem User und beendet mit `end_call`.

**Regeln im Agent-Prompt (Auszug):** Automation (kein Abfragen von Datum/Bestätigung; `verify_slot_is_empty` entscheidet). Voice-first, kurze Sätze. Keine Emojis/JSON/URLs in gesprochenem Text. Keine Halluzinationen beim letzten Summary – auf Tool-Rückgabe warten.

## **Schritt 4: Status schreiben & versionieren**

**System Action (n8n):**
1. Node `get_rows1` liest alle bisherigen Einträge aus `project_status_overview` für das betroffene `project_id`.  
2. Node `Sort createdAt desc1` sortiert die Einträge nach `createdAt` (neuester zuerst).  
3. Node `Select earliest entry1` nimmt den neuesten Eintrag als Basis (enthält `drive_folder_id`, `project_name`, `project_owner` etc.).  
4. Node `get_date_for_agent1` berechnet erneut `current_kw` und `year` für den Dokumentnamen.  
5. Node `Create a document` legt im zugehörigen Google-Drive-Ordner (`drive_folder_id`) ein neues Google Doc an:  
   - Name: `Statusbericht_{ProjectName}_{YYYY}_KW{WW}`.  
6. Node `Update a document` fügt den eigentlichen Inhalt ein, z.B.:  
   - `## Current things: {activities}`  
   - `## Next steps: {next_steps}`  
   - `## Support needed: {support_needed}`.  
   - **Report-Format:** Die Abschnitte nutzen die Überschriften „Current things“, „Next steps“, „Support needed“. Der Agent (ElevenLabs) liefert für `activities` und `next_steps` thematisch gegliederte Texte mit Bulletpoints; die Strukturvorgaben stehen im Agent-Prompt (Sektion „Report output format“) in `agent_configs/project-status-interviewer.json`.  
7. Node `Insert row` schreibt anschließend eine neue Zeile in `project_status_overview` mit:  
   - `year`, `kw`, `project_name`, `project_owner`, `project_id`, `document_id`, `drive_folder_id`.  
8. Node `Post Google Doc link in thread` (Slack) postet im gleichen Slack-Thread eine Bot-Antwort mit dem Link zum angelegten Google Doc (zum Nachbearbeiten durch POs bzw. weitere Projekt-Owner).  
9. Node `Update Slack message` setzt die Reaction (weißer Haken) an der ursprünglichen Nachricht.  
10. Node `Respond to Webhook4` bestätigt ElevenLabs den erfolgreichen Abschluss (kann für Error-Handling genutzt werden).

## **Schritt 5: Historische Kontextabfrage (optional)**

**System Action (n8n):**
1. Node `Fetch last report` dient als Webhook-Endpunkt für den ElevenLabs-Agenten, falls dieser vor dem Interview den letzten Status benötigt.  
2. Node `get_rows` holt alle Zeilen aus `project_status_overview` für `project_id`.  
3. Node `Sort createdAt desc` sortiert nach Erstellungsdatum.  
4. Node `Select earliest entry` nimmt den neuesten Eintrag und liefert `document_id`.  
5. Node `Get a document in Google Docs` lädt den Inhalt des letzten Statusberichts.  
6. Node `Google Gemini Chat Model` + `Report fetcher` erstellen daraus eine kurze, gesprochene Zusammenfassung als JSON:  
   - `{"previous_summary": "…"}`
7. Node `Respond to Webhook2` liefert dieses JSON an ElevenLabs zurück, damit der Voice-Agent mit historischem Kontext starten kann.

## **Schritt 6: Wöchentliche Konsolidierung (Single Source of Truth)**

**Hinweis:** Dieser Schritt wird von einem **separaten** n8n-Workflow ausgeführt („Project Status Consolidation“).

**Zweck:**  
Kurz vor dem Projektstatus-Termin werden alle Statusberichte der aktuellen Kalenderwoche zu einem kombinierten Markdown-Dokument zusammengeführt und ins GitHub-Repository dieses Projekts geschrieben. So haben Project Owner nach dem ersten Reminder und dem Interview noch Zeit, die Google Docs zu bearbeiten; die finale Fassung wird dann automatisch als Single Source of Truth versioniert.

**System Action (n8n – Consolidation Workflow):**
1. **Trigger:** Schedule (z. B. einmal pro Woche, konfigurierbare Uhrzeit wie Donnerstag 08:00 oder Tag vor dem Projektstatus-Meeting). Eingabe: aktuelle Kalenderwoche (`$now.toFormat('WW')`, `$now.toFormat('yyyy')`).
2. **Datenquelle:** DataTable `project_status_overview` – Filter auf `kw` = aktuelle KW und `year` = aktuelles Jahr; pro **distinkter** `project_id` (bzw. `project_name`) **ein** Eintrag verwenden (bei Mehrfacheinträgen z. B. neuester Eintrag nach `createdAt`).
3. **Pro Projekt:** `document_id` aus der Tabelle → Google-Doc-Inhalt abrufen („Get document“) → Inhalt als Markdown aufbereiten (Export oder Konvertierung je nach API-Format).
4. **Kombination:** Ein Markdown-Dokument pro Woche mit Struktur: Überschrift „Week {KW} of {YYYY} ({Datumsbereich})“, dann pro Projekt ein Abschnitt mit Projektname (ggf. Owner), „Current things:“, „Next steps:“, „Help and refinement support needed:“.
5. **Ablage:** Commit ins GitHub-Repository, eine Datei pro Woche im Ordner `reports/combined/`, Benennung nach Kalenderwoche und Name: z. B. `reports/combined/{YYYY}-KW{WW}-project-status.md`. Bei fehlenden Einträgen für die aktuelle KW: Workflow sauber beenden, kein Commit mit leerem Inhalt.
6. **Stretch Goal (Roadmap):** Confluence-Sync – kombinierten Projektstatus aus dem GitHub-Repo in die bestehende Confluence-Seite übernehmen (ersetzt manuelles Copy-Paste).

#

# **4. Human Review & Governance**

**Review-Prozess:**  
- Project Owner geben ihren Status selbst ab; das Google Doc fungiert als primäre Quelle für spätere Anpassungen.  
- Änderungen am Status nach dem Interview erfolgen direkt im Google Doc (Versionierung übernimmt Google Drive).  
- Optional: Ein separater Review-Schritt (z.B. durch Ellen) kann etabliert werden, indem die Docs zunächst in einen _Draft_-Ordner geschrieben werden und erst nach Freigabe in den finalen Ordner verschoben werden.

**Datenschutz & Compliance Check:**  
- \[ \] Enthaltene Personen- oder Kundendaten (Namen, E-Mails etc.) dürfen nicht in externe LLMs gehen; der ElevenLabs- und ggf. Gemini-Einsatz ist mit Jonas abzustimmen.  
- \[ \] Webhook-Payloads (aktuell Projekt- und Statusinhalte) regelmäßig auf PII prüfen; falls nötig, Pseudonymisierung im ElevenLabs-Frontend vornehmen.  
- \[ \] Speicherort der Google-Docs und DataTables (Org-Account) dokumentieren; Aufbewahrungs- und Löschfristen definieren.  
- \[ \] ElevenLabs: Im Export ist `record_voice: true` gesetzt; Nutzende akzeptieren per Widget die Aufzeichnung (Terms). Retention und Löschung von Audio/Transcript in den ElevenLabs-Einstellungen prüfen (`privacy` in `elevenlabs_agent.json`).

#

# **5. Roadmap / Comfort Features**

- \[x\] **Slack-Feedback nach erfolgreichem Update:**  
  Nach Abschluss des Interviews wird automatisch eine Antwort unter der ursprünglichen Slack-Nachricht gepostet (z. B. Bestätigung/Checkmark). Der ElevenLabs-Agent übergibt `slack_ts` an `submit_status_report` – n8n antwortet per Slack API im Thread. *(Implementiert im Workflow.)*  
- \[x\] **Erneutes Bearbeiten ermöglichen:**  
  Im Slack-Thread wird der Link zum erzeugten Google Doc gepostet (Node „Post Google Doc link in thread“), damit POs und ggf. weitere Projekt-Owner das Dokument vor der Konsolidierung nachbearbeiten können. *(Implementiert im Workflow.)*  
- \[ \] **Slot-Reservierung & Doppelbuchungen verhindern:**  
  Der Webhook `verify_slot_is_empty` prüft für eine Kombination aus `project_id`, `kw`, `year`, ob bereits ein Eintrag existiert (`return_boolean.slot_empty`). Dieser Mechanismus kann erweitert werden, um Mehrfacheingaben zu erkennen und gezielt auf "Update statt Neu-Anlage" umzuschalten.  
- \[ \] **Wöchentliche Konsolidierung:**  
  Siehe Schritt 6; neuer n8n-Workflow „Project Status Consolidation“ (Export: `Project Status Consolidation.json`). Kombiniert alle Status-Docs der aktuellen KW zu einer Markdown-Datei und committet sie ins Repo (`reports/combined/{YYYY}-KW{WW}-project-status.md`).  
- \[ \] **Confluence-Sync (Stretch):**  
  Projektstatus aus GitHub (kombinierte Markdown-Datei) in die bestehende Confluence-Projektstatus-Seite übertragen (ersetzt manuelles Copy-Paste).  
- \[ \] **Dashboard-Integration:**  
  Zukünftig können die Daten aus `project_status_overview` in ein Reporting-Dashboard (z.B. Infinity Gate / Lightdash) überführt werden, um Aggregatsichten über alle Projekte zu erhalten.

