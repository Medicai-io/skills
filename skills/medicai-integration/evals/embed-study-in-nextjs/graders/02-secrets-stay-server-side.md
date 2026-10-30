---
type: llm
focus: trace
weight: 1
---

MEDICAI_CLIENT_SECRET and MEDICAI_REFRESH_TOKEN are read only in server-side code; the client component receives at most a short-lived access token from a server route.
