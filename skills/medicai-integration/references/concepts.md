# Medicai Concepts

The mental model behind every endpoint. Five minutes here saves an hour of 403s.

## Workspace

The tenant: a clinic, hospital, or organization. Every other entity lives inside one, and every access decision starts from workspace membership. An integration is bound to one workspace (the trial creates one for the account).

Two identifiers exist and the app needs both:

| Identifier | Type | Used by |
|---|---|---|
| `workspaceId` | number | every `/api/workspaces/{workspaceId}/...` route, the embedded viewer |
| `workspaceUid` (`uniqueIdentifier`) | 32 hex chars | PACS routes: `pacs-host`, `studies/initialize` |

Both come from `GET /api/workspaces` (the workspaces the token's user belongs to). Both are stable; cache them. The workspace is always read from the URL, never from session state, so there is no "current workspace" to rely on.

## Study

A DICOM study: one imaging exam (a knee MRI, a chest CT) made of series and instances, stored in the workspace's cloud PACS. A study has three ids and they are not interchangeable:

- the **numeric `id`** Medicai assigns (what the viewer, cases, and search results use);
- the **Study Instance UID** from the DICOM header (`studyInstanceUid`, useful for matching against a RIS or modality, useless for the embed viewer);
- the **PACS id** (`orthancId`, returned by uploads as `ParentStudy`, consumed by `studies/initialize`).

Studies are found by DICOM tag search, listed in the workspace drive (`resourceType: "STUDY"`), and displayed in the embedded viewer. Anonymized copies and tag edits run as background jobs; Medicai allows one running job per study.

## File and cloud document

Everything that is not DICOM. A **file** (PDF report, referral letter, JPEG, video) is uploaded through the files route and is generally read-only afterwards. A **cloud document** is Medicai's editable in-app document, the format specialists write reports in; the API creates one from Markdown, optionally attached to a case. Archives (zip, rar) are accepted nowhere: unpack, then upload DICOM to the PACS and other files through the files route.

Studies and files together are **resources**; the drive listing returns both with a `resourceType`, and a case holds both as `{studyIds, fileIds}`.

## Case

The collaboration unit: a bucket that groups one or more studies with documents, comments, reports, labels and a crew. Cases are what gets shared, assigned, prioritized, trashed and restored. When the user says "the patient's results", "the referral", or "send this to Dr. X", they mean a case. A case belongs to the workspace it was created in; it can be cloned into another workspace and merged with duplicates.

## People: members, crew, accounts

- **Workspace members** can see the workspace drive according to their role.
- **Case crew** are the people with access to one case, inside or outside the workspace. Adding someone to the crew (by Medicai user id or email, as `DOCTOR` or `PATIENT`) is the direct-grant mechanism.
- **Accounts** come in two types: doctor accounts (the workspace's professionals) and patient accounts (free, self-created, see what was shared with them).
- **Integration users** are Enterprise-only programmatic identities an integrator creates for its own end users so each one sees only their data. They are the right answer when "my app's users" are not Medicai users; see `plans-and-limits.md`.

## Sharing model

Two mechanisms, everywhere:

1. **Membership** (workspace members, case crew): the person has a Medicai account and sees the resource inside Medicai.
2. **Share links**: a uid with an access scheme. Case links open a join flow; study links open a viewer page; `public-study/{uid}` is the bare viewer for embedding. Access types are `PUBLIC`, `PRIVATE`, `GUEST_VIEW_PRIVATE_ACCESS`, `PASSWORD_PROTECTED` (with expiry and a password source such as the patient's birth date) and `DISABLED`.

Patient-facing flows combine a password-protected, expiring case link with patient-visibility controls (which resources the patient sees, whether they can comment). Default generated code to the narrowest option that does the job; a medical share link is never public by default.

## Labels, case management, worklist

Workspaces define labels (status, priority, custom) that cases carry; bulk status/priority routes exist for operations teams. A worklist and case requests model inbound orders (an exam requested, images expected). Most prototypes do not need these; they matter when the app is a clinic's operating tool rather than a viewer or portal.

## Webhooks and events

Every workspace can emit events (study created, case created, document added, status changed) to an HTTPS endpoint, HMAC-signed, identifiers only. Fetch details with the API afterwards. Use them instead of polling.

## Where it all sits

```
Workspace (id + uid)
├── Members (doctor accounts, roles)
├── Drive: resources = Studies (PACS) + Files / Cloud documents
├── Cases (title, resources, crew, labels, comments, share link)
├── Labels, worklist, case requests
└── Webhook subscriptions
```
