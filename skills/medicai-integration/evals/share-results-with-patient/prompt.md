---
tags: [medicai-integration]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

We have an Express backend (TypeScript) for our clinic portal that already talks to Medicai with a service account (refresh-token flow, env vars MEDICAI_*). Add an endpoint POST /cases/:caseId/share-with-patient that returns a link the patient can open on her phone to see her results. Patients don't have Medicai accounts. Keep in mind we're still on the trial and testing with demo cases.
