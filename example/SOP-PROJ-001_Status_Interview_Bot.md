## **id: "SOP-PROJ-001" title: "Automatisierter Projektstatus-Interviewbot" version: "0.1.0-DRAFT" owner: "Ellen Egyptien" tech\_lead: "Andreas" compliance\_lead: "Jonas (eHex)" last\_updated: "2025-10-28" tags: \["Project Management", "Reporting", "Interactive", "n8n"\]**

# 

# **1\. Executive Summary**

**Ziel:** Ablösung manueller Statusberichte durch einen KI-Agenten, der Projektverantwortliche proaktiv interviewt.

**Kern-Logik:** Der Agent akzeptiert keine "Copy-Paste"-Antworten. Er vergleicht Aussagen mit historischen Daten (Vektordatenbank/SQL) und hakt bei Widersprüchen (z.B. Zeitplanabweichungen) kritisch nach.

**Output:** Ein konsistentes, strukturiertes Statusdokument und ein Update im Analytics-Dashboard (BigQuery/Lightdash).

**Security Level:** High (Verarbeitung sensibler Projektdaten \-\> PII Check erforderlich).

# 

# **2\. Agent Configuration (Machine Readable)**

Dieser Bereich definiert die Parameter für den n8n-Workflow und das LiteLLM Gateway.

**Trigger:**

* **Type:** Scheduled (Jeden Freitag 09:00 Uhr) ODER Manual Trigger (via Slack/Mattermost Command).  
* **Target Audience:** Liste der Project Leads aus 01\_PRIVATE/02\_AREAS/PEOPLE/active\_leads.json.

**Model Settings:**

* **Primary Model:** gpt-4o (für das Interview/Logik \- via LiteLLM Proxy).  
* **Privacy Fallback:** llama-3-70b-local (wenn Projektdaten als "Strictly Confidential" markiert sind).  
* **Temperature:** 0.3 (Kreativ genug für Fragen, strikt bei Fakten).

**Data Connections (The Context):**

* **Long-term Memory:** Postgres/VectorDB (für historische Statusberichte).  
* **Analytics:** BigQuery (für KPI-Export).  
* **Tool Access:** \_Tools/project\_mgt/bin/get\_last\_report.py

# 

# **3\. The Process Workflow**

## **Schritt 1: Context Loading (Die Vorbereitung)**

*System Action (n8n):*

1. Lade den letzten Statusbericht des Projekts aus dem Git-Archiv oder der Datenbank.  
2. Lade die aktuellen Jira-Metriken (Offene Tickets, Blocker).  
3. Generiere den "Interview-Leitfaden" basierend auf Diskrepanzen (z.B. "Jira sagt 5 Bugs, letzter Report sagte 0").

## **Schritt 2: The Interview Loop (Interaktive Phase)**

Dies ist kein linearer Schritt, sondern eine Schleife in n8n/Chat-Interface (Slack/Teams).

### **System Prompt (Persona)**

Du bist ein erfahrener Projektmanager-Auditor. Deine Aufgabe ist es, den WAHREN Status des Projekts zu ermitteln.

DEIN WISSEN:  
\- Letzter Status: {{last\_report\_summary}}  
\- Geplantes Ziel: {{milestone\_goal}}

DEINE REGELN:  
1\. Sei höflich, aber kritisch.  
2\. Wenn der User sagt "Alles grün", aber Jira "5 kritische Bugs" zeigt \-\> Frag nach\!  
3\. Wenn der User Termine verschiebt, frag nach dem "Warum" und den Auswirkungen auf Abhängigkeiten.  
4\. Akzeptiere keine Ein-Wort-Antworten.

ZIEL:  
Erstelle am Ende eine valide Einschätzung für Risiken, Fortschritt und Budget.

## **Schritt 3: Synthesis & Documentation (System A Update)**

Nach Abschluss des Interviews generiert der Agent den finalen Bericht.

### **Output Schema (JSON Enforcement)**

{  
  "project\_id": "PROJ-123",  
  "reporting\_period": "YYYY-WW",  
  "rag\_status": "RED|AMBER|GREEN",  
  "management\_summary": "Kurzer Fließtext für die Geschäftsführung.",  
  "blockers": \["Blocker 1", "Blocker 2"\],  
  "achievements": \["Erfolg 1"\],  
  "timeline\_check": {  
    "on\_track": boolean,  
    "deviation\_reason": "string"  
  },  
  "sentiment\_score": 0.8  
}

*Agent Action:*

1. Erstelle Markdown-Datei: 01\_PRIVATE/01\_PROJECTS/{Project}/reports/{YYYY-MM-DD}\_Status.md.  
2. Push JSON-Daten nach BigQuery (für Lightdash Dashboard).

# 

# **4\. Human Review & Governance**

**Review-Prozess:**

* Der generierte Bericht wird als "Draft" an Ellen/Management geschickt.  
* Freigabe erfolgt per Button in n8n/Slack \-\> Erst dann wird der Status "offiziell".

**Datenschutz-Auflagen (per Jonas):**

* \[ \] PII-Check: Namen von Mitarbeitern dürfen nicht an öffentliche LLMs gesendet werden (Anonymisierung oder lokales Modell nutzen).  
* \[ \] Speicherort: Vektordatenbank muss on-premise oder in zertifizierter EU-Cloud liegen.

# 

# **5\. Roadmap & Open Points (Für Development Team)**

* \[ \] **Andreas:** Anbindung der "Context Database" (Wie speichern wir 'Krankheit von Teammitgliedern' so, dass der Bot es nächste Woche noch weiß?)  
* \[ \] **Jonas:** Entscheidung ob Local LLM (Llama 3\) performant genug für das Interview ist oder ob wir Azure OpenAI (Enterprise) nutzen.  
* \[ \] **Ellen:** Definition der Eskalationsstufen (Wann wird ein Bericht automatisch auf "RED" gesetzt?).