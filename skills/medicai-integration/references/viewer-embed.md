# Embedding the DICOM Viewer

Put a full radiology-grade viewer (window/level, measurements, MPR, series navigation) into any web app with an iframe. No DICOM parsing, no viewer library, no canvas code.

Pick the method by who is looking:

| Viewer audience | Method |
|---|---|
| Nobody has a Medicai account yet (first demo, prototype before signup, "just show me a DICOM I have here") | Demo viewer URLs (below, no auth) |
| The logged-in user / your own backend account's studies | `/embed/viewer` + postMessage (below) |
| Anyone with a link (public or anonymized studies, patient handoffs) | `public-study` share link (below) |
| Your product's own end-users, each seeing only their data | Same `/embed/viewer`, driven by Enterprise integration-user JWTs; contact Medicai |

## Demo viewer: no account, no auth

Two public viewer URLs render without any credentials and are not subject to the integration-origin check: both were rendered inside an iframe from an unregistered localhost parent on 2026-11-04, so a plain iframe works from any origin, localhost included:

| URL | What it shows |
|---|---|
| `https://app.medicai.io/ohif-viewer/v3/viewer?staticWado=true&disableClose=true&dicomWebRoot=https:%2F%2Fapp.medicai.io/static-dicomweb&StudyInstanceUIDs=1.2.826.0.1.3680043.8.1055.1.20111103111148288.98361414.79379639` | A preset demo study served from a static DICOMweb root. `disableClose=true` hides the viewer's close button |
| `https://app.medicai.io/ohif-viewer/v3/localbasic` | An empty viewer with the OHIF local-files screen: the user picks DICOM files or a folder from their own disk through the viewer's UI and they render in the browser. Nothing is uploaded to Medicai |

```html
<iframe src="https://app.medicai.io/ohif-viewer/v3/localbasic"
        style="width:100%; height:800px; border:0" allow="fullscreen"></iframe>
```

Use the preset study when the user wants to see the viewer working before signing up, and the local-files viewer when they want to look at DICOM they already have (a CD, a download) without storing it anywhere. Say which is which: the preset study is Medicai demo data, not the user's. Both URLs are view-only; search, cases, sharing and the user's own stored studies need an account and Method 1.

## Method 1: /embed/viewer + postMessage

The iframe loads a fixed URL; auth and study selection arrive via postMessage.

```html
<iframe id="medicai-viewer"
        src="https://app.medicai.io/embed/viewer?type=OHIF_V3&withoutClose=true"
        style="width:100%; height:800px; border:0"
        allow="fullscreen"></iframe>
```

```js
const iframeWindow = document.getElementById("medicai-viewer").contentWindow;
iframeWindow.postMessage({
  type: "send-auth-data",
  workspaceId: 1234,          // numeric workspace id
  studyIds: [12345],          // numeric study ids, NOT Study Instance UIDs
  accessToken: "eyJh...",     // the user's JWT
}, "https://app.medicai.io");
```

When to post: the embed page announces itself to its parent with `postMessage({type: "ready-for-message"})` as soon as its listener exists. It can arrive before the iframe's `load` event fires (observed 2026-11-06: handshake first, `load` 100 ms later), so register the parent's `message` listener before setting the iframe `src`, then post the auth payload in response to that message; fall back to the `load` event plus a short delay if it never arrives. Posting before the listener exists is silently lost. With a whitelisted origin the study renders within a second of the post; this flow was verified end to end from `http://localhost:8765` on 2026-11-06 (an Intercom `intercom-snippet__ready` message also arrives and can be ignored).

Key facts:

- **Ids are numeric.** `workspaceId` is the numeric workspace id; `studyIds` are numeric study ids from search results (`id`), case profiles (`studies[].id`), or the workspace drive (`GET /api/workspaces/{workspaceId}/resources/my-drive?page=0&size=50`, entries in `content[]` with `resourceType === "STUDY"`). Study Instance UIDs do not work here.
- **Token refresh = re-post.** Access tokens live ~5 minutes; re-send the same message with a fresh token on an interval. There is no limit on how long a viewer stays open.
- **Parent origin allowlist.** The embed page accepts `send-auth-data` only from parent origins whitelisted for the account, matched as exact origins (scheme, host and port; no wildcards). Two ways to get an origin whitelisted: send the Medicai team the domain and URL the app will embed from, or configure it under Workspace settings > API once that option ships (announced as coming soon). From any other origin the page still sends `ready-for-message`, then drops the payload and logs `Ignoring received message ... from origin <yours>, because this origin is not a valid integration origin` in its own console, leaving a black viewer (observed 2026-11-03). Collect every origin the app will run on before writing the embed, the local one included: use `http://localhost:<port>` for development (one exact origin per port; the numeric loopback address would count as another origin, so do not mix them) and the production hostname. The public demo viewers above are not subject to this check. The `frame-ancestors` CSP on the route is report-only and is not what blocks.
- Multiple studies: put several ids in `studyIds`.
- `withoutClose=true` hides the viewer's close button (what you want inside your own layout).
- A query-param fallback exists but is discouraged (long token in the URL, no refresh).

