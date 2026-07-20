# Authentik

Authentik provides identity, login, registration, and OIDC integration for homelab services. It is deployed with Argo CD through a local Helm wrapper chart, while Authentik-specific objects such as brands, flows, groups, and OIDC applications are managed declaratively with blueprints.

## Source Paths

- Argo CD application: `applications/authentik.yaml`
- Wrapper chart: `infrastructure/authentik/Chart.yaml`
- Wrapper values: `infrastructure/authentik/values.yaml`
- Blueprint ConfigMap template: `infrastructure/authentik/templates/authentik-blueprints-configmap.yaml`
- Database resource: `infrastructure/authentik/templates/authentik-database.yaml`
- Database role resource: `infrastructure/authentik/templates/authentik-database-role.yaml`
- Authentik config ExternalSecret: `infrastructure/authentik/templates/authentik-config-external-secret.yaml`
- Database auth ExternalSecret: `infrastructure/authentik/templates/authentik-db-external-secret.yaml`
- Media PVC: `infrastructure/authentik/templates/authentik-media-pvc.yaml`
- Blueprints: `infrastructure/authentik/blueprints/`
- Branding assets: `infrastructure/authentik/assets/`
- Public route: `infrastructure/routing/authentik-route.yaml`
- Local route: `infrastructure/routing/authentik-route-local.yaml`

## Kubernetes Components

Argo CD deploys `applications/authentik.yaml` into the `authentik` namespace from the path `infrastructure/authentik`. The application uses automated sync, pruning, self-healing, namespace creation, and server-side apply.

The local chart is a wrapper around the upstream Authentik Helm chart:

```yaml
dependencies:
  - name: authentik
    version: 2026.5.4
    repository: https://charts.goauthentik.io
```

Values for the upstream chart must be nested under the top-level `authentik:` key in `infrastructure/authentik/values.yaml`. This is important because settings placed at the wrapper chart root will not reach the upstream subchart.

Main runtime components:

- `authentik-server`: serves the web UI, APIs, OAuth/OIDC endpoints, and login flows.
- `authentik-worker`: runs background jobs and imports mounted blueprints.
- `authentik-server` service: exposed internally as `ClusterIP`.
- `authentik-media` PVC: mounted at `/data` in both server and worker pods.
- ServiceMonitor: enabled for Prometheus scraping through the upstream chart.
- Envoy Gateway HTTPRoutes: expose Authentik internally and publicly.

Current routes:

- `auth.home.lab` routes directly to the internal Authentik server service.
- `auth.rubenspensky.com` routes through the public Gateway listener.

The public route sets forwarded headers so Authentik sees the original public HTTPS request even though Cloudflare terminates TLS before traffic reaches the cluster:

```yaml
X-Forwarded-Proto: https
X-Forwarded-Host: auth.rubenspensky.com
X-Forwarded-Port: "443"
```

Use HTTPS when testing public Authentik:

```text
https://auth.rubenspensky.com
```

Using plain HTTP for the public hostname can cause frontend API, CORS, or WebSocket issues because the public exposure model assumes Cloudflare TLS termination.

## Database And Secrets

Authentik does not deploy the PostgreSQL subchart:

```yaml
authentik:
  postgresql:
    enabled: false
```

Instead, it uses the shared CloudNativePG cluster:

```yaml
authentik:
  authentik:
    postgresql:
      host: homelab-postgres-rw.databases.svc.cluster.local
      name: authentik
      user: authentik
      port: 5432
```

Database resources managed by this chart:

- `DatabaseRole/authentik` in the `databases` namespace creates the PostgreSQL login role.
- `Database/homelab-postgres-authentik` creates the `authentik` database owned by the `authentik` role.
- Both use `retain` reclaim policies so deleting the Kubernetes objects does not automatically drop the database or role.

Secrets are synchronized with External Secrets Operator from AWS Parameter Store. The expected remote key is:

```text
/homelab/authentik
```

Required properties:

- `secretKey`: used as `AUTHENTIK_SECRET_KEY`.
- `databasePassword`: used for both Authentik's PostgreSQL password and the CloudNativePG database role password.

Generated Kubernetes secrets:

- `Secret/authentik-config` in namespace `authentik`.
- `Secret/authentik-db-auth` in namespace `databases`.

`authentik-config` is injected into Authentik pods with `envFrom` and provides:

```text
AUTHENTIK_SECRET_KEY
AUTHENTIK_POSTGRESQL__PASSWORD
```

`authentik-db-auth` is a `kubernetes.io/basic-auth` secret used by CloudNativePG to set the `authentik` database role password.

## Media Storage

