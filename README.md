# Kubernetes Infrastructure GitOps

GitOps-managed Kubernetes infrastructure for a homelab platform.

This is the active infrastructure repository: [rubenspensky-homelab/infrastructure](https://github.com/rubenspensky-homelab/infrastructure).

This repository defines the cluster infrastructure components managed by Argo CD. Application workloads are delegated to the separate `homelab-apps` repository.

## Architecture

![Homelab Kubernetes architecture](./architecture.png)

## Cluster Hardware

| Node | Role | OS | Host | CPU | GPU | Memory | Root Disk |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `k8s-control-01` | Control plane node | Debian GNU/Linux 13 (trixie) x86_64 | Gigabyte Z97N-WIFI | Intel Core i5-4590, 4 cores, up to 3.70 GHz | NVIDIA GeForce GTX 1060 6GB; Intel integrated graphics | 15.46 GiB | 446.14 GiB ext4 |
| `k8s-worker-01` | Worker node | Debian GNU/Linux 13 (trixie) x86_64 | HP ProBook 440 G7 | Intel Core i5-10210U, 8 threads, up to 4.20 GHz | Intel UHD Graphics | 15.47 GiB | 237.49 GiB ext4 |
| `k8s-worker-02` | Worker node | Debian GNU/Linux 13 (trixie) x86_64 | Lenovo ThinkPad T480 | Intel Core i7-8650U, 8 threads, up to 4.20 GHz | NVIDIA GeForce MX150; Intel UHD Graphics 620 | 7.51 GiB | 118.27 GiB ext4 |

`k8s-control-01` runs Linux kernel `6.12.95+deb13-amd64`; `k8s-worker-01` and `k8s-worker-02` run `6.12.86+deb13-amd64`. Swap is disabled on all nodes.

## Current Components

| Category | Component | Purpose |
| --- | --- | --- |
| Applications | homelab-apps | Registers the external application repository in Argo CD |
| Applications | Umami | Provides privacy-focused web analytics backed by PostgreSQL |
| Backup | Velero | Provides cluster backup and restore with S3-compatible object storage |
| CI | GitHub Actions Runner Controller | Manages self-hosted GitHub Actions runners |
| CI | ARC runner scale set | Provides autoscaling GitHub Actions runners |
| CI | BuildKit | Provides a remote build service with persistent cache |
| Database | CloudNativePG | Manages the shared PostgreSQL cluster |
| GitOps | Argo CD root app | Bootstraps the app-of-apps deployment model |
| Hardware | NVIDIA Device Plugin | Advertises NVIDIA GPUs to Kubernetes workloads |
| Identity | Authentik | Provides identity and SSO for cluster services |
| Networking | MetalLB | Provides LoadBalancer IPs on the LAN |
| Networking | MetalLB config | Defines the LAN IP address pool and L2 advertisement |
| Networking | Envoy Gateway | Implements Gateway API for cluster routing |
| Observability | Prometheus Operator CRDs | Installs monitoring APIs before their consumers |
| Observability | kube-prometheus-stack | Provides Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter |
| Observability | Grafana Operator | Manages Grafana dashboards |
| Observability | Loki | Stores logs |
| Observability | Tempo | Stores traces |
| Observability | Alloy | Collects logs and traces |
| Observability | NVIDIA DCGM Exporter | Exposes NVIDIA GPU metrics to Prometheus |
| Routing | Gateway API routing | Defines the homelab gateway and HTTP routes |
| Routing | Cloudflare Tunnel | Provides public access through Cloudflare |
| Security | External Secrets Operator | Synchronizes Kubernetes secrets from AWS backends |
| Storage | local-path-provisioner | Provides the default local StorageClass |
| System | metrics-server | Provides the Kubernetes resource metrics API |

## Application Categories

Every Argo CD `Application` defined by this repository uses these labels:

```yaml
app.kubernetes.io/part-of: homelab-infrastructure
homelab.io/category: <category>
```

| Label value | Applications |
| --- | --- |
| `applications` | `homelab-apps`, `umami` |
| `backup` | `velero` |
| `ci` | `arc-controller-crds`, `arc-controller`, `arc-runner-set`, `buildkit` |
| `database` | `cloudnative-pg`, `postgres-clusters` |
| `gitops` | `homelab-root` |
| `hardware` | `nvidia-device-plugin` |
| `identity` | `authentik` |
| `networking` | `metallb`, `metallb-config`, `envoy-gateway` |
| `observability` | `prometheus-crds`, `prometheus-stack`, `grafana-operator`, `loki`, `tempo`, `alloy`, `dcgm-exporter` |
| `routing` | `routing`, `cloudflare-tunnel` |
| `security` | `external-secrets-crds`, `external-secrets`, `external-secrets-store` |
| `storage` | `local-path-provisioner` |
| `system` | `metrics-server` |

Use the category label in the Argo CD UI or query it directly:

```bash
kubectl get applications -n argocd \
  -l homelab.io/category=observability
```

## Current Routing And Exposure

The cluster uses Envoy Gateway and Gateway API for internal routing.

Current internal routes:

- `grafana.home.lab`
- `argocd.home.lab`
- `auth.home.lab`
- `umami.home.lab`

Current external routes:

- `auth.rubenspensky.com`
- `grafana.rubenspensky.com`
- `umami.rubenspensky.com`
- `telemetry.rubenspensky.com/collect`

Public exposure is handled through Cloudflare Tunnel. Cloudflare provides public TLS termination, so `cert-manager` is not currently required for the public access model.

`cert-manager` may be added later only if the platform needs internal HTTPS certificates, non-Cloudflare ingress TLS, or certificate automation inside the cluster.

## Repository Layout

| Path | Purpose |
| --- | --- |
| `bootstrap/` | Initial Argo CD root application |
| `applications/` | Argo CD `Application` resources for infrastructure components |
| `infrastructure/` | Helm wrapper charts, values, and Kubernetes manifests for each component |
| `docs/` | Operational notes and component-specific documentation |

## External Secrets Operator

External Secrets Operator synchronizes Kubernetes infrastructure secrets from AWS Secrets Manager. Installation order, AWS credential requirements, manual refresh behavior, and current infrastructure secret mappings are documented in [docs/external-secrets.md](./docs/external-secrets.md).

## PostgreSQL

CloudNativePG and the shared PostgreSQL cluster model are documented in [docs/postgresql.md](./docs/postgresql.md).

## Authentik

Authentik deployment structure and the current blueprint-based branding approach are documented in [docs/authentik.md](./docs/authentik.md).

## GPU Support

NVIDIA GPU scheduling and metrics are documented in [docs/gpu.md](./docs/gpu.md). The cluster uses the NVIDIA Device Plugin and DCGM Exporter instead of the GPU Operator to keep GPU support simpler and more predictable for this homelab.
