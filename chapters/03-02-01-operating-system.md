# Step 1 - Installing the Operating System

# Ubuntu 22.04 + K3s GPU Cluster Setup Guide

This guide will walk you through setting up a fresh Ubuntu 22.04 installation and configuring it to run GPU-accelerated workloads using K3s and NVIDIA. We highly recommend Ubuntu 22.04 as it has the best support for NVIDIA GPUs.

## 1. Initial OS Installation

- **Version:** Install **Ubuntu 22.04 LTS** (recommended for the most stable NVIDIA driver support).
- **Installation Type:** Choose **Minimal Install** to keep the system lean.
- **Drivers:** Select **Do not install third-party drivers** during the wizard. We will handle this manually to ensure compatibility.
- **Setup:** Create your primary user, log in, and open your terminal.

---

## 2. Basic System Preparation

First, we’ll install essential tools, enable remote access via SSH, and disable **Swap** (which is required for Kubernetes stability).

```bash
# Update package lists
sudo apt update && sudo apt upgrade -y

# Install basic tools and SSH
sudo apt install curl openssh-server -y
sudo systemctl enable --now ssh

# Optional: Install Tailscale for easy remote networking
curl -fsSL https://tailscale.com/install.sh | sh

# Disable Swap (Required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## 3. NVIDIA Driver & Toolkit Installation

We need both the physical driver and the toolkit that allows containers to access the hardware.

### Install NVIDIA Drivers

Run `ubuntu-drivers devices` to see recommended versions. If `590` is available, proceed, otherwise pick the highest one:

```bash
sudo apt install nvidia-driver-590 -y
```

### Install NVIDIA Container Toolkit

This allows the container runtime (containerd) to interface with your GPU.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit

# Reboot is required to initialize the drivers
sudo reboot
```

### Verify GPU Visibility

After the system reboots, verify that the NVIDIA drivers are working correctly and your GPUs are visible:

```bash
nvidia-smi
```

You should see output displaying your GPU(s), driver version, CUDA version, and current GPU utilization. If you see an error or no GPUs listed, there may be an issue with the driver installation.

---

## 4. K3s and Helm Installation

Once the system reboots, we install the lightweight Kubernetes distribution (**K3s**) and the **Helm** package manager.

```bash
# Install K3s
curl -sfL https://get.k3s.io | sh -

# Set up permissions for the config file
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## 5. Configure K3s for NVIDIA

Note: we are not absolutely sure the next step is required — please check if these lines are already in your config and if so you can skip this step.

> To use GPUs, K3s needs to know about the `nvidia` runtime. You must create/edit a template file so K3s doesn't overwrite your changes on restart.
> 

Create or edit the template file:

`sudo nano /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl`

**Paste the following block** into the file (usually at the end, under the `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]` section):

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
```

*Press `Ctrl+O`, `Enter` to save, and `Ctrl+X` to exit.*

---

## 6. Install NVIDIA GPU Operator

Finally, use Helm to install the operator. Since we already installed the drivers and toolkit on the host, we tell the operator to skip those steps and just manage the runtime.

```bash
# Add NVIDIA Helm repository
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

# Install the operator
helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator --create-namespace \
  --set driver.enabled=false \
  --set toolkit.enabled=false \
  --set operator.defaultRuntime=nvidia
```

---

## 7. Verify the Setup

Wait a minute or two for the pods to initialize, then run this command to see if your node reports an available GPU:

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,GPU:.status.allocatable['nvidia\.com/gpu']"
```

**What success looks like:**

If you see a `1` (or the number of GPUs you have) under the GPU column, you are ready to deploy AI/ML workloads!