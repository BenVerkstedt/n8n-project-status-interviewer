## Project Status Interviewer – n8n + ElevenLabs

This repository contains the configuration and SOPs for the automated weekly project status interviewer, built with **n8n** and an **ElevenLabs voice agent**.

The goal is to replace the manual Confluence status roundup with:
- a weekly Slack reminder in a shared channel,
- a voice interview per project (ElevenLabs),
- and a structured Google Docs report plus a tracking table in n8n.

---

## Repository Structure

- `docs/`  
  SOPs for the workflow, in particular `SOP-PROJ-002_Project_Status_Interviewer.md` (human + machine readable spec).

- `n8n/`  
  n8n workflow export, e.g. `Project Status Interviewer - voice.json`.  
  Import this file into n8n to recreate the workflow.

- `agent_configs/`, `agents.json`, `tools.json`, `tests.json`, `tool_configs/`, `test_configs/`  
  ElevenLabs **Agents CLI** project structure. Used to pull/push the voice agent configuration.

- `example/`  
  Reference material (e.g. manually written weekly status from Confluence and SOP templates).

---

## Prerequisites

- **Node.js** ≥ 18 (for the ElevenLabs CLI).  
- An **ElevenLabs** account with access to the Agents platform and API key.  
- An **n8n** instance (cloud or self-hosted) with:
  - Slack credentials,
  - Google Docs / Google Drive credentials,
  - access to DataTables.

---

## Setup – ElevenLabs Agents CLI

1. Install the Agents CLI (globally):

```bash
npm install -g @elevenlabs/agents-cli
```

2. Authenticate:

```bash
elevenlabs login
```

3. Initialize (already done in this repo, shown for reference):

```bash
cd n8n-project-status-interviewer
elevenlabs agents init
```

This creates:

```text
agents.json
tools.json
tests.json
agent_configs/
tool_configs/
test_configs/
```

4. Pull existing agents from ElevenLabs (to sync changes made in the web UI):

```bash
elevenlabs agents pull
# or, to override local configs with remote changes:
elevenlabs agents pull --update
```

Your project status interviewer agent will appear as a JSON config in `agent_configs/` and is also described in `docs/SOP-PROJ-002_Project_Status_Interviewer.md`.

> Note: The file `elevenlabs_agent.json` is a one-off export and mainly used as reference. The canonical source for CLI operations is `agent_configs/` + `agents.json`.

---

## Setup – n8n Workflow

1. Open n8n and import the workflow JSON from:

```text
n8n/Project Status Interviewer - voice.json
```

2. Configure required credentials in n8n:
- Slack (for the weekly status request message and channel),
- Google Docs / Drive (for creating & updating status docs),
- DataTables (for `project_status_overview` and `project_status_project_owners`).

3. Verify the schedule node:  
Ensure the weekly trigger (e.g. Monday 15:00) and DataTable IDs match your environment.

4. Update the ElevenLabs URL in the Slack button if needed:  
The SOP and workflow currently assume a URL of the form:

```text
https://elevenlabs.io/app/talk-to?agent_id=...&branch_id=...&var_project_id={{ $('Code in JavaScript').item.json.project_id }}
```

---

## Dependencies

This repo is configuration-first; the only runtime dependency tracked here is the ElevenLabs Agents CLI for local agent management.

- **Global CLI (recommended):**

```bash
npm install -g @elevenlabs/agents-cli
```

- **Or as a local dev dependency:**

```bash
npm init -y              # if you want a package.json
npm install --save-dev @elevenlabs/agents-cli
```

In that case, you can run:

```bash
npx elevenlabs agents pull
```

---

## SOP & Governance

The operative and technical behaviour of this workflow (including prompts, tools and data flows) is documented in:

- `docs/SOP-PROJ-002_Project_Status_Interviewer.md`

For any changes to the agent prompt or workflow logic:
- update the SOP first,
- then align n8n and ElevenLabs configuration accordingly,
- and commit both config + SOP together.

