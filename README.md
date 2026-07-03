# Homelab Infrastructure

GitOps-managed Kubernetes infrastructure for a homelab platform.

This repository defines the cluster infrastructure components managed by Argo CD. Application workloads are delegated to the separate `homelab-apps` repository.

## Architecture

![Homelab Kubernetes architecture](./arch.jpg)

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

## Future Components

Only the following platform components are planned next.

### External Secrets Operator With AWS Secrets Manager

External Secrets Operator will be used to synchronize Kubernetes secrets from AWS Secrets Manager.

Expected use cases:

- Cloudflare Tunnel token
- GitHub Actions Runner Controller credentials
- Grafana admin credentials
- Future application credentials

Plaintext Kubernetes secrets should not be committed to Git.

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
