# Building a Small (3-5 Node) SkyPilot + k3s Cluster - Install Instructions

This guide covers setting up a Kubernetes-based ML cluster. Unlike a single-server setup, we will split the roles into two types of nodes:

1. **The Head Node (Control & Interface):** This is the "brain" of the cluster. It runs the Kubernetes control plane, the SkyPilot interface, and Transformer Lab. Users log in here to launch jobs. It does *not* necessarily need a GPU.
2. **Worker Nodes (Compute):** These are the "muscle." They run the actual heavy ML workloads. They have the NVIDIA GPUs and simply execute instructions sent by the Head Node.

---

# Step 1 - Operating System & Base Networking

*Perform this step on **ALL** nodes (Head Node and Worker Nodes).*

## 1. Initial OS Installation

**Phase 1: Ubuntu 22.04 Installation**
We highly recommend **Ubuntu 22.04 LTS** as it has the best support for NVIDIA drivers and container runtimes.

1. **Download & Flash:** Download the [Ubuntu 22.04 LTS ISO]() and flash it to a USB.
2. **BIOS Settings:** Ensure **Secure Boot** is **Disabled** in your BIOS. Secure Boot blocks proprietary NVIDIA drivers.
3. **The Install:**
* Select **Minimal Install** to keep the system lean.
* Select **Do not install third-party drivers** (we will handle this manually).
* Select "Erase disk and install Ubuntu."



## 2. Basic System Preparation

Run the following on all nodes to install essential tools and prepare the network.

**Crucial:** You must disable **Swap**. Kubernetes will fail or behave erratically if Swap is enabled.

```bash
# Update package lists
sudo apt update && sudo apt upgrade -y

# Install basic tools and SSH
sudo apt install curl openssh-server -y
sudo systemctl enable --now ssh

# Optional: Install Tailscale for easy remote networking
curl -fsSL https://tailscale.com/install.sh | sh

# Disable Swap (REQUIRED for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

```

---

# Step 2 - Network File System (NFS)

*We need a shared folder so all nodes can see the same data (datasets, checkpoints, logs).*

## 1. Configure the NFS Server (On the Head Node)

*Assume the Head Node has the large storage disk.*

1. **Install the Server:**
```bash
sudo apt install nfs-kernel-server -y

```


2. **Create the Shared Directory:**
```bash
sudo mkdir -p /mnt/nfs_share
# Adjust permissions so clients can write to it
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share

```


3. **Export the Share:**
Edit `/etc/exports`:
```bash
sudo nano /etc/exports

```


Add this line (allows any node on your subnet to connect):
`/mnt/nfs_share *(rw,sync,no_subtree_check)`
4. **Apply Changes:**
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

```



## 2. Configure NFS Clients (On ALL Worker Nodes)

1. **Install Client:**
```bash
sudo apt install nfs-common -y

```


2. **Create Mount Point:**
```bash
sudo mkdir -p /mnt/shared_data

```


3. **Auto-Mount on Boot:**
Edit `/etc/fstab`:
```bash
sudo nano /etc/fstab

```


Add this line (replace `<HEAD_NODE_IP>` with the IP of your Head Node):
```
<HEAD_NODE_IP>:/mnt/nfs_share /mnt/shared_data nfs defaults,user,exec,_netdev 0 0

```


4. **Mount Now:**
```bash
sudo mount -a

```



---

# Step 3 - Setting up the Head Node

*Perform these steps ONLY on the **Head Node**.*

This node will manage the cluster. We will install the K3s **Server** component here.

## 1. Install K3s Server

```bash
curl -sfL https://get.k3s.io | sh -

```

## 2. Prepare for Workers

You need two pieces of information to connect your workers to this head node: the **IP Address** and the **Node Token**.

1. **Get your IP:** Run `ip addr` (look for your LAN IP, usually starting with 192.168... or 10...).
2. **Get the Node Token:**
```bash
sudo cat /var/lib/rancher/k3s/server/node-token

