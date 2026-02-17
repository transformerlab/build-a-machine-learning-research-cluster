# The "Under-the-Desk" Server (Single Node) - Install Instructions

Here are full step by step instructions to setting up a single workstation to work in a shared environment.

Our recommendations may seem overkill (installing kubernetes on just a single machine) but doing this gives scalability: you can easily scaleup a system like this to 50 nodes by using kubernetes as your underlying layer.

# Step 1 - Installing the Operating System

# Ubuntu 22.04 + K3s GPU Cluster Setup Guide

This guide will walk you through setting up a fresh Ubuntu 22.04 installation and configuring it to run GPU-accelerated workloads using K3s and NVIDIA. We highly recommend Ubuntu 22.04 as it has the best support for NVIDIA GPUs.

## 1. Initial OS Installation

### Phase 1: Ubuntu 22.04 Installation

<img src="./images/canonical-ubuntu-jammy-jellyfish.jpg" width=300>

1. **Download & Flash:** Download the [Ubuntu 22.04 LTS ISO](https://ubuntu.com/download/desktop) and flash it to a USB drive using a tool like **BalenaEtcher** or **Rufus**.

2. **BIOS Settings:** Ensure **Secure Boot** is **Disabled** in your BIOS. Secure Boot can block proprietary NVIDIA drivers from loading.

3. **The Install:** Boot from the USB and choose **"Install Ubuntu."**
   - **Version:** Install **Ubuntu 22.04 LTS** (recommended for the most stable NVIDIA driver support).
   - **Installation Type:** Choose **Minimal Install** to keep the system lean.
   - **Drivers:** Select **Do not install third-party drivers** during the wizard. We will handle this manually to ensure compatibility.
   - Select "Erase disk and install Ubuntu" for a clean ML environment.

4. **Setup:** Create your primary user, log in, and open your terminal.

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

After the system reboots, verify that the NVIDIA drivers are working correctly and your GPUs are visible by running:

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

[**Now you can continue to Step 2 to install Skypilot -->**](./03-03-02-skypilot.md)

---

# Step 2. Installing Skypilot

This next phase connects your Kubernetes cluster to **SkyPilot**, an orchestrator that makes running AI workloads much easier. I've organized this to ensure your environment variables and permissions are handled correctly from the start, and we'll set up SkyPilot as a systemd service to ensure it persists across reboots.

# Step 2: Installing & Configuring SkyPilot

Now that K3s is running with GPU support, we will install SkyPilot to manage your jobs. SkyPilot will use your local K3s cluster as its "cloud" infrastructure.

## 1. Install System Dependencies

SkyPilot requires a few networking utilities to handle port forwarding and communication with the cluster.

```bash
sudo apt update
sudo apt install -y socat netcat-openbsd
```

---

## 2. Link K3s to your User Profile

SkyPilot looks for cluster credentials at `~/.kube/config`. Since K3s stores its config in a system folder, we need to copy and "own" it as a regular user.

```bash
# Create the config directory
mkdir -p ~/.kube

# Copy the K3s configuration
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Take ownership so SkyPilot can read it without sudo
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config

# Point your environment to the new config
export KUBECONFIG=~/.kube/config
```

---

## 3. Install the `uv` Package Manager

We’ll use **uv**, an extremely fast Python package manager, to handle the SkyPilot installation and its virtual environment.

```bash
# Download and install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Reload your shell environment to recognize 'uv'
source $HOME/.local/bin/env
```

---

## 4. Set Up SkyPilot Virtual Environment

SkyPilot works best inside a dedicated Python environment. We’ll use Python 3.10 for maximum compatibility.

```bash
# Create a virtual environment with pip pre-installed
uv venv --seed --python 3.10

# Activate the environment
source .venv/bin/activate

# Install SkyPilot with Kubernetes support
uv pip install "skypilot[kubernetes]"
```

> **Note:** You can also choose to install this on your base env instead of using `uv` if the machine is only being used for the cluster.
---

## 5. Verify & Start the API

Now we check if SkyPilot can "see" the GPUs inside your Kubernetes cluster and start the management dashboard.

### Check GPU Availability

```bash
sky show-gpus --cloud kubernetes
```

> **Note:** You should see your NVIDIA card listed in the output table.
> 

### Launch the SkyPilot API & Dashboard

```bash
sky api start --deploy
```

Note: adding `--deploy` to the api start command makes the API listen on 0.0.0.0 (all networks) so external users can access SkyPilot.

---

## 6. Accessing the Dashboard

Once the API is running, you can access the graphical interface to monitor your cluster:

- **URL:** `http://<YOUR_IP_ADDRESS>:46580/dashboard`
- **User Management:** Visit `http://<YOUR_IP_ADDRESS>:46580/users` to find your **User ID** and **User Name**, which you will need for setting up Transformer Lab

---

## 7. Making SkyPilot Persistent with systemd

To ensure SkyPilot starts automatically after a reboot and runs as a background service, we'll create a systemd unit file.

### 1. Create the service file

Create a new `systemd` unit:

```bash
sudo nano /etc/systemd/system/skypilot-api.service
```

Paste this template, replacing **`YOUR_USERNAME`** with your actual Linux username (e.g. `transformerlab`) and adjusting paths if your home directory or venv path is different:

```ini
[Unit]
Description=SkyPilot API Service
After=network.target k3s.service
Wants=k3s.service

[Service]
Type=simple
User=YOUR_USERNAME
Group=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME
Environment="PATH=/home/YOUR_USERNAME/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="KUBECONFIG=/home/YOUR_USERNAME/.kube/config"
ExecStart=/home/YOUR_USERNAME/.venv/bin/sky api start --deploy --foreground
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

For a machine with a user called `transformerlab`, this likely becomes:

```ini
User=transformerlab
Group=transformerlab
WorkingDirectory=/home/transformerlab
Environment="PATH=/home/transformerlab/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="KUBECONFIG=/home/transformerlab/.kube/config"
ExecStart=/home/transformerlab/.venv/bin/sky api start --deploy --foreground
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### 2. Enable and start the service

Reload `systemd` so it picks up the new unit, then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable skypilot-api.service
sudo systemctl start skypilot-api.service
sudo systemctl status skypilot-api.service
```

---

### 3. Managing the SkyPilot service

```bash
# Start / stop / restart / status
sudo systemctl start skypilot-api
sudo systemctl stop skypilot-api
sudo systemctl restart skypilot-api
sudo systemctl status skypilot-api
```

---

### 4. Logs & troubleshooting

```bash
# Follow live logs
sudo journalctl -u skypilot-api -f

# Last 100 log lines
sudo journalctl -u skypilot-api -n 100
```

If the service won't start, run the command manually first to confirm it works:

```bash
source /home/YOUR_USERNAME/.venv/bin/activate
export KUBECONFIG=/home/YOUR_USERNAME/.kube/config
sky api start --deploy --foreground
```

Once that works, the unit above should keep it running and auto-start it on reboot.

---

### Troubleshooting Tip:

If you close your terminal and return later, remember to re-activate your environment:

```
source .venv/bin/activate
```

---

# Step 3. Install Transformer Lab

Install Transformer Lab by [following the instructions here](https://lab.cloud/for-teams/install)