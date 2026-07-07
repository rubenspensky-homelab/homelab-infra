# NVIDIA GPU Node Setup

This document describes the host-level configuration required to prepare a Kubernetes node with an NVIDIA GPU for GPU workloads.

> **Node**
>
> - **Hostname:** `homelab`
> - **OS:** Debian GNU/Linux 13 (Trixie)
> - **GPU:** NVIDIA GeForce GTX 1060 6GB
> - **Container Runtime:** containerd 1.7
> - **Kubernetes Role:** Control Plane / GPU Node

---

# Architecture

```
+------------------------------------------------------+
| Debian 13                                            |
|                                                      |
|  NVIDIA Driver                                       |
|        │                                             |
|  NVIDIA Container Toolkit                            |
|        │                                             |
|  containerd                                           |
+--------┼---------------------------------------------+
         │
         ▼
+------------------------------------------------------+
| Kubernetes                                            |
|                                                      |
|  NVIDIA GPU Operator                                 |
|      ├── Device Plugin                               |
|      ├── GPU Feature Discovery                       |
|      ├── DCGM Exporter                               |
|      └── Runtime integration                         |
|                                                      |
|  GPU Workloads                                       |
|      ├── Ollama                                      |
|      ├── Open WebUI                                  |
|      └── Future AI workloads                         |
+------------------------------------------------------+
```

---

# 1. Configure Debian repositories

Enable the required repositories.

`/etc/apt/sources.list`

```text
deb http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
deb-src http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian/ trixie-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ trixie-updates main contrib non-free non-free-firmware
```

Update packages:

```bash
apt update
```

---

# 2. Install kernel headers

The NVIDIA kernel module is built through DKMS and requires the kernel headers.

```bash
apt install linux-headers-amd64
# Install headers for the current Debian kernel
```

---

# 3. Install the NVIDIA Driver

```bash
apt install nvidia-driver
# Install the proprietary NVIDIA driver
```

Reboot:

```bash
reboot
```

Validate:

```bash
nvidia-smi
```

Expected output:

```text
Driver Version: 550.x
CUDA Version: 12.x

GPU:
NVIDIA GeForce GTX 1060 6GB
```

---

# 4. Install the NVIDIA Container Toolkit

## Add the NVIDIA repository

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
| gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
| sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
> /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Update repositories:

```bash
apt update
```

Install:

```bash
apt install nvidia-container-toolkit
```

---

# 5. Configure containerd

Configure the NVIDIA runtime.

```bash
nvidia-ctk runtime configure --runtime=containerd
```

Restart containerd:

```bash
systemctl restart containerd
```

---

# 6. Validate containerd configuration

The runtime configuration should exist:

```text
/etc/containerd/conf.d/99-nvidia.toml
```

The runtime should appear inside the effective configuration:

```bash
containerd config dump | grep -A10 nvidia
```

Expected:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]

runtime_type = "io.containerd.runc.v2"

BinaryName = "/usr/bin/nvidia-container-runtime"
```

The main configuration should import the drop-in directory:

```bash
grep imports /etc/containerd/config.toml
```

Expected:

```toml
imports = ["/etc/containerd/conf.d/*.toml"]
```

---

# 7. Label the Kubernetes GPU node

The NVIDIA device plugin DaemonSet expects the GPU control-plane node to be labeled manually.

For the current cluster, the control-plane node `k8s-control-01` has the NVIDIA GPU:

```bash
kubectl label node k8s-control-01 accelerator=nvidia --overwrite
```

Validate:

```bash
kubectl get node k8s-control-01 --show-labels | grep accelerator
```

---

# Host Validation Checklist

## Driver

```bash
nvidia-smi
```

Expected:

- NVIDIA driver loaded
- CUDA version displayed
- GPU detected

---

## Runtime

```bash
containerd config dump | grep nvidia
```

Expected:

- NVIDIA runtime present

---

## Runtime configuration

```bash
cat /etc/containerd/conf.d/99-nvidia.toml
```

Expected:

- `runtimes.nvidia`
- `BinaryName=/usr/bin/nvidia-container-runtime`

---

# Next Step

The host is now ready.

The remaining GPU components should be managed from Kubernetes using the NVIDIA GPU Operator.

Install with Helm:

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=false
```

> **Important**
>
> `driver.enabled=false` because the NVIDIA driver is already managed by the Debian host.

---

# Final Architecture

```
Host (Debian)

├── NVIDIA Driver
├── NVIDIA Container Toolkit
└── containerd

            │

            ▼

Kubernetes

├── NVIDIA GPU Operator
├── Device Plugin
├── GPU Feature Discovery
├── DCGM Exporter
├── Ollama
└── Open WebUI
```

---

# Future Automation

This setup is an excellent candidate for infrastructure automation.

Recommended tools:

- **Ansible** for host provisioning
- **Argo CD** for Kubernetes resources
- **Helm** for the GPU Operator
- **Terraform** (optional) if the node is provisioned in a cloud environment

The goal is for a fresh Debian installation to become a fully configured GPU-enabled Kubernetes node with a single automation workflow.
