---
tags: [medicai-integration]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

A patient brought a CD with her abdominal CT. I copied it to ./cd-export on my laptop (it has the usual autorun.exe, dicomdir, and a nested dicom/ folder with files without extensions). Write a Node.js script that uploads the study into our Medicai workspace and prints the Medicai study id at the end so I can open it. Env has MEDICAI_CLIENT_ID, MEDICAI_CLIENT_SECRET, MEDICAI_CLIENT_NAME, MEDICAI_REFRESH_TOKEN, MEDICAI_WORKSPACE_ID and MEDICAI_WORKSPACE_UID. This is a trial workspace; the CD is a demo disc.
