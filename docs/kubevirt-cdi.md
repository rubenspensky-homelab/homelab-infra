# KubeVirt And CDI

KubeVirt extends Kubernetes with virtual machine APIs. It installs controllers, handlers, webhooks, CRDs, and the `KubeVirt` custom resource needed to run VMs as Kubernetes-managed workloads.

CDI, the Containerized Data Importer, provides data import workflows for VM disks. It is used to import disk images from HTTP, registries, uploads, and other sources into PVCs that KubeVirt VMs can consume.

This repository installs only Kubernetes manifests through Argo CD. It does not install host packages or configure kernel modules on the nodes.

## Versions

Pinned upstream release manifests:

- KubeVirt `v1.8.4` from the official `kubevirt/kubevirt` GitHub release assets.
- CDI `v1.65.0` from the official `kubevirt/containerized-data-importer` GitHub release assets.

These versions are pinned in:

- `infrastructure/kubevirt/kustomization.yaml`
- `infrastructure/cdi/kustomization.yaml`

KubeVirt is reconciled before CDI. Within each component, the operator manifest is applied before the custom resource using Argo CD sync waves.

## Node Requirements

The hosts already expose `/dev/kvm` and already load `kvm_intel` and `kvm`. Validate node virtualization support directly on each node with:

```bash
ls -l /dev/kvm
lsmod | grep kvm
egrep -c '(vmx|svm)' /proc/cpuinfo
```

Expected results:

- `/dev/kvm` exists.
- `kvm_intel` and `kvm` are loaded.
- CPU virtualization flags are present.

## Validate Installation

After Argo CD syncs the applications, validate KubeVirt and CDI with:

```bash
kubectl get pods -n kubevirt
kubectl get kubevirt -n kubevirt
kubectl get pods -n cdi
kubectl get cdi -n cdi
kubectl get crds | grep kubevirt
kubectl get crds | grep cdi
```

Useful inspection commands:

```bash
kubectl describe kubevirt kubevirt -n kubevirt
kubectl describe cdi cdi -n cdi
kubectl get datavolumes -A
kubectl get virtualmachines -A
kubectl get virtualmachineinstances -A
```

No `VirtualMachine` resources are created by this repository.
