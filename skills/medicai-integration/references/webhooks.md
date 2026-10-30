# Webhooks

React to what happens in a Medicai workspace (a study arrives, a document lands on a case, a case is created) with an HTTP POST to an endpoint you control. Use webhooks instead of polling whenever the app needs to "do something when...".

Webhooks are per-workspace. Manage them in the Medicai UI under **Workspace settings → Webhooks** (create, test, reactivate; the signing secret is shown once at creation, copy it immediately). A management API also exists (`/api/workspaces/{workspaceId}/webhooks`, numeric workspace id) with create/update/delete, `.../test`, `.../redeliver`, and `GET .../webhooks/event-types` for the subscribable list. If the Webhooks tab is missing, the workspace needs it enabled by Medicai.

## What you receive

POST with JSON envelope and two headers: `X-Medicai-Signature` (`t=<unixSeconds>,v1=<hexHmac>`) and `X-Medicai-Event-Id` (same value on every retry; use it as your idempotency key).

```json
{
  "externalId": "a1b2c3d4-...",
  "type": "CASE_DOCUMENT_ADDED",
  "createdAt": "2026-07-27T10:30:00Z",
  "workspaceId": 42,
  "resourceType": "CASE",
  "resourceId": 1234,
  "apiVersion": "v1",
  "data": { "relatedIds": [9001, 9002] }
}
```

**Payloads carry identifiers only**, never patient names, file names, or titles. You fetch details through the API with your own token.

Events most prototypes care about: `STUDY_CREATED`, `CASE_CREATED`, `CASE_STUDY_ADDED`, `CASE_DOCUMENT_ADDED` (its `data.relatedIds` are the newly attached document ids), `DOCUMENT_CREATED`, plus case status/priority/assignee change events.

## Verify every request

`v1` is HMAC-SHA256 of `"<t>.<raw-body>"` keyed with the signing secret. Use the raw bytes (no re-serializing), compare constant-time, reject stale timestamps:

```js
const crypto = require("crypto");
function verify(rawBody, header, secret, toleranceSeconds = 300) {
  const parts = Object.fromEntries(header.split(",").map(p => p.split("=")));
  const expected = crypto.createHmac("sha256", secret).update(`${parts.t}.${rawBody}`).digest("hex");
  const a = Buffer.from(expected), b = Buffer.from(parts.v1 ?? "");
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) return false;
  return Math.abs(Date.now() / 1000 - Number(parts.t)) <= toleranceSeconds;
}
```

## Delivery rules that shape your handler

- **Return 2xx immediately**, process asynchronously (15s response timeout counts as failure).
- **At-least-once delivery**: deduplicate on `X-Medicai-Event-Id`.
- Retries: 8 attempts with growing backoff; 50 consecutive failures suspend the subscription (reactivation re-pings your endpoint).
- Creation only succeeds if your endpoint answers 2xx to a synchronous ping at save time, so deploy the handler before creating the subscription.
- Endpoint must be HTTPS, publicly resolvable (no localhost/private IPs; use a tunnel like ngrok during development), no redirects, no credentials in the URL.
- New subscriptions start from now; no historical events.

## The standard integration shape

```
webhook arrives -> fetch the resource by id -> process -> upload result -> attach to the case
```

Download: `GET /api/files/{id}`. Upload: `POST /api/workspaces/{workspaceId}/files` (multipart `files` + `fileDate`). Attach: `POST /api/cases/{caseId}/resources` with `{"studyIds": [], "fileIds": []}`. One gotcha: attaching your own result emits `CASE_DOCUMENT_ADDED` too, so skip events your own uploads produce (track the file ids you created).
