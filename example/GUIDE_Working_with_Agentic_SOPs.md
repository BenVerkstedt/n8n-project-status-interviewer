# **Guide: Arbeiten mit Agentic SOPs**

Dieses Dokument beschreibt den Workflow für das Prozess-Team (Ellen, Andreas, Jonas), um neue Automatisierungen von der Idee bis zur Produktion zu bringen.

## **Das Konzept: "Docs-as-Code"**

Wir schreiben keine toten Dokumente. Unsere SOPs (Standard Operating Procedures) sind **lebendig**.

1. **Lesbar für Menschen:** Das Management versteht den Prozess.  
2. **Lesbar für Maschinen:** n8n liest Konfigurationen, Prompts und Schemas direkt aus diesem Dokument.

## **Der Lebenszyklus einer SOP**

### **Phase 1: Definition (Process Owner / Ellen)**

*Ziel: Fachliche Anforderung klären.*

1. Kopiere templates/SOP-TEMPLATE\_Agentic\_Process.md in den passenden Unterordner (z.B. finance/).  
2. Benenne die Datei sinnvoll (z.B. SOP-FIN-001\_Invoice\_Check.md).  
3. Fülle **Kapitel 1 (Executive Summary)** und **Kapitel 3 (Workflow \- Textteil)** aus.  
4. Beschreibe im **System Prompt** in natürlicher Sprache, was die KI tun soll.  
5. Setze Status auf 0.1.0-DRAFT.

### **Phase 2: Architecture & Compliance (Compliance Lead / Nana)**

*Ziel: Sicherstellen, dass wir sicher bauen.*

1. Prüfe **Security Level**.  
2. Definiere **Model Settings** (Darf das zu OpenAI oder muss es lokal laufen?).  
3. Fülle die **Datenschutz Checkliste** in Kapitel 4 aus.  
4. Definiere Speicherorte für Vektordatenbanken oder Logs.

### **Phase 3: Implementation (Tech Lead / Andreas)**

*Ziel: Den Prozess zum Leben erwecken.*

1. Öffne n8n.  
2. Erstelle einen neuen Workflow.  
3. **WICHTIG:** Kopiere den Prompt nicht hardcoded in n8n. Nutze den Read SOP-Node, um den Prompt dynamisch aus der Markdown-Datei zu laden.  
   * *Warum?* Wenn Ellen den Prompt später ändern will, muss sie nur die Textdatei bearbeiten, nicht den Code.  
4. Definiere das **Output Schema (JSON)** im Dokument und übertrage es in den Structured Output Parser des LLMs.  
5. Implementiere die Tools (Python Skripte) im \_Tools Ordner, falls nötig.

### **Phase 4: Production & Review**

1. Wenn der Test erfolgreich ist: Setze Version auf 1.0.0.  
2. Commit & Push ins Git Repository.  
3. Der n8n-Produktionsserver zieht sich die Änderung.

## **Best Practices**

### **1\. Der Prompt gehört ins Repo**

Verstecke Prompts niemals in n8n-Nodes. Der Prompt ist Geschäftslogik. Er gehört in Kapitel 3.2 der SOP.

### **2\. JSON ist die Schnittstelle**

LLMs sind geschwätzig. Zwinge sie immer in ein JSON-Format (Kapitel 3.2).

* Schlecht: "Schreib mir eine Zusammenfassung."  
* Gut: {"summary": "string", "sentiment": "number"}.  
  So kann n8n danach zuverlässig weitermachen (z.B. If sentiment \< 0.5 then Alert Slack).

### **3\. Trennung von Logik und Daten**

Die SOP enthält **keine** echten Kundendaten oder Passwörter.

* Passwörter \-\> n8n Credentials.  
* Kundendaten \-\> Datenbanken.  
* SOP \-\> Nur die Logik.

### **4\. Das "Mudroom" Prinzip**

Der Agent darf niemals direkt in 01\_PRIVATE Originaldateien überschreiben.

* Input immer aus 98\_INBOX oder AI\_WORKSPACE/\_input.  
* Output immer als **neue Datei** oder **Draft**.  
* Der Mensch (oder ein strikter Logik-Step) schiebt das Finale Ergebnis ins Archiv.