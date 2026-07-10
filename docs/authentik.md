# Authentik

## Source paths

- Application: `applications/authentik.yaml`
- Wrapper chart: `infrastructure/authentik/Chart.yaml`
- Wrapper values: `infrastructure/authentik/values.yaml`
- Branding blueprint ConfigMap: `infrastructure/authentik/templates/authentik-brand-blueprint-configmap.yaml`
- Media PVC: `infrastructure/authentik/templates/authentik-media-pvc.yaml`
- Branding assets: `infrastructure/authentik/assets/rubenspensky-logo.svg`, `infrastructure/authentik/assets/rubenspensky-icon.svg` (currently not applied by the active blueprint after a failed external-logo attempt)
- Routing: `infrastructure/routing/authentik-route.yaml`, `infrastructure/routing/authentik-route-local.yaml`

## Deployment model

Authentik is deployed through a local wrapper chart that depends on the upstream `authentik` Helm chart. Values intended for the upstream chart must be nested under the top-level `authentik:` key in `infrastructure/authentik/values.yaml`.

This matters for blueprint-based customization. The working configuration in this repository is:

```yaml
authentik:
  blueprints:
    configMaps:
      - authentik-brand-blueprints
```

A sibling top-level block such as the following does **not** reach the upstream subchart and will not be mounted by the Authentik worker:

```yaml
blueprints:
  configMaps:
    - authentik-brand-blueprints
```

## Media storage for file uploads

This repository now mounts `/data` for Authentik so the built-in file management UI can be used for branding assets.

Implementation details:

- `AUTHENTIK_STORAGE__BACKEND=file`
- `AUTHENTIK_STORAGE__FILE__PATH=/data`
- `authentik-media` PVC mounted into both the server and worker pods
- worker pod affinity is set to co-locate with the server pod so a single `ReadWriteOnce` volume can be mounted by both pods on the same node

This is intended to support Authentik media uploads such as logos and favicons through **Customization > Files**.

## Branding via blueprints

This repository manages Authentik branding declaratively with a blueprint stored in a Helm-rendered `ConfigMap`:

- `infrastructure/authentik/templates/authentik-brand-blueprint-configmap.yaml`

The blueprint currently provisions `authentik_brands.brand` entries for:

- `auth.home.lab`
- `auth.rubenspensky.com`

Current behavior:

- `auth.rubenspensky.com` is marked as the default brand.
- `auth.home.lab` keeps a separate non-default brand.
- The blueprint customizes `branding_title`, `branding_custom_css`, and now references uploaded media file names for `branding_logo` and `branding_favicon`.
- The current test values for logo and favicon are `rubenspensky-logo.svg` and `rubenspensky-icon.svg`, which correspond to files uploaded through Authentik's file management UI.

## Runtime behavior

The upstream chart discovers blueprint files from mounted `ConfigMap` and `Secret` sources. In this repository, enabling `authentik.blueprints.configMaps` causes the Authentik worker deployment to mount the named `ConfigMap`, after which Authentik can discover and apply `*.yaml` blueprint files from it.

Operationally, if a branding blueprint appears to have no effect, first verify that the blueprint source is configured under `authentik.blueprints` rather than at the wrapper chart root.