The chart creates `PersistentVolumeClaim/authentik-media` when `mediaPersistence.enabled` is true. Current settings:

```yaml
mediaPersistence:
  enabled: true
  size: 2Gi
  accessModes:
    - ReadWriteOnce
```

Authentik is configured to use file storage at `/data`:

```yaml
AUTHENTIK_STORAGE__BACKEND: file
AUTHENTIK_STORAGE__FILE__PATH: /data
```

Both server and worker mount the PVC at `/data`. Because the volume is `ReadWriteOnce`, the worker has required pod affinity to schedule on the same node as the server pod. This allows both pods to use the same media volume without requiring a shared RWX storage backend.

The media volume is intended for Authentik-managed files such as uploaded branding assets. The current brand blueprint references these repository assets through GitHub raw URLs:

- `infrastructure/authentik/assets/favicon.png`
- `infrastructure/authentik/assets/homelab-backrgound.png`

## Blueprint Loading

Blueprints are managed through Helm. The wrapper chart renders one ConfigMap:

```text
ConfigMap/authentik-blueprints
```

The ConfigMap is produced by:

```text
infrastructure/authentik/templates/authentik-blueprints-configmap.yaml
```

Current rendered keys:

```text
00-cleanup-legacy.yaml
10-demo-access-v3.yaml
20-demo-apps-v2.yaml
```

The upstream Authentik Helm chart mounts every ConfigMap listed under `authentik.blueprints.configMaps` into the worker pod:

```yaml
authentik:
  blueprints:
    configMaps:
      - authentik-blueprints
```

Only the worker imports blueprints. The server does not mount or apply them directly.

This nesting is required. A top-level sibling block like this will not work because it does not reach the upstream subchart:

```yaml
blueprints:
  configMaps:
    - authentik-blueprints
```

If a blueprint appears to have no effect, first verify that the ConfigMap name is listed under `authentik.blueprints.configMaps` and then check the worker logs.

## Why Blueprints Are Needed

Helm installs Authentik itself, but it does not fully describe the identity configuration inside Authentik. Blueprints fill that gap.

Blueprints are needed because these objects live inside Authentik's database, not as native Kubernetes resources:

- Brands and login styling.
- Authentication flows.
- Enrollment and registration flows.
- Prompt fields and stages.
- User write and user login stages.
- Groups.
- OIDC providers and applications.
- Policy bindings that restrict app access.

Managing these objects with blueprints keeps the identity configuration reproducible through GitOps. A new cluster or restored Authentik instance can converge toward the expected login experience, registration behavior, demo user group, and OIDC application setup without manually recreating objects in the UI.

Blueprints also let dependent objects reference each other with `!KeyOf` and existing Authentik defaults with `!Find`. This is important for flows because stages, bindings, brands, and applications must point to stable object IDs.

## Current Blueprints

Blueprint source files live in:

```text
infrastructure/authentik/blueprints/
```

The chart renders them into the single `authentik-blueprints` ConfigMap.

### Cleanup Legacy

Source file:

```text
infrastructure/authentik/blueprints/cleanup-legacy.yaml
```

Rendered key:

```text
00-cleanup-legacy.yaml
```

Purpose:

- Removes the early registration objects that used the old `user-registration` names.
- Prevents stale flows and stages from remaining in Authentik after the demo-specific flow names were introduced.
- Keeps the active setup focused on the `demo-*` flow names.

Objects removed with `state: absent`:

- Flow `user-registration`.
- Prompt `user-registration-field-email`.
- Prompt `user-registration-field-username`.
- Prompt `user-registration-field-password`.
- Prompt stage `user-registration-prompt-stage`.
- User write stage `user-registration-user-write-stage`.
- User login stage `user-registration-user-login-stage`.

This blueprint is intentionally kept as cleanup code because Authentik will not automatically delete old internal objects just because they disappeared from another blueprint.

### Demo Access

Source file:

```text
infrastructure/authentik/blueprints/demo-access.yaml
```

Rendered key:

```text
10-demo-access-v3.yaml
```

Purpose:

- Creates the demo user access model.
- Defines public self-registration.
- Defines the login flow used by the public Authentik brand.
- Applies the current login page branding and CSS.

Objects managed by this blueprint:

- Group `demo-users`.
- Enrollment flow `demo-user-registration`.
- Registration prompt fields for `email`, `username`, and `password`.
- Prompt stage `demo-user-registration-prompt-stage`.
- User write stage `demo-user-registration-user-write-stage`.
- User login stage `demo-user-registration-user-login-stage`.
- Authentication flow `demo-authentication`.
- Password stage `demo-authentication-password-stage`.
- Identification stage `demo-authentication-identification-stage`.
- User login stage `demo-authentication-user-login-stage`.
- Flow stage bindings for registration and authentication.
- Brand for `auth.rubenspensky.com`.

