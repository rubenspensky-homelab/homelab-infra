# Authentik

This directory is a Helm wrapper chart for Authentik. Argo CD deploys it from `applications/authentik.yaml` using the path `infrastructure/authentik`.

## Blueprint Loading

Blueprints are managed through Helm, not Kustomize.

The wrapper chart renders one ConfigMap:

```text
ConfigMap/authentik-blueprints
```

The template is:

```text
templates/authentik-blueprints-configmap.yaml
```

The Authentik Helm chart mounts every ConfigMap listed in `values.yaml` under `authentik.blueprints.configMaps` into the `authentik-worker` pod:

```yaml
authentik:
  blueprints:
    configMaps:
      - authentik-blueprints
```

Only the worker imports blueprints. The server does not mount them directly.

## Current Blueprints

Blueprint source files live in:

```text
blueprints/
```

Current rendered ConfigMap keys:

```text
00-cleanup-legacy.yaml
10-demo-access-v2.yaml
20-demo-apps.yaml
```

`00-cleanup-legacy.yaml` removes early registration objects that used the old `user-registration` names.

`10-demo-access-v2.yaml` is generated from `blueprints/demo-access.yaml` and owns the demo user access stack:

- group `demo-users`
- enrollment flow `demo-user-registration`
- prompt fields `email`, `username`, `password`
- prompt stage
- user write stage
- user login stage
- authentication flow `demo-authentication`
- password stage
- identification stage
- flow stage bindings
- brand selection for `auth.rubenspensky.com`

`20-demo-apps.yaml` is generated from `blueprints/demo-apps.yaml` and owns demo OIDC applications.

Current OIDC applications:

```text
frontend-demo
```

`frontend-demo` uses:

```text
Client ID: frontend-demo
Client type: public
Issuer: https://auth.rubenspensky.com/application/o/frontend-demo/
Launch URL: https://frontend-demo.rubenspensky.com
Redirect URIs:
- https://frontend-demo.rubenspensky.com/auth/callback
- http://localhost:5173/auth/callback
- https://ts-backend-demo.rubenspensky.com/docs
- http://localhost:3000/docs
```

The application has a group policy binding for `demo-users`.

Frontend development values:

```env
VITE_OIDC_AUTHORITY=https://auth.rubenspensky.com/application/o/frontend-demo/
VITE_OIDC_CLIENT_ID=frontend-demo
VITE_OIDC_REDIRECT_URI=http://localhost:5173/auth/callback
VITE_OIDC_POST_LOGOUT_REDIRECT_URI=http://localhost:5173/
VITE_OIDC_SCOPE=openid profile email
```

Backend API validation values:

```env
AUTH_PROVIDER=oidc
AUTH_ISSUER=https://auth.rubenspensky.com/
AUTH_JWKS_URL=https://auth.rubenspensky.com/application/o/frontend-demo/jwks/
AUTH_AUDIENCE=frontend-demo
```

The `frontend-demo` provider currently uses Authentik's global issuer mode. The
OIDC discovery and JWKS endpoints are still scoped to the application slug, but
the JWT `iss` claim is the Authentik root URL:

```text
Discovery: https://auth.rubenspensky.com/application/o/frontend-demo/.well-known/openid-configuration
JWKS: https://auth.rubenspensky.com/application/o/frontend-demo/jwks/
Issuer claim: https://auth.rubenspensky.com/
Audience claim: frontend-demo
```

Backends must validate the token signature, issuer, audience, and expiration.

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

The demo access objects are intentionally in one blueprint file because Authentik does not guarantee import order between separate mounted blueprint files. Keeping these dependent objects together allows stable `!KeyOf` references.

## Demo Registration Flow

Expected user flow:

1. User opens `https://auth.rubenspensky.com`.
2. Authentik redirects to `/if/flow/demo-authentication/?next=%2F`.
3. The Identification Stage shows email/username and password login.
4. The Identification Stage shows the registration link through `enrollment_flow`.
5. User opens `demo-user-registration`.
6. User enters email, username, and password.
7. User Write Stage creates the user.
8. User Write Stage adds the user to `demo-users`.
9. User Login Stage signs the user in immediately.

Demo applications can later restrict access using the `demo-users` group.

## Brand And Login Flow

The brand for `auth.rubenspensky.com` sets:

```yaml
flow_authentication: !KeyOf demo-authentication-flow
```

The registration link is not configured on the brand. In Authentik 2026.5, it is configured on the Identification Stage:

```yaml
enrollment_flow: !KeyOf demo-user-registration-flow
```

## Forcing Blueprint Reimport

Authentik can keep a mounted blueprint instance with the same `(metadata.name, path)` and not immediately reapply later content-only changes, especially brand CSS changes.

To force a reimport while still updating the same Authentik objects, version the blueprint instance name and rendered ConfigMap key, but keep object identifiers unchanged.

Example:

```yaml
metadata:
  name: Homelab - Demo Access v2
```

```yaml
data:
  10-demo-access-v2.yaml: |
```

Do not change object identifiers like flow slugs, stage names, group name, or brand domain unless you intentionally want new Authentik objects.

## Validation

Render locally:

```bash
helm lint infrastructure/authentik
helm template authentik infrastructure/authentik --namespace authentik
```

Check the ConfigMap:

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
kubectl -n authentik exec deploy/authentik-worker -c worker -- ak shell -c "from authentik.brands.models import Brand; b=Brand.objects.get(domain='auth.rubenspensky.com'); print('content_empty', 'content: \"\"' in b.branding_custom_css); print('language_new', 'rgba(5, 5, 5, 0.72)' in b.branding_custom_css)"
```

Check logs:

```bash
kubectl -n authentik logs deploy/authentik-worker -c worker --since=30m | grep -Ei 'blueprint|demo-access|error|failed|exception'
```

Restart after sync when testing blueprint or brand changes:

```bash
kubectl -n authentik rollout restart deployment/authentik-worker deployment/authentik-server
kubectl -n authentik rollout status deployment/authentik-worker
kubectl -n authentik rollout status deployment/authentik-server
```

## HTTPS Requirement

Use HTTPS when testing public Authentik:

```text
https://auth.rubenspensky.com
```

Using `http://auth.rubenspensky.com` can cause frontend API/CORS/WebSocket issues because the public route sets forwarded headers for HTTPS.
