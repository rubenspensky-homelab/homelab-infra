# Authentik

## Source paths

- Application: `applications/authentik.yaml`
- Wrapper chart: `infrastructure/authentik/Chart.yaml`
- Wrapper values: `infrastructure/authentik/values.yaml`
- Branding blueprint ConfigMap: `infrastructure/authentik/templates/authentik-brand-blueprint-configmap.yaml`
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

## Branding via blueprints

This repository manages Authentik branding declaratively with a blueprint stored in a Helm-rendered `ConfigMap`:

- `infrastructure/authentik/templates/authentik-brand-blueprint-configmap.yaml`

The blueprint currently provisions `authentik_brands.brand` entries for:

- `auth.home.lab`
- `auth.rubenspensky.com`

Current behavior:

- `auth.rubenspensky.com` is marked as the default brand.
- `auth.home.lab` keeps a separate non-default brand.
- The active working blueprint customizes `branding_title` and `branding_custom_css`.
- Repository SVG assets exist for future branding work, but the external-logo attempt was removed after the mounted blueprint entered an error state during application.

## Runtime behavior

The upstream chart discovers blueprint files from mounted `ConfigMap` and `Secret` sources. In this repository, enabling `authentik.blueprints.configMaps` causes the Authentik worker deployment to mount the named `ConfigMap`, after which Authentik can discover and apply `*.yaml` blueprint files from it.

Operationally, if a branding blueprint appears to have no effect, first verify that the blueprint source is configured under `authentik.blueprints` rather than at the wrapper chart root.
