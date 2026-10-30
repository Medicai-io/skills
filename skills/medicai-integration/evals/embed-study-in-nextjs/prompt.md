---
tags: [medicai-integration]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

I'm building a Next.js 15 app (App Router, JavaScript) for our orthopedic clinic and we have a Medicai trial workspace with a few demo knee MRIs in it. I want a page at app/patients/[id]/imaging/page.js that shows the patient's MRI in a viewer inside our layout. For now just look the study up by accession number, which I'll pass as a query param. I already have MEDICAI_CLIENT_ID, MEDICAI_CLIENT_SECRET, MEDICAI_CLIENT_NAME, MEDICAI_REFRESH_TOKEN and MEDICAI_WORKSPACE_ID in .env.local. Write the server route and the client component, with the files in a src/ folder.
