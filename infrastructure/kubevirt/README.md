# KubeVirt

This folder vendors the official KubeVirt release manifests so Argo CD can render and apply them deterministically.

Pinned version: `v1.8.4`

Upstream release: https://github.com/kubevirt/kubevirt/releases/tag/v1.8.4

Vendored files:

- `kubevirt-operator.yaml`
- `kubevirt-cr.yaml`

To upgrade KubeVirt, choose a new upstream release and replace both vendored manifests with the matching release assets:

```bash
KUBEVIRT_VERSION=v1.8.4
curl -fsSL "https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml" -o infrastructure/kubevirt/kubevirt-operator.yaml
curl -fsSL "https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml" -o infrastructure/kubevirt/kubevirt-cr.yaml
```

After changing versions, update the version comments in `kustomization.yaml` and run:

```bash
kubectl kustomize infrastructure/kubevirt
```