The `demo-users` group is the authorization boundary for demo applications. Users created through the registration flow are automatically added to this group by the user write stage.

The registration flow `demo-user-registration` is an enrollment flow with `authentication: require_unauthenticated`. It collects three required fields:

- `email`
- `username`
- `password`

The user write stage is configured with:

```yaml
user_creation_mode: always_create
create_users_as_inactive: false
create_users_group: !KeyOf demo-users-group
user_type: internal
```

This means self-registered users are created as active internal Authentik users and immediately assigned to `demo-users`.

The final registration stage logs the user in immediately after account creation. Without this stage, a user could register successfully but still need to manually authenticate afterward.

The authentication flow `demo-authentication` is the login flow selected by the public brand. It contains:

- An identification stage that allows login by email or username.
- A password stage using Authentik's built-in backend.
- A user login stage that completes the session.

The registration link is configured on the identification stage, not on the brand:

```yaml
enrollment_flow: !KeyOf demo-user-registration-flow
```

The public brand for `auth.rubenspensky.com` sets:

```yaml
branding_title: Rubenspensky Auth
branding_logo: https://raw.githubusercontent.com/rubenspensky-homelab/homelab-infra/main/infrastructure/authentik/assets/favicon.png
branding_favicon: https://raw.githubusercontent.com/rubenspensky-homelab/homelab-infra/main/infrastructure/authentik/assets/favicon.png
branding_default_flow_background: https://raw.githubusercontent.com/rubenspensky-homelab/homelab-infra/main/infrastructure/authentik/assets/homelab-backrgound.png
flow_authentication: !KeyOf demo-authentication-flow
```

The same brand also owns `branding_custom_css`. The CSS applies the current login page visual style:

- Dark monochrome theme.
- Grid background.
- Neo-brutalist rectangular login card.
- Monospace typography.
- Custom input, button, alert, link, avatar, and menu styles.
- Mobile-specific layout adjustments.
- Reduced-motion handling.
- Hiding the default powered-by footer.

The access blueprint intentionally keeps the group, flows, stages, stage bindings, and brand in one file. Authentik does not guarantee import order across separate mounted blueprint files, and these objects depend on each other through `!KeyOf` references.

### Demo Apps

Source file:

```text
infrastructure/authentik/blueprints/demo-apps.yaml
```

Rendered key:

```text
20-demo-apps-v2.yaml
```

Purpose:

- Creates OIDC integration for demo applications.
- Connects the OIDC provider to the demo authentication flow.
- Restricts the demo application to the `demo-users` group.

Current OIDC provider:

```text
frontend-demo-oidc-provider
```

Provider behavior:

- Uses `demo-authentication` as the authentication flow.
- Uses Authentik's default implicit-consent authorization flow.
- Uses Authentik's default provider invalidation flow.
- Uses Authentik's self-signed certificate keypair for signing.
- Uses public client type with client ID `frontend-demo`.
- Enables `authorization_code` and `refresh_token` grants.
- Includes claims in the ID token.
- Uses `user_uuid` as the subject mode.

Configured scopes:

- `openid`
- `email`
- `profile`

Configured redirect URIs:

- `https://frontend-demo.rubenspensky.com/auth/callback`
- `http://localhost:5173/auth/callback`
- `https://ts-backend-demo.rubenspensky.com/docs`
- `http://localhost:3000/docs`

Current Authentik application:

```text
Frontend Demo
```

Application details:

- Slug: `frontend-demo`
- Launch URL: `https://frontend-demo.rubenspensky.com`
- Provider: `frontend-demo-oidc-provider`
- Opens in a new tab.
- Description: `Frontend demo application for registered demo users.`

Access is restricted with a policy binding against the `demo-users` group. This makes registration meaningful: users who self-register through the demo flow are automatically placed into the group required to access the demo app.

OIDC values for frontend development:

```env
VITE_OIDC_AUTHORITY=https://auth.rubenspensky.com/application/o/frontend-demo/
VITE_OIDC_CLIENT_ID=frontend-demo
VITE_OIDC_REDIRECT_URI=http://localhost:5173/auth/callback
VITE_OIDC_POST_LOGOUT_REDIRECT_URI=http://localhost:5173/
VITE_OIDC_SCOPE=openid profile email
```

OIDC values for backend token validation:

