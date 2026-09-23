# Deploying ms-365-mcp-server to Google Cloud Run

This guide deploys the Microsoft 365 MCP server so that every Equancy user
connects with **their own** Microsoft account and gets **their own** rights.
The server holds no identity of its own and stores no tokens.

```
Claude Desktop  --- HTTPS ---> Cloud Run  --- Bearer ---> Microsoft Graph
      |                    (1 container,
      |                     stateless)
      +----- sign-in -----> Microsoft Entra ID
             (in the user's browser, once)
```

Two files :

| File | Role |
|------|------|
| `Dockerfile` | Node build, `ENTRYPOINT ["node","dist/index.js"]`, no `CMD` |
| `deploy-cloudrun.yaml` | The Cloud Run service: image, arguments, environment, scaling. |

---

## 0. Prerequisites

- A GCP project (one for staging, one for prod) and the `gcloud` CLI.
- An **Entra app registration**. One admin must create the app registration 
and grant tenant-wide consent.

```bash
gcloud auth login                      # not needed in Cloud Shell
gcloud config set project PROJECT_ID
```

---

## 1. App registration in Entra

### 1.1 Decide the scopes

The `--allowed-scopes` value in `deploy-cloudrun.yaml` is the security
boundary: any tool whose Graph permissions are not covered is hidden.

```bash
npm ci
npm run generate   # creates src/generated/client.ts (gitignored), needs internet
npm run build

# Full list of tools 
node dist/index.js --org-mode --list-permissions

# Narrow list, with fewer tools
node dist/index.js --org-mode \
  --allowed-scopes "User.Read Files.Read Sites.Selected" \
  --list-permissions
```

Read `effectivePermissions` (what to request) and `disabledTools` (what you
give up). Then put the value of `effectivePermissions` in the YAML.

### 1.2 Register the app

Azure Portal → Microsoft Entra ID → App registrations → New registration.
("Azure AD" and "Entra ID" are the same product; the portal still shows both names.)

| Setting | Value |
|---------|-------|
| Supported account types | **Accounts in this organizational directory only** (single tenant) |
| Redirect URI, platform **Web** | `https://claude.ai/api/mcp/auth_callback` |
| API permissions | Microsoft Graph → **Delegated** → the list from 1.1 |
| Certificates & secrets | create one client secret, copy the value now |

> **The redirect URI is the MCP client's callback**
> This server forwards the client's `redirect_uri` to Entra, so the
> authorization code is delivered straight to Claude Desktop and never
> reaches the server.

Then admin **grants tenant-wide admin consent**. This only removes
the per-user consent popup — it grants no administrative rights to anyone.
All permissions are *delegated*, so Microsoft Graph still applies each user's
own rights on every call.

To restrict who may use the server during a pilot: Enterprise Applications →
this app → Properties → **Assignment required = Yes**, then assign a group.

Note the **Application (client) ID** and the **Directory (tenant) ID**.

---

## 2. Enable the GCP APIs

```bash
gcloud services enable \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com \
  --project PROJECT_ID
```

---

## 3. Create the Artifact Registry repository

Skip if `mcp-servers` already exists in the project.

```bash
gcloud artifacts repositories create mcp-servers \
  --repository-format=docker \
  --location=REGION \
  --description="MCP Servers" \
  --project=PROJECT_ID
```

---

## 4. Store the client secret

Store the Entra's app's secret (Certificates & secrets > Clients secrets)

```bash
printf '%s' 'THE_CLIENT_SECRET' | \
  gcloud secrets create ms365-mcp-client-secret --data-file=- --project PROJECT_ID
```

Let the Cloud Run runtime service account read it:

```bash
PROJECT_NUMBER=$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')

gcloud secrets add-iam-policy-binding ms365-mcp-client-secret \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" \
  --project PROJECT_ID
```

To rotate later, add a new version — the YAML pins `key: latest`:

```bash
printf '%s' 'THE_NEW_SECRET' | \
  gcloud secrets versions add ms365-mcp-client-secret --data-file=-
```

---

## 5. Build and push the image


```bash
gcloud builds submit \
  --tag REGION-docker.pkg.dev/PROJECT_ID/mcp-servers/ms-365-mcp-server:latest \
  --project PROJECT_ID
```

> The Dockerfile runs `npm run generate`, which downloads Microsoft's live
> Graph OpenAPI spec at build time. Two builds of the same commit can
> therefore differ slightly.

---

## 6. Fill the placeholders and deploy

`deploy-cloudrun.yaml` is `deploy-cloudrun.yaml.example` without comments,
with `REGION` set to `europe-west1`. Replace every `TODO_` except
`TODO_CLOUD_RUN_SERVICE_URL`, plus `PROJECT_ID` in the image path. Read the
`.example` for the reason behind each setting.

```bash
gcloud run services replace deploy-cloudrun.yaml --region REGION --project PROJECT_ID
```

Read the URL back:

```bash
gcloud run services describe ms-365-mcp-server \
  --region REGION --project PROJECT_ID --format 'value(status.url)'
```

Put that URL in `MS365_MCP_PUBLIC_URL` and deploy again. The first deploy
exists only to learn the URL.

---

## 7. Allow unauthenticated traffic

Claude Desktop cannot present a Google IAM identity token, so Google-level
authentication must be open. **Entra is the gate**: every `/mcp` request
still requires a valid Microsoft bearer token, and returns `401` without one.

```bash
gcloud run services add-iam-policy-binding ms-365-mcp-server \
  --member="allUsers" --role="roles/run.invoker" \
  --region=REGION --project=PROJECT_ID
```

---

## 8. Verify

Three calls cover the whole surface.