```


*(Copy this token; you will need it for the workers).*

## 3. Install Helm

We need Helm to install the NVIDIA software later.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

```

---

# Step 4 - Setting up Worker Nodes

*Perform these steps on **EVERY Worker Node**.*

> **💡 Administrator Note:**
> If you have many worker nodes, running these commands manually on every machine is tedious and error-prone.
> We highly recommend using an automation tool like **Ansible** to run these steps across all your workers simultaneously.

## 1. Install NVIDIA Drivers

We need the physical drivers on the host so the GPU hardware is recognized.

```bash
# Search for drivers
ubuntu-drivers devices

# Install the recommended driver (e.g., 590 or latest)
sudo apt install nvidia-driver-590 -y

```

## 2. Install NVIDIA Container Toolkit

This allows K3s (containerd) to "talk" to the GPU.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit

# REBOOT NOW to load drivers
sudo reboot

```

## 3. Join the Cluster (Install K3s Agent)

After rebooting, install K3s in **Agent Mode**. Replace `<HEAD_IP>` and `<TOKEN>` with the values you retrieved in Step 3.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<HEAD_IP>:6443 K3S_TOKEN=<TOKEN> sh -

```

## 4. Configure K3s for NVIDIA

We must tell K3s to use the NVIDIA runtime. Create/edit the configuration template:

`sudo nano /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl`

**Paste the following block** into the file:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"

```

Save and exit, then restart K3s:

```bash
sudo systemctl restart k3s-agent

```

---

# Step 5 - Finalize the Cluster

*Go back to the **Head Node** for this step.*

Now that workers are connected, we need to deploy the software that manages the GPUs across the cluster.

## 1. Verify Nodes are Connected

Run this on the Head Node:

```bash
sudo kubectl get nodes

```

You should see your Head Node (control-plane) and all your Worker nodes listed.

## 2. Install NVIDIA GPU Operator

We will use Helm to install the operator. Since we manually installed drivers on the workers (to ensure stability), we tell the operator to skip driver installation.

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

## 3. Verify GPU Availability

Wait a few minutes, then check if the cluster sees the GPUs:

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,GPU:.status.allocatable['nvidia\.com/gpu']"

```

If you see numbers under the `GPU` column for your worker nodes, your cluster is ready!

---

# Step 6 - Install SkyPilot & Transformer Lab

*Perform this on the **Head Node**.*

Since the Head Node acts as the interface, we install the user tools here.

## 1. Configure Kubeconfig for the User

SkyPilot needs to access the cluster as your user, not root.

```bash
# Create the config directory
mkdir -p ~/.kube

# Copy the K3s configuration
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Take ownership so you can read it without sudo
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config

# Point your environment to the new config
export KUBECONFIG=~/.kube/config

```

## 2. Install uv & SkyPilot

```bash
# Install uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Create virtual env
uv venv --seed --python 3.10
source .venv/bin/activate

# Install SkyPilot with Kubernetes support
uv pip install "skypilot[kubernetes]"

```

> **Note:** You can also choose to install this on your base env instead of using `uv` if the machine is only being used for the cluster.

## 3. Verify & Start the API

Now we check if SkyPilot can "see" the GPUs inside your Kubernetes cluster and start the management dashboard.

```bash
sky show-gpus --cloud kubernetes
```

Start the API (accessible from your browser):

```bash
sky api start --deploy
```

Note: adding `--deploy` to the api start command makes the API listen on 0.0.0.0 (all networks) so external users can access SkyPilot.

Once the API is running, you can access the graphical interface to monitor your cluster:

- **URL:** `http://<YOUR_IP_ADDRESS>:46580/dashboard`
- **User Management:** Visit `http://<YOUR_IP_ADDRESS>:46580/users` to find your **User ID** and **User Name**, which you will need for setting up Transformer Lab

## 4 Making SkyPilot Persistent with systemd

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

## 5. Install Transformer Lab

Install Transformer Lab by [following the instructions here](https://lab.cloud/for-teams/install). It will automatically detect the local SkyPilot configuration.