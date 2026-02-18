**id: "SOP-XXX-000"**   
\# Eindeutige ID (z.B. SOP-FIN-001)   
**title: "Titel des Prozesses"**   
\# z.B. Automatisierte Rechnungsprüfung   
**version: "0.1.0-DRAFT"**   
\# SemVer (0.1.0 \= Draft, 1.0.0 \= Live)   
**owner: "Name Process Owner"**   
\# Fachliche Verantwortung (z.B. Ellen)   
**tech\_lead: "Name Tech Lead"**   
\# Technische Umsetzung (z.B. Andreas)   
**compliance\_lead: "Name"**   
\# Datenschutz/Sec (z.B. Jonas)   
**last\_updated: "YYYY-MM-DD"**   
**tags: \["Department", "Type", "n8n"\]**

# 

# **1\. Executive Summary (Für Menschen)**

**Ziel:**

Beschreibe in 1-2 Sätzen, welches Problem dieser Agent löst.

*Beispiel: Reduzierung der manuellen Dateneingabe im CRM um 80%.*

**Kern-Logik:**

Wie "denkt" der Agent?

*Beispiel: Der Agent liest E-Mails, extrahiert PDFs und vergleicht die Summen mit der Bestellung.*

**Output:**

Was ist das greifbare Ergebnis?

*Beispiel: Ein Eintrag in Salesforce und eine Slack-Nachricht.*

**Security Level:**

Low (Öffentliche Daten) | Medium (Interne Daten) | High (PII/Finanzdaten)

# 

# **2\. Agent Configuration (Für n8n / Maschinen)**

*Hinweis an Tech-Lead: Diese Werte werden vom Workflow-Parser ausgelesen.*

**Trigger:**

* **Type:** Scheduled (Cron) | Webhook | FileWatcher | Manual  
* **Schedule/Path:** Jeden Montag 08:00 oder 98\_INBOX/\*.pdf  
* **Target Audience:** (Optional) Wer ist betroffen?

**Model Settings:**

* **Primary Model:** gpt-4o (Standard) | claude-3-opus (Complex) | haiku (Fast)  
* **Privacy Fallback:** llama-3-local (Wenn Security Level \= High)  
* **Temperature:** 0.0 (Strikt) bis 0.7 (Kreativ)

**Data Connections (Context):**

* **Read Access:** Welche Ordner/Datenbanken darf der Agent lesen?  
* **Write Access:** Wo darf er schreiben? (z.B. AI\_WORKSPACE/\_temp)  
* **Tools:** Welche Python-Skripte aus \_Tools/ werden benötigt?

# 

# **3\. The Process Workflow**

## **Schritt 1: Input & Context (Vorbereitung)**

Was muss passieren, bevor die KI denkt?

*System Action:*

1. (z.B. Lade Datei aus Inbox)  
2. (z.B. Suche zugehörige Kunden-ID in SQL Datenbank)

## **Schritt 2: The Core Logic (Thinking)**

Der eigentliche KI-Task.

### **System Prompt**

*Anweisung: Kopiere dies in den n8n AI-Agent Node.*

DU BIST:  
(Rolle definieren, z.B. Ein erfahrener Buchhalter)

DEINE AUFGABE:  
(Was genau soll getan werden?)

KONTEXT DATEN:  
\- Input: {{input\_data}}  
\- History: {{historical\_context}}

REGELN:  
1\. (Regel 1\)  
2\. (Regel 2\)

### **Output Schema (JSON Enforcement)**

*Anweisung: Dies definiert die Struktur der Antwort.*

{  
  "field\_1": "string",  
  "field\_2": "number",  
  "decision": "APPROVE|REJECT",  
  "reasoning": "string"  
}

## **Schritt 3: Execution (Action)**

Was passiert mit dem JSON-Output?

*Agent Action:*

1. (z.B. Erstelle PDF in Ordner X)  
2. (z.B. Sende API Request an ERP)

# 

# **4\. Human Review & Governance**

**Review-Prozess:**

* Wer muss das Ergebnis freigeben? (Human-in-the-Loop)  
* Wo passiert die Freigabe? (Jira, Slack Button, Datei verschieben)

**Datenschutz & Compliance Check:**

* \[ \] PII Prüfung (Personenbezogene Daten?)  
* \[ \] Speicherort Prüfung (EU/US Server?)  
* \[ \] Löschkonzept (Werden Temp-Daten gelöscht?)

# 

# **5\. Roadmap / Changelog**

* **0.1.0** (Datum): Initiale Erstellung durch \[Name\].