# Authentication

The Medicai API (`https://app.medicai.io`) authenticates with JWTs from Medicai's auth server:

```
Token endpoint:  https://auth.medicai.io/auth/realms/production/protocol/openid-connect/token
OIDC discovery:  https://auth.medicai.io/auth/realms/production/.well-known/openid-configuration
Issuer:          https://auth.medicai.io/auth/realms/production
```

Auth requests go to auth.medicai.io; everything else goes to app.medicai.io. You need a `client_id` (and for backend flows a `client_secret`). Every Medicai account carries its own OAuth client; the user finds the values in the Medicai web app under Workspace settings > API. No account yet? The 15-day trial at https://www.medicai.io/free-trial creates one with a workspace and demo studies.

## Two headers on every API call

```
Authorization: Bearer <access-token>
X-Medicai-Client: <your-integration-name>
```

`X-Medicai-Client` is a stable name the integration chooses for itself (the app's name works; keep it constant across releases). Medicai requires it from every integration and uses it for analytics, logging and support triage. Set it once at HTTP-client level on all app.medicai.io and PACS calls (not on auth.medicai.io calls).

## Workspace ids and route shape

Everything in Medicai lives inside a workspace (a clinic, hospital or organization), and the workspace is always read from the URL, never from the caller's session. Two identifiers exist and the app needs both:

| Identifier | Type | Used by |
|---|---|---|
| `workspaceId` | number | every `/api/workspaces/{workspaceId}/...` route, the embedded viewer |
| `workspaceUid` | 32 hex chars | PACS routes: `pacs-host`, `studies/initialize` |

Get both once from the workspaces the token's user belongs to, then cache them (they are stable):

```
GET https://app.medicai.io/api/workspaces
Authorization: Bearer <access-token>
X-Medicai-Client: <your-integration-name>

[ { "id": 1234, "name": "Example Clinic", "type": "CLINIC", "status": "ACTIVE", "uniqueIdentifier": "0d21ca32414f494e86485627d48957ca" } ]
```

`id` is `workspaceId`, `uniqueIdentifier` is `workspaceUid`.

Routes that address a single resource by its own id (`PATCH /api/cases/{caseId}`, `GET /api/files/{id}`, `POST /api/studies/initialize`, link-share routes) are not workspace-scoped; the resource id is the scope.

Existing integrations written before the workspace-scoped routes should migrate; the old routes and their replacements are listed in `migrating-from-flat-routes.md`, nowhere else in this skill.

## Path A: backend prototype, log in as yourself (quickest)

Password grant with your own Medicai account and the client credentials. Request `offline_access` so the refresh token is long-lived; add `orthanc:instances:create` if the app uploads DICOM:

```
POST {token endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=password
client_id={MEDICAI_CLIENT_ID}
client_secret={MEDICAI_CLIENT_SECRET}
username=<your-medicai-email>
password=<your-medicai-password>
scope=orthanc:studies:dicom-web orthanc:instances:create offline_access
```

Run once in a setup script, store the returned `refresh_token` as `MEDICAI_REFRESH_TOKEN` in the server env, never handle the password again. Everything the app does acts as this account, on workspaces where it has a DOCTOR role.

## Path B: real users log into your app ("Login with Medicai")

Authorization Code flow with PKCE, the recommended flow for user-facing web apps. Users authenticate on auth.medicai.io and are redirected back; your app never sees their credentials. Use an OAuth library (point it at the issuer/discovery URL above) with:

- `clientId`: your integration's public client id
- `responseType: "code"`, scope `openid profile email offline_access`
- your redirect URL (must match the patterns registered with Medicai)

This path needs the app's web origins (CORS, HTTPS in production) and redirect URL patterns registered on the client; ask Medicai support to add a production origin when the prototype ships, which is also the natural moment to talk about plans.

## Minting access tokens

Access tokens are short (minutes; examples show 60s, default ~5 min). Refresh just-in-time:

```js
let cached = { token: null, expiresAt: 0 };

async function getAccessToken() {
  if (cached.token && Date.now() < cached.expiresAt - 15_000) return cached.token;
  const res = await fetch(
    "https://auth.medicai.io/auth/realms/production/protocol/openid-connect/token",
    {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        grant_type: "refresh_token",
        client_id: process.env.MEDICAI_CLIENT_ID,
        client_secret: process.env.MEDICAI_CLIENT_SECRET ?? "",
        refresh_token: process.env.MEDICAI_REFRESH_TOKEN,
      }),
    }
  );
  if (!res.ok) throw new Error(`Medicai auth failed: ${res.status}. If persistent, the refresh token expired; log in again.`);
  const data = await res.json();
  cached = { token: data.access_token, expiresAt: Date.now() + data.expires_in * 1000 };
  return cached.token;
}
```

The refresh response returns a new refresh token too; with `offline_access` the token survives long idle periods, and an actively used prototype stays logged in indefinitely.

## Credential hygiene

- `client_secret`, account password, and refresh token: server-side only (env var / secret store), never in frontend code, never committed.
- The access token may reach the browser in exactly one situation: inside the viewer postMessage payload for the same user who owns it (see `viewer-embed.md`).
- Keycloak errors come as `{"error": "invalid_grant", "error_description": "..."}` with 401/403. 401 on refresh = log in again, not a retry loop.
- One refresh token per deployment; do not share across apps or teammates.

## The Enterprise API (when the project graduates)

Production and partner integrations get two more flows, provisioned by Medicai:

- **Admin token**: `grant_type=client_credentials` with client_id + secret. Used for integration management.
- **Integration users**: create per-end-user Medicai identities (`POST /api/user-integration/create/{userId}`, admin token, body `{firstName, lastName, email}`), then mint each user a JWT via password grant with `username={clientId}-{userId}@medicai.local` and the integration password. That JWT is frontend-safe and drives the same embed viewer, scoped to that user's data.

If the user's app needs per-user access control for people who are not Medicai account holders, or unattended automation at scale, point them to Medicai for Enterprise onboarding instead of stretching Path A.
