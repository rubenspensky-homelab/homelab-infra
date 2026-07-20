# GPU Support

GPU support in this cluster is intentionally split into two small components instead of using the NVIDIA GPU Operator:

- NVIDIA Device Plugin exposes NVIDIA GPUs to Kubernetes workloads.
- NVIDIA DCGM Exporter exposes NVIDIA GPU metrics to Prometheus.

This keeps GPU enablement predictable and minimal for the homelab. The GPU Operator was tested, but it caused multiple operational problems in this environment, so it is not used here. The cluster manages the host-level NVIDIA driver and container runtime configuration outside of Kubernetes, then uses Kubernetes only for device registration and metrics collection.

## Source Paths

- Device plugin application: `applications/nvidia-device-plugin.yaml`
- Device plugin manifests: `infrastructure/nvidia-device-plugin/`
- DCGM Exporter application: `applications/dcgm-exporter.yaml`
- DCGM Exporter wrapper chart: `infrastructure/dcgm-exporter/`

## Why GPU Support Needs Extra Components

Kubernetes does not automatically know how to schedule NVIDIA GPUs just because the host has a GPU and driver installed. The kubelet needs a device plugin to report allocatable GPU devices, and workloads need the NVIDIA runtime to make those devices available inside containers.

Observability is a separate concern. A workload can use a GPU without exposing useful metrics about temperature, memory, power, or utilization. DCGM Exporter provides those metrics for Prometheus and Grafana.

In this cluster:

- Device Plugin answers: can Kubernetes schedule and attach a GPU to a pod?
- DCGM Exporter answers: what is the GPU doing after workloads start using it?

## Why The GPU Operator Is Not Used

The NVIDIA GPU Operator is designed to manage the full NVIDIA stack from inside Kubernetes, including driver-related components, toolkit configuration, device plugin deployment, and observability integrations.

That is useful in larger or more homogeneous environments, but it was too heavy for this homelab and caused several operational issues during testing. For this cluster, the simpler and more reliable approach is:

- Install and maintain NVIDIA drivers on the host OS.
- Configure the NVIDIA container runtime on the host.
- Keep Kubernetes manifests focused on the device plugin and metrics exporter.
- Avoid an operator reconciling host-level GPU components that are already managed manually.

This reduces moving parts and makes failures easier to isolate. If GPU scheduling breaks, the likely scope is the host driver/runtime or the device plugin. If GPU dashboards break, the likely scope is DCGM Exporter or Prometheus scraping.

## NVIDIA Device Plugin

The NVIDIA Device Plugin is deployed by Argo CD from:

```text
applications/nvidia-device-plugin.yaml
```

It applies the Kustomize manifests in:

```text
infrastructure/nvidia-device-plugin/
```

The application deploys into `kube-system` with sync wave `-8`, so GPU resource registration happens early before GPU-aware workloads are scheduled.

Managed resources:

- `ServiceAccount/nvidia-device-plugin`
- `ClusterRole/nvidia-device-plugin`
- `ClusterRoleBinding/nvidia-device-plugin`
- `RuntimeClass/nvidia`
- `DaemonSet/nvidia-device-plugin-daemonset`

The DaemonSet uses NVIDIA's static device plugin image:

```text
nvcr.io/nvidia/k8s-device-plugin:v0.17.1
```

The plugin registers with kubelet through the host device plugin socket directory:

```text
/var/lib/kubelet/device-plugins
```

Once registered, kubelet can advertise GPU capacity as:

```text
nvidia.com/gpu
```

Workloads can then request GPU access with resource limits such as:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

The DaemonSet uses:

```yaml
runtimeClassName: nvidia
```

The corresponding RuntimeClass is:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

This expects the container runtime on the node to have an `nvidia` runtime handler configured.

Current scheduling constraints:

```yaml
nodeSelector:
  accelerator: nvidia
  node-role.kubernetes.io/control-plane: ""
```

Current tolerations:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

