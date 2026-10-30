---
type: llm
focus: trace
weight: 1
---

Each DICOM file is POSTed as raw bytes to <pacs-url>/instances, ParentStudy ids are collected as a set, and POST /api/studies/initialize is called with workspaceUid, status=NEW and each PACS study id.
