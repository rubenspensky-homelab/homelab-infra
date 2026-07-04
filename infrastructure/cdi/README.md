# CDI

This folder vendors the official CDI release manifests so Argo CD can render and apply them deterministically.

Pinned version: `v1.65.0`

Upstream release: https://github.com/kubevirt/containerized-data-importer/releases/tag/v1.65.0

Vendored files:

- `cdi-operator.yaml`
- `cdi-cr.yaml`

To upgrade CDI, choose a new upstream release and replace both vendored manifests with the matching release assets:

```bash
CDI_VERSION=v1.65.0
curl -fsSL "https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml" -o infrastructure/cdi/cdi-operator.yaml
curl -fsSL "https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml" -o infrastructure/cdi/cdi-cr.yaml
```

After changing versions, update the version comments in `kustomization.yaml` and run:

```bash
kubectl kustomize infrastructure/cdi
```
