# Homelab Infrastructure

GitOps-managed Kubernetes infrastructure for a homelab platform.

This repository defines the cluster infrastructure components managed by Argo CD. Application workloads are delegated to the separate `homelab-apps` repository.

## Architecture

![Homelab Kubernetes architecture](./arch.jpg)

## Cluster Hardware

| Node | Role | OS | Host | CPU | GPU | Memory | Root Disk |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `homelab` | Control plane / main node | Debian GNU/Linux 13 (trixie) x86_64 | Z97N-WIFI | Intel Core i5-4590, 4 cores, up to 3.70 GHz | NVIDIA GeForce GTX 1060 6GB; Intel integrated graphics | 15.46 GiB | 422.51 GiB ext4 |
| `k8s-worker-01` | Worker node | Debian GNU/Linux 13 (trixie) x86_64 | HP ProBook 440 G7 | Intel Core i5-10210U, 8 threads, up to 4.20 GHz | Intel UHD Graphics | 15.47 GiB | 220.63 GiB ext4 |

Both nodes run Linux kernel `6.12.94+deb13-amd64` with swap disabled.

## Current Components

| Component | Purpose |
| --- | --- |
| Argo CD root app | Bootstraps the app-of-apps deployment model |
| homelab-apps | Registers the external application repository in Argo CD |
| MetalLB | Provides LoadBalancer IPs on the LAN |
| MetalLB config | Defines the LAN IP address pool and L2 advertisement |
| Envoy Gateway | Gateway API implementation for cluster routing |
| Gateway API routing | Defines the homelab gateway and HTTP routes |
| Cloudflare Tunnel | Provides public access through Cloudflare |
| local-path-provisioner | Default local storage provisioner |
| metrics-server | Kubernetes resource metrics API |
| kube-prometheus-stack | Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter |
| Loki | Log storage backend |
| Tempo | Trace storage backend |
| Alloy | Log and trace collector |
| GitHub Actions Runner Controller | Controller for self-hosted GitHub Actions runners |
| ARC runner scale set | Autoscaling runner set for GitHub Actions |
| BuildKit | Remote build service with persistent cache |
| NVIDIA Device Plugin | Advertises NVIDIA GPUs to Kubernetes workloads |
| KubeVirt | Kubernetes-native virtualization for running virtual machines |
| CDI | Containerized Data Importer for importing VM disk images into PVCs |
| External Secrets Operator | Synchronizes Kubernetes secrets from AWS Secrets Manager |

## Current Routing And Exposure

The cluster uses Envoy Gateway and Gateway API for internal routing.

Current internal routes:

- `grafana.home.lab`
- `argocd.home.lab`

Public exposure is handled through Cloudflare Tunnel. Cloudflare provides public TLS termination, so `cert-manager` is not currently required for the public access model.

`cert-manager` may be added later only if the platform needs internal HTTPS certificates, non-Cloudflare ingress TLS, or certificate automation inside the cluster.

## Repository Layout

| Path | Purpose |
| --- | --- |
| `bootstrap/` | Initial Argo CD root application |
| `applications/` | Argo CD `Application` resources for infrastructure components |
| `infrastructure/` | Helm wrapper charts, values, and Kubernetes manifests for each component |
| `docs/` | Operational notes and component-specific documentation |

## KubeVirt And CDI

KubeVirt and CDI installation details, node requirements, and validation commands are documented in [docs/kubevirt-cdi.md](./docs/kubevirt-cdi.md).

## External Secrets Operator

External Secrets Operator is installed with Helm through Argo CD. It uses a `ClusterSecretStore` named `aws-secrets-manager` configured for AWS Secrets Manager in `us-east-2`.

The AWS IAM credentials are intentionally not committed to Git. Create this Secret out-of-band before the `ClusterSecretStore` reconciles:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: external-secrets
type: Opaque
stringData:
  access-key-id: <aws-access-key-id>
  secret-access-key: <aws-secret-access-key>
```

Argo CD sync waves apply the resources in dependency order:

1. `external-secrets`: installs the operator and CRDs.
2. `external-secrets-store`: creates the AWS Secrets Manager `ClusterSecretStore`.
3. `infra-secrets`: creates infrastructure `ExternalSecret` resources that generate Kubernetes Secrets.

The infrastructure `ExternalSecret` resources use `refreshInterval: 0`, so they do not poll AWS Secrets Manager periodically after syncing. To force a refresh, update the `external-secrets.io/force-sync` annotation with a new value:

```bash
kubectl annotate externalsecret cloudflare-tunnel-token \
  -n cloudflare \
  external-secrets.io/force-sync=$(date +%s) \
  --overwrite
```

For a GitOps-driven refresh, change the same annotation value in Git and let Argo CD sync it.

The current infrastructure secret targets use alternate Kubernetes Secret names so they can be tested before replacing the manually created Secrets:

| ExternalSecret | Namespace | AWS Secrets Manager key | Generated Kubernetes Secret |
| --- | --- | --- | --- |
| `cloudflare-tunnel-token` | `cloudflare` | `homelab/infra/cloudflare-tunnel` | `tunnel-token-eso` |
| `arc-github-app` | `arc-systems` | `homelab/infra/arc-github-app` | `arc-github-app-eso` |

After validation, change the generated Secret names to `tunnel-token` and `arc-github-app`, then remove the manually created Secrets.

Expected AWS Secrets Manager JSON values:

```json
{
  "token": "<cloudflare-tunnel-token>"
}
```

```json
{
  "github_app_id": "<github-app-id>",
  "github_app_installation_id": "<github-app-installation-id>",
  "github_app_private_key": "<github-app-private-key>"
}
```

## Future Components

Only the following platform components are planned next.

### Velero With S3 Backups

Velero will be used for cluster backup and restore.

Expected setup:

- Store backups in S3
- Back up Kubernetes resources
- Add scheduled backups
- Define and test a restore process

## Bootstrap

High-level bootstrap flow:

1. Install Kubernetes.
2. Install Argo CD.
3. Apply `bootstrap/root-app.yaml`.
4. Let Argo CD reconcile the infrastructure applications from `applications/`.
