# Uploading Studies and Files

Two different pipelines. Pick by file type:

- **DICOM studies** (.dcm, imaging from a scanner or exported from another PACS): upload to the workspace PACS, then register the study with the API.
- **Ordinary documents** (PDF reports, referral letters, images): upload through the main API's file endpoint.

Your token must include the `orthanc:instances:create` scope for Pipeline A (request it at login; see `authentication.md`). Send `X-Medicai-Client` on all calls in both pipelines.

## Pipeline A: DICOM studies

### A0. Filter to actual DICOM files first

User-supplied folders (especially CD/DVD exports) are full of vendor junk, and DICOMs usually have no file extension. Detect by reading the first ~150 bytes: it is DICOM if bytes [128..131] spell `DICM`, or (older files) bytes [0,1] equal [8,0]. Skip files named `dicomdir`/`dirfile` (they match but are redundant). Archives (zip/rar) must be unpacked first; Medicai does not accept them. Uploading non-DICOM to the PACS wastes the workspace's bandwidth and can get the source blocked. If zero files pass the filter, the user almost certainly picked the wrong folder; say so instead of uploading.

Read sequentially from optical media; parallelize freely from SSDs.

### A1. Find the PACS base URL for the workspace

```
GET {MEDICAI_API_HOST}/api/workspaces/{workspaceUid}/pacs-host
Authorization: Bearer <ACCESS_TOKEN>
```

Pass the 32-hex workspace UID (`uniqueIdentifier` from `GET /api/workspaces`), not the numeric id. Returns a plain-text URL on a different hostname than the API. **Do not cache it; call this before every new upload session.**

### A2. Upload DICOM instances to the PACS

```
POST <pacs-root>/instances
Authorization: Bearer <ACCESS_TOKEN>
X-Medicai-Client: <integration-name>

<raw DICOM file bytes>
```

One request per file. Each success returns `{"ParentStudy": "<orthanc-study-id>"}`; collect the **distinct** ids in a set (one folder can contain several studies). Errors: `OrthancError: "Bad file format"` / `"Inexistent tag"` = invalid DICOM, warn and continue; 50x = retry with backoff.

### A3. Register each study with the Medicai API

The PACS having the files does not mean the Medicai application knows about them. For every distinct ParentStudy id:

```
POST {MEDICAI_API_HOST}/api/studies/initialize?workspaceUid=<32-hex-uid>&status=NEW&id=<orthanc-study-id>
Authorization: Bearer <ACCESS_TOKEN>
```

Empty body. `status=NEW` for a normal study (other values serve case-request flows). The response is the study as a drive row: `{id, resourceType: "STUDY", studyInstanceUid, orthancId, modalities, patient, previewUrl, ...}`. Its numeric `id` is the Medicai study id the viewer takes and cases attach; read it from this response and print or store it. There is no need to search for the study afterwards or to parse DICOM tags to find it. After this, the studies are searchable, viewable, and attachable to cases; refresh any study list the app is showing.

### Finding studies later

```
GET {MEDICAI_API_HOST}/api/workspaces/{workspaceId}/studies/metadata/search-study?limit=10&offset=0&<DICOM-tag>=<value>
Authorization: Bearer <ACCESS_TOKEN>
```

Query by DICOM tag values: exact (`00080060=MR`), any-of (`00080060=MR,CT`), text wildcards (`00100010=Alex*`, `00100010=Al*dru`), date ranges (`00080020=19940423-19950423`, open-ended `00080020=19940423-`). Common tags: accession `00080050`, patient ID `00100020`, patient name `00100010`, modality `00080060`, study date `00080020`. The response's numeric `id` field is what the embed viewer needs.

## Pipeline B: ordinary documents

```
POST {MEDICAI_API_HOST}/api/workspaces/{workspaceId}/files
Authorization: Bearer <ACCESS_TOKEN>
X-Medicai-Client: <integration-name>
Content-Type: multipart/form-data
```

Multipart fields, strictly paired:

- `files`: the binary content. Repeat the key once per file.
- `fileDate`: ISO8601 with at most millisecond precision. One per file, same order.

Node example:

```js
import { readFile } from "node:fs/promises";

const form = new FormData();
const buf = await readFile("./report.pdf");
form.append("files", new Blob([buf], { type: "application/pdf" }), "report.pdf");
form.append("fileDate", new Date().toISOString().slice(0, 23) + "Z");

const res = await fetch(`${process.env.MEDICAI_API_HOST}/api/workspaces/${process.env.MEDICAI_WORKSPACE_ID}/files`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${await getAccessToken()}`,
    "X-Medicai-Client": process.env.MEDICAI_CLIENT_NAME,
  },
  body: form,
});
const uploaded = await res.json(); // one drive row per file, in request order: { id, resourceType: "FILE", name, url, mimeType, ... }
const fileIds = uploaded.map(row => row.id); // numeric ids, what cases take
```

Errors:

- **400**: file count does not match `fileDate` count.
- **415**: disallowed file type (DICOM belongs in Pipeline A).

Related file endpoints: `GET /api/files/{id}` (download), `PUT /api/files/{id}` (replace content, single file, multipart field name is `file` not `files`), `PATCH /api/files/{id}` (update details), `GET /api/files/{id}/preview`.

Identifier note: workspace-scoped routes take the **numeric workspace id** (int64), while `pacs-host` and `studies/initialize` take the **32-hex workspace UID**. They are different values; mixing them up produces 403s.

## Pipeline C: cloud documents (reports, notes, summaries)

A cloud document is Medicai's editable in-app document, the format specialists write reports in. The API creates one from Markdown, so a prototype can turn generated or typed text into a real document the clinician opens and edits inside Medicai:

```
POST {MEDICAI_API_HOST}/api/workspaces/{workspaceId}/files/cloud-documents
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json

{ "name": "Knee MRI report", "content": "# Findings\n\nNo acute ...", "caseId": 123 }
```

`content` is CommonMark plus `~~strikethrough~~`; plain text works too. `caseId` attaches the document to that case; omit it for a standalone drive document. Returns the drive row (`id`, `resourceType: "FILE"`, `name`, `url`, ...). Prefer this over uploading a generated PDF whenever the user wants the result to stay editable in Medicai.

To react when files or studies land (process a document, run analysis, notify someone), subscribe to workspace webhooks; see `webhooks.md`.

## Prototype data warning

What users upload is their responsibility under Medicai's terms, and trial workspaces are for synthetic or properly anonymized data. Point them at the demo studies in their workspace, public DICOM sample sets, or Medicai's anonymization (`POST /api/workspaces/{workspaceId}/cases/{caseId}/anonymize`, and `POST /api/workspaces/{workspaceId}/studies/modify-dicom` for tag-level changes) before anything identifiable enters a prototype. When they are ready for real patient data, that is the move to a paid plan with a BAA (US) or equivalent data-processing agreement (EU).
