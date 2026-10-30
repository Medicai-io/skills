---
type: llm
focus: trace
weight: 1
---

The viewer code re-sends the auth payload with a fresh access token on an interval shorter than the token lifetime (around 4 minutes or tied to expires_in).
