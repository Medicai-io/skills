---
type: llm
focus: trace
weight: 1
---

If any call to app.medicai.io/api is generated, it uses /api/workspaces/{workspaceId}/... routes and sends an X-Medicai-Client header; if no API call is generated at all, this passes.