### Next.js example (App Router)

Server route hands the client only what it needs:

```js
// app/api/viewer-session/route.js
import { getAccessToken } from "@/lib/medicai-auth"; // caching pattern from authentication.md

export async function GET(request) {
  const accession = new URL(request.url).searchParams.get("accession");
  const token = await getAccessToken();
  const headers = {
    Authorization: `Bearer ${token}`,
    "X-Medicai-Client": process.env.MEDICAI_CLIENT_NAME,
  };
  const results = await fetch(
    `https://app.medicai.io/api/workspaces/${process.env.MEDICAI_WORKSPACE_ID}/studies/metadata/search-study?limit=1&00080050=${encodeURIComponent(accession)}`,
    { headers }
  ).then(r => r.json());
  if (!results?.length) return Response.json({ error: "study not found" }, { status: 404 });
  return Response.json({
    workspaceId: Number(process.env.MEDICAI_WORKSPACE_ID),
    studyIds: [results[0].id],
    accessToken: token, // short-lived; client re-fetches this route to refresh
  });
}
```

Client component:

```jsx
"use client";
import { useEffect, useRef } from "react";

export function StudyViewer({ accession }) {
  const frame = useRef(null);
  useEffect(() => {
    let timer;
    async function send() {
      const auth = await fetch(`/api/viewer-session?accession=${accession}`).then(r => r.json());
      frame.current?.contentWindow?.postMessage(
        { type: "send-auth-data", ...auth },
        "https://app.medicai.io"
      );
      timer = setTimeout(send, 4 * 60 * 1000); // re-post before the ~5 min token expiry
    }
    const el = frame.current;
    el?.addEventListener("load", send);
    return () => { el?.removeEventListener("load", send); clearTimeout(timer); };
  }, [accession]);
  return (
    <iframe ref={frame}
      src="https://app.medicai.io/embed/viewer?type=OHIF_V3&withoutClose=true"
      style={{ width: "100%", height: 800, border: 0 }}
      allow="fullscreen" title="Medicai DICOM viewer" />
  );
}
```

Protect the server route with the app's own login; it returns a live token for the backend's account.

## Case gallery embed

Same pattern for a whole case (its studies and files): iframe `https://app.medicai.io/embed/gallery`, postMessage `{type: "send-gallery-auth-data", caseId, accessToken}`, refresh identically.

## Method 2: public-study link (no auth in the client)

For "anyone with the link" viewing. Backend flow, 2 calls:

1. `GET /api/studies/{studyId}/link-share` → `{uid, type}`
2. If `type` is not `PUBLIC`: `PATCH /api/studies/{studyId}/link-share` body `{"type": "PUBLIC"}`

Embed `https://app.medicai.io/public-study/{uid}` in an iframe. The uid lives ~3 days (cache up to 1 day; `PATCH .../link-share/reset` rotates it). `share-study/{uid}` is the same study wrapped in Medicai page chrome; `public-study` is the bare viewer.

Use only for genuinely public or anonymized studies, or deliberate patient handoffs; for patient flows prefer `PASSWORD_PROTECTED` share types with expiry (see `cases-and-sharing.md`). Never make a study public as a lazy way to show it to its owner; that is what Method 1 is for.

## UX notes

- Lazy-load the iframe (render only when the imaging tab/panel opens); the viewer streams pixel data and is heavy by nature.
- Give it room: minimum ~700px height for a usable read; full-width on mobile.
- Show a loading state until the first postMessage lands; the viewer is blank before auth arrives.

## Per-tenant PACS deployments (Enterprise)

Dedicated-PACS customers embed differently: a `/v3/viewer?StudyInstanceUIDs=...&accessToken=...` URL with study-scoped tokens minted machine-to-machine. If the user's Medicai host is not app.medicai.io, they are on that path; point them to Medicai's PACS Viewer Integration guide.
