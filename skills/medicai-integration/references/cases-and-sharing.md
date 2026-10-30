# Cases and Sharing

Cases are Medicai's collaboration unit: a bundle of studies and files with a crew (the people who can see it), comments, and share links. Use cases when the user's app needs "send this scan to a colleague", "give the patient access", or "group this patient's imaging".

## Create a case

```
POST {MEDICAI_API_HOST}/api/workspaces/{workspaceId}/cases
Authorization: Bearer <ACCESS_TOKEN>
X-Medicai-Client: <integration-name>
Content-Type: application/json

{
  "title": "Knee MRI review - J. Doe",
  "resources": {
    "studyIds": [ 123 ],
    "fileIds": [ 456 ]
  },
  "crew": [
    { "userId": 0, "email": "colleague@clinic.ro", "forUserType": "DOCTOR" }
  ]
}
```

`resources` is required (it may be empty) and takes **numeric ids**: `studyIds` for studies registered via `studies/initialize`, `fileIds` for files uploaded via `/api/workspaces/{workspaceId}/files`. Crew members (`forUserType`: `DOCTOR` or `PATIENT`) get access to the case resources. Title max 320 chars. Responds `201 Created` with the request body plus the new `id`. Always use the workspace-scoped form with the numeric workspace id.

Listing: `GET /api/workspaces/{workspaceId}/cases?page=0&size=50&searchInTrash=false` plus optional `searchValue` (free text). `searchInTrash` is required; `true` searches trashed cases instead of active ones. The response is a page: `{content: [case], totalElements, totalPages, number, size, first, last}` (the October 2026 API dropped the old `pageable`/`sort` fields and the `following` filter). Patient details on a case: `PATCH /api/cases/{caseId}` with `{patientName, sex, description, ssn, birthDate}` plus optional `patientEmail`, `phone {countryCode, number}` (returns nothing; re-fetch). Study thumbnails for listings: `GET /api/studies/{studyId}/preview`.

Useful follow-ups:

| Action | Endpoint |
|---|---|
| Add studies/files later | `POST /api/workspaces/{workspaceId}/cases/{caseId}/resources` (body: same `studyIds`/`fileIds` shape) |
| Remove resources | `POST /api/workspaces/{workspaceId}/cases/{caseId}/resources/delete` |
| Add people | `POST /api/cases/{caseId}/add-crew` |
| Remove people | `POST /api/cases/{caseId}/remove-crew` |
| Update title | `PATCH /api/cases/{caseId}/title` |
| Comment on a case | `POST /api/cases/{caseId}/comments` (edit: `PATCH /api/cases/comments/{commentId}`) |
| Set status / priority label | `POST /api/workspaces/{workspaceId}/cases/{caseId}/case-management/{labelId}` and `.../case-management/priority/{labelId}` |
| List/search cases | `GET /api/workspaces/{workspaceId}/cases?page=0&size=50&searchInTrash=false` |
| Case detail | `GET /api/cases/{caseId}` |
| Copy to another workspace | `POST /api/cases/{caseId}/clone/{targetWorkspaceId}` |
| Attach a written report | `POST /api/workspaces/{workspaceId}/files/cloud-documents` with `caseId` (Markdown in, editable document out; see `upload-studies.md`) |
| Trash / restore | `PATCH /api/workspaces/{workspaceId}/cases/move-to-trash` and `.../restore` (body: array of case ids). Add `?withResources=true` to also trash the studies and files this workspace owns that only these cases use; restoring the case later does not restore them |
| Delete permanently | `POST /api/workspaces/{workspaceId}/cases/delete` body `{"caseIds": [..], "withResources": false}`. Irreversible; in generated code require an explicit confirmation step and default `withResources` to false |

## Share links (case-level)

Every case can have a share link with an access type:

```
GET   /api/cases/{caseId}/link-share        // current link info
PATCH /api/cases/{caseId}/link-share        // set access type + options
```

The PATCH body supports:

```json
{
  "type": "PASSWORD_PROTECTED",
  "expirationMinutes": 10080,
  "generatePassword": true,
  "passwordSource": "PATIENT_BIRTH_DATE"
}
```

Access types:

| Type | Meaning |
|---|---|
| `PUBLIC` | Anyone with the link can view |
| `PRIVATE` | Link requires being on the case |
| `GUEST_VIEW_PRIVATE_ACCESS` | Guests can view; full access stays private |
| `PASSWORD_PROTECTED` | Link plus a password (set one, or generate from `passwordSource`: RANDOM, PATIENT_ID, PATIENT_BIRTH_YEAR, PATIENT_BIRTH_DATE, PATIENT_INITIALS_BIRTH_YEAR) |
| `DISABLED` | Link off |

Recipients open the link and join via `POST /api/cases/{caseShareUid}/join`; password-protected links verify via `POST /api/cases/link-share/{caseShareUid}/verify-password`. Access requests: `POST /api/cases/access-request/{caseShareUid}`, answered by the owner with `POST /api/cases/access-request/access-response/{id}`.

## Patient-facing access

For "here are your results" flows, three controls work together:

| Control | Endpoint |
|---|---|
| Patient view of a case | `GET /api/cases/{caseId}/patient` |
| Choose which resources the patient sees | `POST /api/cases/{caseId}/resources/patient-visibility` |
| Allow/disallow patient comments | `PATCH /api/cases/{caseId}/patient-comments-enabled` |

The recommended patient pattern: a `PASSWORD_PROTECTED` share link with `passwordSource` set to something the patient knows (e.g. `PATIENT_BIRTH_DATE`) plus an expiry, and patient-visibility trimmed to what they should see.

## Share links (single study)

When the app shares one study rather than a case:

```
GET   /api/studies/{studyId}/link-share        // returns (or creates) the study share link
PATCH /api/studies/{studyId}/link-share        // type: PUBLIC | PRIVATE | PASSWORD_PROTECTED (+ expirationMinutes, password options)
PATCH /api/studies/{studyId}/link-share/reset  // rotate uid + expiry
```

Password verification: `POST /api/studies/link-share/{uid}/verify-password`. Temporary downloads: `POST /api/studies/{studyId}/temporary-download` creates a time-limited download token.

## Anonymization

`POST /api/workspaces/{workspaceId}/cases/{caseId}/anonymize` produces an anonymized copy of a case in that workspace (returns a job; poll it). Use it before any sharing that leaves the clinical circle (teaching, second opinions abroad, demos). For tag-level control over DICOM metadata use `POST /api/workspaces/{workspaceId}/studies/modify-dicom`. Medicai allows one running job per study; do not fire several modify jobs at the same study.

## Reacting to case activity

To have the app respond when something happens on a case (a study added, status changed, a document attached), use workspace webhooks instead of polling; see `webhooks.md`. The case activity feed is also queryable: `GET /api/cases/{caseId}/activity-events`.

## Design guidance for prototypes

- Default share links to `DISABLED` or `PRIVATE` in generated code. Make the user explicitly choose anything broader; never default a medical share link to public.
- For patient access, prefer `PASSWORD_PROTECTED` with an expiry over bare `PUBLIC` links, and trim patient-visibility to the resources that concern them.
- Surface the share URL and its access type together in the UI so the user always sees how exposed a case is.
- Remind the user: prototypes must run on synthetic or anonymized data until they have a paid plan and a BAA or equivalent agreement.
