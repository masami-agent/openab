# Microsoft Teams Enterprise Deployment

Deploy OpenAB with MS Teams in an enterprise Kubernetes environment. This guide covers Azure Entra ID configuration, Azure Bot Service setup, Teams app packaging, and Kubernetes deployment.

> **Unified Mode (available since v0.9.0-beta.4; validated with v0.9.0-beta.10):**
> The OAB binary embeds the Teams adapter, so no separate gateway container is
> needed. Complete the prerequisites and Steps 1–3, then follow
> [Step 4A](#step-4a-deploy-in-unified-mode-recommended). The stable `v0.9.0`
> release is not yet available; the commands below pin the validated beta.

## Prerequisites

- An Azure subscription with permissions to create resources
- A Microsoft 365 tenant with Teams enabled (Commercial Cloud Trial works for testing)
- A Kubernetes cluster with an Ingress controller and TLS
- `kubectl` CLI
- IT admin access to Teams Admin Center (for app approval)

## Step 1: Register an Azure Entra ID Application

1. Go to [Azure Portal → Microsoft Entra ID → App registrations](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade)
2. Click **New registration**
3. Configure:
   - **Name**: `openab-teams-bot` (or your preferred name)
   - **Supported account types**: **Single tenant** (Accounts in this organizational directory only)
   - **Redirect URI**: leave empty
4. Click **Register**

After creation, note from the **Overview** page:

| Value | Used As |
|---|---|
| Application (client) ID | `TEAMS_APP_ID` |
| Directory (tenant) ID | `<YOUR_TENANT_ID>` in OAuth endpoint |

### Create a Client Secret

1. Go to **Certificates & secrets** → **Client secrets** → **New client secret**
2. Set a description and expiration (recommended: 12 or 24 months)
3. Click **Add**
4. **Copy the Value immediately** — it is only shown once → `TEAMS_APP_SECRET`

> **Security note**: Store the client secret in a Kubernetes Secret. Never commit it to source control. Set a calendar reminder to rotate before expiration.

> **Note**: Multi-tenant bot creation was deprecated by Microsoft on July 31, 2025. Single Tenant is the only supported path for new bots.

## Step 2: Create an Azure Bot Resource

1. Go to [Azure Portal → Create a resource](https://portal.azure.com/#create/hub) → search **Azure Bot** → **Create**
2. Configure:
   - **Bot handle**: a unique name (e.g. `openab-prod`)
   - **Subscription / Resource group**: your enterprise subscription
   - **Pricing tier**: F0 (free) for testing, S1 for production
   - **Type of App**: **Single Tenant**
   - **Creation type**: **Use existing app registration**
   - **App ID**: paste `TEAMS_APP_ID` from Step 1
   - **App tenant ID**: paste your Directory (tenant) ID
3. Click **Review + Create** → **Create**

### Configure the Messaging Endpoint

1. Go to the Bot resource → **Configuration**
2. Set **Messaging endpoint** to your Kubernetes Ingress URL:
   ```
   https://<YOUR_INGRESS_HOST>/webhook/teams
   ```

### Enable the Teams Channel

1. Go to **Channels** → click **Microsoft Teams**
2. Accept the terms of service → **Save**

> **Testing tip**: After enabling the Teams channel, use the **Open in Teams** link (Azure Bot → Channels → Teams) for quick testing without uploading an app package. This link only works for people who have it — it does not make the bot discoverable org-wide.

> **⚠️ Do not use "Test in Web Chat"** for outbound reply testing. Azure Portal's Web Chat uses `webchat.botframework.com` which returns 403 for Single Tenant bot replies. Only real Teams clients (`smba.trafficmanager.net`) work for outbound.

## Step 3: Build a Teams App Manifest

Create a directory with three files:

### `manifest.json`

```json
{
  "$schema": "https://developer.microsoft.com/en-us/json-schemas/teams/v1.25/MicrosoftTeams.schema.json",
  "manifestVersion": "1.25",
  "version": "1.0.0",
  "id": "<GENERATE_A_UUID_V4>",
  "developer": {
    "name": "<YOUR_ORGANIZATION_NAME>",
    "websiteUrl": "https://<YOUR_COMPANY_WEBSITE>",
    "privacyUrl": "https://<YOUR_COMPANY_WEBSITE>/privacy",
    "termsOfUseUrl": "https://<YOUR_COMPANY_WEBSITE>/terms"
  },
  "name": {
    "short": "OpenAB",
    "full": "OpenAB AI Assistant"
  },
  "description": {
    "short": "AI coding assistant powered by OpenAB",
    "full": "Connect to an AI coding assistant through Microsoft Teams."
  },
  "icons": {
    "outline": "outline.png",
    "color": "color.png"
  },
  "accentColor": "#ffffff",
  "bots": [
    {
      "botId": "<YOUR_TEAMS_APP_ID>",
      "scopes": ["personal", "team", "groupChat"],
      "isNotificationOnly": false,
      "supportsFiles": false
    }
  ],
  "validDomains": []
}
```

- `id` — Teams app ID (generate a fresh UUID v4, not the same as `botId`)
- `botId` — Azure Entra ID Application (client) ID from Step 1

### Icons

- `outline.png` — 32×32 transparent background, white icon
- `color.png` — 192×192 full-color icon

### Package

```bash
zip openab-teams-app.zip manifest.json outline.png color.png
```

## Step 4A: Deploy in Unified Mode (Recommended)

Unified Mode routes Bot Framework webhooks directly to the OAB pod:

```
Teams Client → Bot Framework → K8s Ingress (HTTPS) → OAB Pod (:8080/webhook/teams)
```

### Create the Teams Secret

Create `openab-teams-secret.yaml` and provision the client secret through your
normal secret-management workflow. Do not commit the populated file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openab-teams
type: Opaque
stringData:
  TEAMS_APP_SECRET: "<YOUR_CLIENT_SECRET>"
```

For production, prefer an external secret controller or another mechanism that
keeps the secret value out of local files and shell history.

### Helm Values

Create `values.yaml`:

```yaml
agents:
  kiro:
    persistence:
      enabled: true
      size: 1Gi
    env:
      TEAMS_APP_ID: "<YOUR_APPLICATION_ID>"
      TEAMS_OAUTH_ENDPOINT: "https://login.microsoftonline.com/<YOUR_TENANT_ID>/oauth2/v2.0/token"
    secretEnv:
      - name: TEAMS_APP_SECRET
        secretName: openab-teams
        secretKey: TEAMS_APP_SECRET
    configToml: |
      [teams]
      app_id = "${TEAMS_APP_ID}"
      app_secret = "${TEAMS_APP_SECRET}"
      oauth_endpoint = "${TEAMS_OAUTH_ENDPOINT}"
      allowed_tenants = ["<YOUR_TENANT_ID>"]
      allowed_users = ["29:1abc..."]

      [agent]
      command = "kiro-cli"
      args = ["acp", "--trust-all-tools"]
      working_dir = "/home/agent"

      [pool]
      max_sessions = 10
      session_ttl_hours = 24
```

`secretEnv` injects the Secret into the OAB process so it can resolve the
`[teams]` configuration. Do **not** add any `TEAMS_*` keys to
`[agent].inherit_env`: the ACP agent does not need these adapter credentials,
and inheriting them would make the secret accessible to prompts and tools.

### User Trust and Tenant Scope

The recommended configuration restricts both the Azure AD tenant and individual
Bot Framework sender IDs. Find each user's `29:…` sender ID in OAB logs and add
it to `allowed_users`.

To allow every user in the configured tenant instead, replace `allowed_users`
with the explicit broad-access opt-in below. Keep `allowed_tenants`; otherwise,
activities from other tenants also pass the tenant gate.

```toml
[teams]
allowed_tenants = ["<YOUR_TENANT_ID>"]
allow_all_users = true
```

### Service and Ingress

Create `openab-teams-networking.yaml`. The selector must include the chart name,
Helm release, and agent component. This example assumes the release name
`openab`, used by the install command below; update the instance label if you
choose another release name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: openab-teams
spec:
  selector:
    app.kubernetes.io/name: openab
    app.kubernetes.io/instance: openab
    app.kubernetes.io/component: kiro
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openab-teams
spec:
  tls:
    - hosts:
        - <YOUR_INGRESS_HOST>
      secretName: <YOUR_TLS_SECRET>
  rules:
    - host: <YOUR_INGRESS_HOST>
      http:
        paths:
          - path: /webhook/teams
            pathType: Prefix
            backend:
              service:
                name: openab-teams
                port:
                  number: 8080
```

Apply the resources and install the validated chart version:

```bash
kubectl apply -f openab-teams-secret.yaml
helm upgrade --install openab oci://ghcr.io/openabdev/charts/openab \
  --version 0.9.0-beta.10 \
  -f values.yaml
kubectl apply -f openab-teams-networking.yaml
```

### Verify Unified Mode

1. Confirm exactly one pod matches the Service selector:
   ```bash
   kubectl get pods \
     -l app.kubernetes.io/name=openab,app.kubernetes.io/instance=openab,app.kubernetes.io/component=kiro
   ```
2. Check startup logs with `kubectl logs deployment/openab-kiro` and verify the
   Teams adapter is listening on `0.0.0.0:8080`.
3. Send a message from an allowed Teams user and confirm a reply. A user or
   tenant outside the configured allowlists must be rejected.

## Step 4B: Deploy in Standalone Gateway Mode (Legacy)

Use this mode only when you need the separate gateway architecture:

```
┌─────────────────────────────────────────────────────────────┐
│  Your Kubernetes Cluster                                    │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Ingress    │───▶│   Gateway    │◀──▶│     OAB      │  │
│  │  (HTTPS/TLS) │    │  (BYO deploy)│ WS │  (Helm chart)│  │
│  └──────┬───────┘    └──────────────┘    └──────────────┘  │
│         │                                                   │
└─────────┼───────────────────────────────────────────────────┘
          │ HTTPS
┌─────────┴───────────┐
│  Bot Framework      │
│  (Microsoft Cloud)  │
└─────────────────────┘
```

| Component | Deployed by | Description |
|---|---|---|
| **Gateway** | You (K8s Deployment or Docker) | Receives Bot Framework webhooks, validates JWT, routes replies. Reads `TEAMS_*` env vars. |
| **OAB** | Helm chart (`openab`) | Connects outbound to Gateway via WebSocket. No inbound ports needed. |

### Deploy the Gateway

The Gateway is deployed separately from the OAB Helm chart. Use a standard Kubernetes Deployment:

### Gateway Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openab-gateway-teams
type: Opaque
stringData:
  TEAMS_APP_ID: "<YOUR_APPLICATION_ID>"
  TEAMS_APP_SECRET: "<YOUR_CLIENT_SECRET>"
  TEAMS_OAUTH_ENDPOINT: "https://login.microsoftonline.com/<YOUR_TENANT_ID>/oauth2/v2.0/token"
  TEAMS_ALLOWED_TENANTS: "<YOUR_TENANT_ID>"
```

> **⚠️ Single Tenant bots must set `TEAMS_OAUTH_ENDPOINT`** to the tenant-specific endpoint. The default (`botframework.com`) only works for Multi Tenant bots and will cause `401 Unauthorized` errors. This is the #1 setup pitfall.

### Gateway Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openab-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openab-gateway
  template:
    metadata:
      labels:
        app: openab-gateway
    spec:
      containers:
        - name: gateway
          image: ghcr.io/openabdev/openab-gateway:0.9.0-beta.10
          ports:
            - containerPort: 8080
          envFrom:
            - secretRef:
                name: openab-gateway-teams
          env:
            - name: RUST_LOG
              value: "info"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: openab-gateway
spec:
  selector:
    app: openab-gateway
  ports:
    - port: 8080
      targetPort: 8080
```

### Ingress

Route Bot Framework webhooks to the Gateway using your existing Ingress controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openab-gateway
  annotations:
    # Adjust for your Ingress controller (nginx, ALB, Traefik, etc.)
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - <YOUR_INGRESS_HOST>
      secretName: <YOUR_TLS_SECRET>
  rules:
    - host: <YOUR_INGRESS_HOST>
      http:
        paths:
          - path: /webhook/teams
            pathType: Prefix
            backend:
              service:
                name: openab-gateway
                port:
                  number: 8080
```

> Bot Framework requires HTTPS. Your Ingress controller handles TLS termination — the Gateway pod listens on plain HTTP (:8080).

### Deploy OAB with Helm

OAB connects outbound to the Gateway via WebSocket. Create
`legacy-values.yaml` with the required raw `configToml` and an explicit user
allowlist:

```yaml
agents:
  kiro:
    configToml: |
      [gateway]
      url = "ws://openab-gateway:8080/ws"
      platform = "teams"
      allowed_users = ["29:1abc..."]
```

The chart mounts `configToml` verbatim. The `allowed_users` entry is required by
the default-deny identity trust gate; find each user's `29:…` sender ID in OAB
logs. To admit every user that passed the Gateway's tenant check, use the
explicit `allow_all_users = true` opt-in instead.

Install the same pinned version used by the Gateway image:

```bash
helm upgrade --install openab oci://ghcr.io/openabdev/charts/openab \
  --version 0.9.0-beta.10 \
  -f legacy-values.yaml
```

Do not set `agents.kiro.gateway.enabled` here: that option asks the chart to
deploy another Gateway, while this guide already created `openab-gateway`
separately. The OAB pod needs only the WebSocket URL and trust configuration;
it does not need the Teams client secret.

## Step 5: IT Admin — Approve the Teams App

Enterprise tenants typically restrict custom app installation. An IT admin must approve the app.

### Upload the App Package

1. Go to [Teams Admin Center](https://admin.teams.microsoft.com/) → **Teams apps** → **Manage apps**
2. Click **Upload new app** → select `openab-teams-app.zip`
3. The app appears with status **Blocked** (default for new custom apps)

### Configure Permission Policies

1. Go to **Teams apps** → **Permission policies**
2. Edit the **Global (Org-wide default)** policy or create a new one:
   - Under **Custom apps**, allow the OpenAB app
3. If using a custom policy, assign it to target users or groups

### Configure Setup Policies (Optional)

To pin the app for users automatically:

1. Go to **Teams apps** → **Setup policies**
2. Edit the relevant policy → **Installed apps** → **Add apps** → select OpenAB
3. Optionally add to **Pinned apps** for sidebar visibility

### Bot Discovery Methods

| Method | Who can find it | Best for |
|---|---|---|
| **Open in Teams link** | Only people with the link | Quick testing |
| **Teams Admin Center upload** | Everyone in the org | Enterprise deployment |
| **App Store publish** | Everyone worldwide | Commercial bots |

### Verify

After policy propagation (may take up to 24 hours):

1. Users go to **Apps** → **Built for your org** → find OpenAB → **Add**
2. For personal chat: open the app and start chatting
3. For channels: add the app to a team → use `@OpenAB` to mention the bot

## Tenant Allowlist

Unified Mode uses `[teams].allowed_tenants` as shown in Step 4A. In Standalone Gateway Mode, restrict which Azure AD tenants can interact with the bot by adding this value to the Gateway Secret:

```yaml
stringData:
  TEAMS_ALLOWED_TENANTS: "<YOUR_TENANT_ID>"
```

Multiple tenants: `"<TENANT_ID_1>,<TENANT_ID_2>"`. If not set, all tenants are allowed.

## Sovereign Cloud Configuration

Use the endpoints for your cloud in both deployment modes. In Unified Mode,
replace `TEAMS_OAUTH_ENDPOINT` in `agents.kiro.env` and add
`TEAMS_OPENID_METADATA` there; the `[teams]` section resolves both variables.
In Standalone Gateway Mode, add both values to the Gateway Secret.

| Cloud | `TEAMS_OAUTH_ENDPOINT` | `TEAMS_OPENID_METADATA` |
|---|---|---|
| Public (default) | `https://login.microsoftonline.com/<TENANT>/oauth2/v2.0/token` | `https://login.botframework.com/v1/.well-known/openidconfiguration` |
| Azure Government | `https://login.microsoftonline.us/<TENANT>/oauth2/v2.0/token` | `https://login.botframework.azure.us/v1/.well-known/openidconfiguration` |
| Azure China (21Vianet) | `https://login.partner.microsoftonline.cn/<TENANT>/oauth2/v2.0/token` | `https://login.botframework.azure.cn/v1/.well-known/openidconfiguration` |

## Environment Variables Reference (Gateway)

| Variable | Required | Default | Description |
|---|---|---|---|
| `TEAMS_APP_ID` | Yes | — | Azure Entra ID application (client) ID |
| `TEAMS_APP_SECRET` | Yes | — | Azure Entra ID client secret |
| `TEAMS_OAUTH_ENDPOINT` | Yes (Single Tenant) | `https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token` | Tenant-specific OAuth endpoint |
| `TEAMS_OPENID_METADATA` | No | `https://login.botframework.com/v1/.well-known/openidconfiguration` | OpenID metadata for JWT validation |
| `TEAMS_ALLOWED_TENANTS` | No | (allow all) | Comma-separated tenant IDs |
| `TEAMS_WEBHOOK_PATH` | No | `/webhook/teams` | Webhook endpoint path |

## Troubleshooting

### 401 Unauthorized when bot tries to reply

OAuth endpoint mismatch. Single Tenant bots must use the tenant-specific endpoint.

**Fix**: Verify `TEAMS_OAUTH_ENDPOINT` is set to `https://login.microsoftonline.com/<YOUR_TENANT_ID>/oauth2/v2.0/token`

### "Test in Web Chat" works but Teams doesn't reply

Web Chat uses Direct Line (`webchat.botframework.com`), which has different auth than Teams (`smba.trafficmanager.net`). Web Chat may accept inbound but reject outbound for Single Tenant bots.

**Fix**: Always test with a real Teams client. Do not rely on Web Chat for outbound reply testing.

### Bot doesn't appear in Teams

IT admin has not approved the custom app, or permission policy hasn't propagated.

**Fix**:
1. Verify the app is uploaded in Teams Admin Center → Manage apps
2. Check Permission policies allow the custom app
3. Wait up to 24 hours for policy propagation

### Standalone Gateway receives webhook but no reply in Teams

Check Gateway pod logs:
```bash
kubectl logs deployment/openab-gateway --tail=50
```

Look for: `teams → gateway` (received) → `gateway → teams` (sent) → `teams activity sent` (success) or `teams send error` (failure).

### JWT validation failed

The Teams adapter auto-refreshes JWKS on cache miss. If failures persist, verify
that the OpenID metadata endpoint configured for your cloud is reachable from
the component that receives the webhook:

```bash
# Unified Mode
kubectl exec deployment/openab-kiro -- \
  curl -s https://login.botframework.com/v1/.well-known/openidconfiguration

# Standalone Gateway Mode
kubectl exec deployment/openab-gateway -- \
  curl -s https://login.botframework.com/v1/.well-known/openidconfiguration
```

## Security Considerations

- **Credentials in Kubernetes Secrets** — never in ConfigMaps or Deployment manifests
- **Rotate client secrets** before expiration — set a reminder based on the expiration chosen in Step 1
- **Use a tenant allowlist** in production — configure `[teams].allowed_tenants` in Unified Mode or `TEAMS_ALLOWED_TENANTS` in Standalone Gateway Mode
- **Network policies** — consider restricting Gateway pod egress to Bot Framework endpoints
- **Minimize inbound exposure** — Unified Mode should expose only `/webhook/teams` through the TLS Ingress; in Standalone Gateway Mode, the OAB pod remains private and connects outbound to the Gateway only

## References

- [Azure Bot Service documentation](https://learn.microsoft.com/en-us/azure/bot-service/)
- [Register a bot with Azure](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration)
- [Teams app permission policies](https://learn.microsoft.com/en-us/microsoftteams/teams-app-permission-policies)
- [Teams custom app policies](https://learn.microsoft.com/en-us/microsoftteams/teams-custom-app-policies-and-settings)
- [Bot Framework authentication](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-authentication)
- [Teams app manifest schema](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