```env
AUTH_PROVIDER=oidc
AUTH_ISSUER=https://auth.rubenspensky.com/
AUTH_JWKS_URL=https://auth.rubenspensky.com/application/o/frontend-demo/jwks/
AUTH_AUDIENCE=frontend-demo
```

The `frontend-demo` provider currently uses Authentik's global issuer mode. Discovery and JWKS endpoints are scoped to the application slug, but the JWT `iss` claim is the Authentik root URL:

```text
Discovery: https://auth.rubenspensky.com/application/o/frontend-demo/.well-known/openid-configuration
JWKS: https://auth.rubenspensky.com/application/o/frontend-demo/jwks/
Issuer claim: https://auth.rubenspensky.com/
Audience claim: frontend-demo
```

Backends must validate token signature, issuer, audience, and expiration.

Scalar/docs local OAuth values:

```env
DOCS_OAUTH_CLIENT_ID=frontend-demo
DOCS_OAUTH_AUTHORIZATION_URL=http://localhost:3000/docs/oauth/authorize
DOCS_OAUTH_UPSTREAM_AUTHORIZATION_URL=https://auth.rubenspensky.com/application/o/authorize/
DOCS_OAUTH_TOKEN_URL=https://auth.rubenspensky.com/application/o/token/
DOCS_OAUTH_REDIRECT_URI=http://localhost:3000/docs
```

Scalar/docs public OAuth redirect URI:

```env
DOCS_OAUTH_REDIRECT_URI=https://ts-backend-demo.rubenspensky.com/docs
```

## Runtime Flow

Expected public login and registration flow:

1. User opens `https://auth.rubenspensky.com`.
2. Authentik selects the brand for `auth.rubenspensky.com`.
3. The brand uses `demo-authentication` as its authentication flow.
4. The identification stage shows email or username login with password authentication.
5. The identification stage exposes the registration link through `enrollment_flow`.
6. User opens `demo-user-registration`.
7. User enters email, username, and password.
8. User write stage creates an active internal user.
9. User write stage adds the user to `demo-users`.
10. User login stage signs the user in immediately.
11. Demo applications can grant access based on the `demo-users` group policy binding.

## Forcing Blueprint Reimport

Authentik can keep a mounted blueprint instance with the same `(metadata.name, path)` and not immediately reapply later content-only changes, especially brand CSS changes.

To force a reimport while still updating the same Authentik objects, version the blueprint instance name and rendered ConfigMap key, but keep object identifiers unchanged.

Example blueprint metadata version:

```yaml
metadata:
  name: Homelab - Demo Access v3
```

Example rendered ConfigMap key version:

```yaml
data:
  10-demo-access-v3.yaml: |
```

Do not change object identifiers like flow slugs, stage names, group name, provider name, application slug, or brand domain unless you intentionally want new Authentik objects.

## Validation

Render locally:

```bash
helm lint infrastructure/authentik
helm template authentik infrastructure/authentik --namespace authentik
```

Check the rendered blueprint ConfigMap:

```bash
kubectl -n authentik get configmap authentik-blueprints -o yaml
```

Check mounted blueprint instances:

```bash
kubectl -n authentik exec deploy/authentik-worker -c worker -- ak shell -c "from authentik.blueprints.models import BlueprintInstance; print([(b.name, b.path, b.status, b.last_applied) for b in BlueprintInstance.objects.filter(path__contains='authentik-blueprints').order_by('path')])"
```

Check the active brand authentication flow:

```bash
kubectl -n authentik exec deploy/authentik-worker -c worker -- ak shell -c "from authentik.brands.models import Brand; print([(b.domain, getattr(b.flow_authentication, 'slug', None)) for b in Brand.objects.all()])"
```

Expected result for the public brand:

```text
('auth.rubenspensky.com', 'demo-authentication')
```

Check the brand CSS if changes are not visible:

```bash
kubectl -n authentik exec deploy/authentik-worker -c worker -- ak shell -c "from authentik.brands.models import Brand; b=Brand.objects.get(domain='auth.rubenspensky.com'); print('title', b.branding_title); print('has_css', bool(b.branding_custom_css)); print('has_theme_marker', 'Monochrome Neo-Brutalist' in b.branding_custom_css)"
```

Check worker logs for blueprint import issues:

```bash
kubectl -n authentik logs deploy/authentik-worker -c worker --since=30m | grep -Ei 'blueprint|demo-access|demo-apps|cleanup|error|failed|exception'
```

Restart after sync when testing blueprint or brand changes:

```bash
kubectl -n authentik rollout restart deployment/authentik-worker deployment/authentik-server
kubectl -n authentik rollout status deployment/authentik-worker
kubectl -n authentik rollout status deployment/authentik-server
```
