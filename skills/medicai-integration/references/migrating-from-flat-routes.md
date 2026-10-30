# Migrating from Flat Routes

Read this only when the user already has Medicai code written before October 2026 and it calls routes without `/api/workspaces/{workspaceId}` in the path. New code never needs this file: every route it should use is in the other references, already in the workspace-scoped form.

## What changed and when

Older routes resolved the workspace from the caller's session. Since October 2026 the workspace is read from the URL instead, and the old routes are on a removal schedule:

| Date | What happens |
|---|---|
| Now | Old routes still answer, scoped to the user's current workspace, and every response carries a `Deprecation` header (dated 2027-02-01) and `Sunset: Tue, 01 Jun 2027` |
| 2027-06-01 | Old routes are removed |

`X-Medicai-Client` is mandatory on every call; older code that never sent it is rejected regardless of route shape. Add the header first, then migrate the routes.

## How to find the calls to migrate

Log response headers for one run of the integration and grep for `Deprecation` or `Sunset`. Every request that produced one of them is on the list below. The numeric `workspaceId` comes from `GET /api/workspaces` (`id` field; see `authentication.md`, "Workspace ids and route shape").

## Route map

| Old route | Replacement |
|---|---|
| `GET /api/cases` | `GET /api/workspaces/{workspaceId}/cases` |
| `POST /api/cases` | `POST /api/workspaces/{workspaceId}/cases` |
| `POST /api/cases/{caseId}/resources` (also `/delete`, `/anonymize`, `/case-management/...`) | `POST /api/workspaces/{workspaceId}/cases/{caseId}/...` |
| `PATCH /api/cases/move-to-trash`, `PATCH /api/cases/restore`, `POST /api/cases/delete` | `/api/workspaces/{workspaceId}/cases/move-to-trash`, `/restore`, `/delete` |
| `GET /api/resources/my-drive`, `GET /api/resources/shared` | `GET /api/workspaces/{workspaceId}/resources/my-drive`, `/shared` |
| `POST /api/files`, `POST /api/files/clone` | `POST /api/workspaces/{workspaceId}/files`, `/files/clone` |
| `GET /api/studies/metadata/search-study` | `GET /api/workspaces/{workspaceId}/studies/metadata/search-study` |
| `POST /api/studies/jobs`, `POST /api/studies/modify-dicom` | `POST /api/workspaces/{workspaceId}/studies/jobs`, `/modify-dicom` |
| `GET /api/labels`, `PATCH /api/labels/add`, `PATCH /api/labels/remove` | `/api/workspaces/{workspaceId}/labels...` |

Routes that address one resource by its own id (`PATCH /api/cases/{caseId}`, `GET /api/files/{id}`, `POST /api/studies/initialize`, link-share routes) did not change.

## Request and response shapes that changed with the routes

Moving the path is not always enough. Differences between the pre-October and October 2026 API exports that break naive migrations:

- Case listing requires `searchInTrash` (`?page=0&size=50&searchInTrash=false`); omitting it is a 400.
- Case listing responses no longer carry `pageable` and `sort` objects and now include facet counts; code that read `page.pageable.pageNumber` needs `page.number`.
- The `following` filter on case listing is gone.
- Trash and delete take `withResources` to say whether the case's studies and files go with it; the old calls had no such choice.
- The old search filter schemas were replaced by `SearchFilters` and `SearchFacets`; compare the request body against the current Swagger (`https://app.medicai.io/documentation/swagger-ui/index.html`) rather than assuming field names carried over.

## Migration checklist

1. Send `X-Medicai-Client` from the HTTP client, not per call.
2. Read `workspaceId` (numeric) and `workspaceUid` (32 hex) once from `GET /api/workspaces` and keep both in configuration.
3. Replace each route from the map above; keep the resource-by-id routes as they are.
4. Re-run the integration and confirm no response carries `Deprecation` or `Sunset`.
5. Fix the shape changes listed above where the compiler or the 400s point.