This means the device plugin is intentionally targeted at the NVIDIA-capable control-plane node. In the current hardware inventory, that node is `k8s-control-01`, which has an NVIDIA GeForce GTX 1060 6GB.

## Why The Device Plugin Is Required

Without the device plugin:

- Kubernetes will not advertise `nvidia.com/gpu` on the node.
- Pods cannot reliably request GPU resources with `resources.limits.nvidia.com/gpu`.
- The scheduler cannot reserve GPU capacity for workloads.
- Kubelet will not allocate NVIDIA devices to containers through the device plugin API.
- GPU workloads may start without actual GPU access or fail depending on runtime configuration.

The device plugin does not install the NVIDIA driver. It assumes the host is already prepared with the driver, NVIDIA Container Toolkit, and container runtime handler.

## DCGM Exporter

DCGM Exporter is deployed by Argo CD from:

```text
applications/dcgm-exporter.yaml
```

It deploys the wrapper chart in:

```text
infrastructure/dcgm-exporter/
```

The wrapper chart depends on NVIDIA's upstream DCGM Exporter Helm chart:

```yaml
dependencies:
  - name: dcgm-exporter
    version: 4.8.2
    repository: https://nvidia.github.io/dcgm-exporter/helm-charts
```

The application deploys into the `monitoring` namespace with sync wave `-4`. It is part of the observability stack and is expected to be scraped by Prometheus.

Current values:

```yaml
dcgm-exporter:
  runtimeClassName: nvidia
  nodeSelector:
    accelerator: nvidia
    node-role.kubernetes.io/control-plane: ""
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
  serviceMonitor:
    enabled: true
```

The node selector and tolerations match the device plugin placement so metrics are collected from the same NVIDIA-capable node.

## Why DCGM Exporter Is Required

The device plugin makes the GPU schedulable, but it does not provide complete GPU observability. DCGM Exporter exposes NVIDIA Data Center GPU Manager metrics in a Prometheus-compatible format.

These metrics are useful for understanding:

- GPU utilization.
- GPU memory usage.
- Temperature.
- Power draw.
- Clock behavior.
- Device health.
- Error conditions.

With `serviceMonitor.enabled: true`, kube-prometheus-stack can discover and scrape DCGM Exporter automatically. Grafana can then use Prometheus data for GPU dashboards and alerts.

## Relationship Between Components

The two components have different responsibilities:

| Component | Responsibility |
| --- | --- |
| NVIDIA Device Plugin | Advertises GPUs to Kubernetes and enables GPU scheduling with `nvidia.com/gpu` |
| DCGM Exporter | Exposes GPU metrics for Prometheus and Grafana |

Both depend on the host NVIDIA stack being configured correctly. Neither component replaces the host driver or NVIDIA Container Toolkit setup.

## Node Requirements

The GPU node must have:

- NVIDIA driver installed on the host OS.
- NVIDIA Container Toolkit installed and configured.
- Container runtime handler named `nvidia`.
- Node label `accelerator=nvidia`.
- Any GPU-related taints matched by the DaemonSet tolerations.

Current manifests also require the control-plane label selector:

```text
node-role.kubernetes.io/control-plane=
```

If another node is added with an NVIDIA GPU, update the node selectors if GPU support should run outside the current control-plane GPU node.

## Validation

Check RuntimeClass:

```bash
kubectl get runtimeclass nvidia
```

Check the device plugin pod:

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=nvidia-device-plugin
```

Check GPU resources advertised by the node:

```bash
kubectl describe node k8s-control-01 | grep -A5 nvidia.com/gpu
```

Check DCGM Exporter pods:

```bash
kubectl -n monitoring get pods | grep dcgm
```

Check DCGM Exporter ServiceMonitor:

```bash
kubectl -n monitoring get servicemonitor | grep dcgm
```

Check whether Prometheus sees DCGM metrics:

```text
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_FB_USED
DCGM_FI_DEV_GPU_TEMP
```
