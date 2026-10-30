---
type: llm
focus: trace
weight: 1
---

The PACS base URL is obtained from GET /api/workspaces/{workspaceUid}/pacs-host using the 32-hex UID (not the numeric id) at the start of the upload run, not hardcoded or cached across runs.