```bash
URL=$(gcloud run services describe ms-365-mcp-server \
  --region REGION --project PROJECT_ID --format 'value(status.url)')

curl "$URL/"                                      # "Microsoft 365 MCP Server is running"
curl "$URL/.well-known/oauth-protected-resource"  # JSON, https:// origins
curl -X POST "$URL/mcp" -d '{}'                   # 401 + WWW-Authenticate
```

Then the full OAuth round trip:

```bash
npx @modelcontextprotocol/inspector
```

---

## 9. Connect a client

Claude Desktop → Settings → Connectors → **Add custom connector** →
`https://<service-url>/mcp`

A browser window opens, the user signs in with their Equancy account, and the
tools appear. Nothing to install, no shared token, no IT visit.

Config-file equivalent, if you push settings centrally:

```json
{
  "mcpServers": {
    "equancy-ms365": {
      "type": "streamable-http",
      "url": "https://<service-url>/mcp"
    }
  }
}
```

---

## Scaling: why `minScale` and `maxScale` are both `1`

During sign-in the server keeps the client's PKCE challenge in a
process-local `Map` between `GET /authorize` and `POST /token`.

Those two requests come from **two different programs** — `/authorize` from
the user's browser, `/token` from Claude Desktop — so Cloud Run's
cookie-based session affinity cannot route them to the same instance.

- `maxScale > 1` → the two legs can land on different instances
- `minScale: 0` → the instance can be reclaimed while the user is still
  typing their password

With`containerConcurrency: 80`, one instance is ample for an internal rollout.
Scaling out would require moving that map to Redis or Firestore — a small,
self-contained change to `src/server.ts`, worth doing only if one instance
ever becomes the bottleneck.

---

## Environment variables

Set in `deploy-cloudrun.yaml`. The full list is in the upstream `README.md`.

| Variable | Purpose |
|----------|---------|
| `MS365_MCP_CLIENT_ID` | Entra application (client) ID |
| `MS365_MCP_TENANT_ID` | Equancy tenant GUID. Never `common`. |
| `MS365_MCP_CLIENT_SECRET` | From Secret Manager |
| `MS365_MCP_PUBLIC_URL` | This service's public URL |
| `MS365_MCP_ALLOWED_REDIRECT_URIS` | Exact-match allowlist of client callbacks |
| `MS365_MCP_CORS_ORIGIN` | Allowed browser origin |

**Never set `MS365_MCP_TRUST_PROXY_AUTH`.** It skips the bearer check and
makes every user share one server-side identity, which destroys the per-user
rights model this deployment exists for. It is meant for proxies that
authenticate upstream, not for Cloud Run.

---

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| Startup probe fails, no useful logs | `--http 0.0.0.0:8000` missing from `args`. The server ignores `$PORT`. |
| `AADSTS50011` after signing in | The redirect URI registered in Entra is this service's URL. It must be the MCP client's callback. |
| `AADSTS501481` after signing in | More than one instance, or the instance restarted mid-sign-in. Check `minScale`/`maxScale`. |
| `AADSTS90009` | OBO mode only — the resource scope must be the GUID form, not `api://…`. Not applicable to this deployment. |
| Deploy step fails in Cloud Build | Cloud Build's service account needs `roles/run.admin` and `roles/iam.serviceAccountUser`. |
| Container starts, `/mcp` always 401 | Expected without a token. Test with the MCP Inspector, not `curl`. |
| `429 Too Many Requests` | Rate limits are per client IP: 120/min on `/mcp`, 30/min on `/authorize` and `/token`. Users behind one office NAT share a bucket. |

---

## Continuous deployment

The Cloud Build trigger that builds this branch and deploys to Cloud Run is
configured outside this repository. It needs:

- `roles/run.admin` and `roles/iam.serviceAccountUser` on the Cloud Build
  service account
- the image tag in `deploy-cloudrun.yaml` to match what the trigger pushes

Staging and production use the same file, but these values differ per project:

| Value | Why it differs |
|-------|----------------|
| `PROJECT_ID` in the image path | One Artifact Registry per project |
| `MS365_MCP_PUBLIC_URL` | Each project gets its own Cloud Run URL |
| `ms365-mcp-client-secret` | Secret Manager is per project: create it in both (step 4) |

---

## Security notes

- **Stateless**: no tokens are stored server-side. Each request carries the
  caller's own bearer token, held by Claude Desktop in the user's OS
  credential store.
- **Delegated permissions only**: this codebase has no client-credentials
  flow, so it can never act without a signed-in user.
- **Audit trail**: one JSON line per Graph call on stderr, captured by Cloud
  Logging, including `user_principal_name`, tool name and status.
- **Revocation**: disabling an account in Entra cuts MCP access at the next
  token refresh. Nothing to clean up on the GCP side.

---

## Improvements

| Idea | Why | Cost |
|------|-----|------|
| **On-Behalf-Of (`--obo`)** | Claude Desktop holds a token for *this app* only, never a Graph token; the server exchanges it per request | Entra: *Expose an API* → `access_as_user` scope. `--allowed-scopes` no longer filters discovery in OBO mode (see README) |
| Move the PKCE map to Firestore or Redis | Allows `maxScale > 1` and `minScale: 0` | Small change in `src/server.ts` |
| Deploy to staging first, then promote | Tests Graph spec drift (live `npm run generate`) before prod | A second trigger |
| Pin the image by commit (`:$SHORT_SHA`), not `:latest` | Rollback to an exact revision | Trigger + YAML tag |
| Alert on Cloud Build failure and client secret expiry | Expired secret = every sign-in fails (`AADSTS7000222`) | Cloud Monitoring alert + calendar reminder |
