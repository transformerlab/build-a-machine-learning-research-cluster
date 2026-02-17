# Building a Scalable AI Research Cluster

**Stack:** Rancher + SkyPilot + Transformer Lab

This document lays out a path using **Rancher** as a centralized platform to manage Kubernetes. We will install Rancher on a single **Management Node**, and it will manage a separate cluster of **Compute Nodes**.

Complete Rancher docs are available at [What is Rancher?](https://ranchermanager.docs.rancher.com/).

### Architecture Overview

1. **Management Node (The "Brain"):**
  * Runs **Rancher** (UI/Management), **SkyPilot** (Orchestrator), and **Transformer Lab** (User Interface).
  * Hardware: CPU-optimized.
  * Software: Runs a lightweight K3s purely to host the Rancher platform.


2. **Compute Cluster (The "Muscle"):**
  * Runs the heavy ML workloads.
  * Hardware: GPU-accelerated (NVIDIA).
  * Software: Managed by Rancher (RKE2/K3s installed automatically via Rancher agent).



---

# Phase 1: Infrastructure Prep (All Nodes)

*Perform these steps on **ALL** machines (Management Node and all 20+ Compute Nodes).*

## 1. OS & Networking

**OS Baseline:** Ensure all nodes use **Ubuntu 22.04 LTS**.

**Required Packages:**

```bash
sudo apt update
sudo apt install -y curl iproute2 ca-certificates git

```

**Set Hostnames:**
Run this on every node, replacing `node-01` with the unique name (e.g., `mgmt-01`, `node-01`... `node-20`).

```bash
sudo hostnamectl set-hostname node-01

```

**Resolution (/etc/hosts):**
Edit `/etc/hosts` to ensure the node can resolve itself and the management node.

```bash
# Add lines like:
# 192.168.1.10  mgmt-01
# 192.168.1.11  node-01

```

**Disable Swap (Crucial):**
Kubernetes will not run with swap enabled.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

```

**Kernel Modules:**
Load modules for container networking.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

```

## 2. Shared Storage (NFS)

*Compute nodes need a shared view of data.*

**A. On the NFS Server (Likely the Management Node):**

```bash
sudo apt install nfs-kernel-server -y
sudo mkdir -p /mnt/nfs_share
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share
echo "/mnt/nfs_share *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

```

**B. On ALL Compute Nodes (Clients):**

```bash
sudo apt install nfs-common -y
sudo mkdir -p /mnt/shared_data
# Add to /etc/fstab (Replace <MGMT_IP> with actual IP):
echo "<MGMT_IP>:/mnt/nfs_share /mnt/shared_data nfs defaults,user,exec,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a

```

# Phase 2: Install Rancher Management Server

*Perform this ONLY on the **Management Node** (`mgmt-01`) or up to 3 management nodes for redudancy*

We will install Rancher on your **Management Node(s)**. For the remainder of these instructions, we will assume a single management node, although High Availability environments will often recommended to dedicate 3 nodes. Note that if the single management node goes down you will lose access to Rancher, but your compute cluster will still be operational.

Your management node will not be used as a compute worker.

<aside>
💡
Some HPC environments refer to a “head node’” In this guide we use the term **management node** to describe the nodes running Rancher, to avoid confusion with Kubernetes control-plane nodes that run in the compute cluster.
</aside>

### Server Requirements

A per the [Rancher docs](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements/port-requirements#ports-for-rancher-server-nodes-on-k3s)

- Open ports
    - TCP on port 80 and 443
    - Other cluster nodes need to be able to access port 6443 over TCP


## 1. Install Lightweight K8s (K3s)

We use K3s to host the Rancher software itself.

```bash
# Install K3s (Server Mode)
curl -sfL https://get.k3s.io | sudo sh -s - --write-kubeconfig-mode 644

# Set Environment Variable
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc

# Verify
kubectl get nodes

```

> **Note:** Do not install K3s on the compute nodes yet. Rancher will handle that later.

## 2. Install Helm & Cert-Manager

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash

# Install Cert-Manager (Required for Rancher SSL)
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
  
# Verify rollout
kubectl -n cert-manager rollout status deploy/cert-manager --timeout=180s

```

## 3. Install Rancher

Replace `<HOST_NAME>` with your DNS name (e.g., `rancher.lab.com`) or `<PUBLIC_IP>.sslip.io`.

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system --create-namespace \
  --set hostname="<HOST_NAME>" \
  --set replicas=1 \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@yourdomain.com \
  --set letsEncrypt.ingress.class=traefik

```

Wait for the deployment to finish:

```bash
kubectl -n cattle-system rollout status deploy/rancher --timeout=600s

```

**Initial Login:**

<img src="./images/rancher-install.png" width="500">

1. Open `https://<HOST_NAME>` in your browser.
2. Log in with `admin` / `admin`.
3. Set the **Server URL** (Keep this consistent; changing it later is difficult).

---

# Phase 3: Provision the Compute Cluster

*We will now turn the worker nodes into a cluster managed by Rancher.*

For more detailed requirements on compute cluster nodes, including detailed port requirements, see [Node Requirements for Rancher Managed Clusters](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/node-requirements-for-rancher-managed-clusters)

## 1. Prepare GPU Drivers (On All Compute Nodes)

*Before joining Rancher, ensure NVIDIA drivers are installed on all worker nodes.*

```bash
# Identify suitable driver
ubuntu-drivers devices

# Install Driver (e.g., 590)
sudo apt install nvidia-driver-590 -y

# Install Container Toolkit (Crucial for Docker/Containerd)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit

# Reboot to load driver
sudo reboot

```

## 2. Create Cluster in Rancher UI

1. Log into Rancher -> **Cluster Management** -> **Create**.
2. Select **Custom**.
3. Name: `paper-cluster-compute-prod`.
4. **Network Provider:** Select **Calico** (Preferred for HPC/GPU).
5. Click **Create**.

## 3. Register the Nodes

Rancher will display a registration command (starts with `sudo docker run...` or `curl...`).

* **For the first 3 nodes:** Check **etcd**, **Control Plane**, and **Worker**. (High Availability).
* **For the remaining 17+ nodes:** Check **Worker** only.

Run the generated command on each respective node.

> **Verification:** In Rancher UI, wait for nodes to turn "Active".

## 4. Install NVIDIA GPU Operator (via Rancher)

Once the cluster is active, we deploy the software that lets Kubernetes see the GPUs.

1. In Rancher, select your `paper-cluster-compute-prod` cluster.
2. Go to **Apps & Marketplace** -> **Charts**.
3. Search for **NVIDIA GPU Operator** and Install.
4. **Important Options:**
* `driver.enabled`: **false** (We installed drivers manually on host).
* `toolkit.enabled`: **false** (We installed toolkit manually on host).
* `operator.defaultRuntime`: **nvidia**.

## 5. Install Monitoring

Install Rancher Monitoring: Apps & Marketplace > Charts, find the [Monitoring](https://www.google.com/search?q=Monitoring&rlz=1C5CHFA_enCA1101CA1101&oq=how+do+i+enable+rancher+monitoring&gs_lcrp=EgZjaHJvbWUyCQgAEEUYORigATIHCAEQIRifBTIHCAIQIRifBTIHCAMQIRifBdIBCDM2MjlqMGo3qAIAsAIA&sourceid=chrome&ie=UTF-8&mstk=AUtExfA9R16GX9sg89is3G2LvwPzeZeu21cjP9RGTyrXtjfnLYu32jlj7sX_bg8Qf6LgckmR4jmllvde3wlI8r0cHny8CXI5-lHfSbDDv16IpEs0XZcQuSGt-Wu7YjofDwoB6Ufo8c5hLEQKWjcEW4hUa-KuEtiJB1S6R0yQL_hVBoDvDdU&csui=3&ved=2ahUKEwj0id_lmLSSAxVXkokEHUX0AM0QgK4QegQIARAE) chart



---

# Phase 4: SkyPilot Setup

*Perform this on the **Management Node**.*

## 1. Install uv & SkyPilot Environment

We use `uv` for fast Python management.

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Create virtual env (Python 3.10)
uv venv --seed --python 3.10
source .venv/bin/activate

# Install SkyPilot
uv pip install skypilot
uv pip install "skypilot[kubernetes]"

```

## 2. Connect SkyPilot to the Compute Cluster

SkyPilot on the management node needs to "talk" to the new Compute Cluster.

1. **Download Config:** In Rancher UI -> Cluster Management -> Find `paper-cluster-compute-prod` -> Click **⋮** -> **Download KubeConfig**.
2. **Save Config:**
```bash
mkdir -p ~/.kube
nano ~/.kube/config
# Paste the content of the downloaded file here
chmod 600 ~/.kube/config

```


3. **Verify Connection:**
```bash
export KUBECONFIG=~/.kube/config
kubectl get nodes
# Should list all 20+ compute nodes

```



## 3. Initialize SkyPilot

```bash
# Install dependencies
sudo apt-get update && sudo apt-get install -y socat netcat

# Check Cloud (Kubernetes)
sky check
# Verify Output: "Kubernetes: Enabled"

```

## 4. Start SkyPilot API (Systemd Service)

We will run the API as a background service so it persists after reboots.

Create the service file: `sudo nano /etc/systemd/system/skypilot-api.service`

```ini
[Unit]
Description=SkyPilot API Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu
Environment="PATH=/home/ubuntu/.local/bin:/usr/bin:/bin"
Environment="KUBECONFIG=/home/ubuntu/.kube/config"
ExecStart=/home/ubuntu/.venv/bin/sky api start --deploy --foreground
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

```

*(Replace `ubuntu` with your actual username).*

**Start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now skypilot-api

```

## 5. Final Verification

```bash
sky show-gpus --cloud kubernetes

```

---

# Phase 5: Transformer Lab & HTTPS

*Perform this on the **Management Node**.*

## 1. Install Transformer Lab

[Follow the instructions here]().
*(Skip step 1 of the linked guide as we have already configured SkyPilot).*

## 2. Setup Nginx Reverse Proxy (HTTPS)

Expose the lab securely on standard ports (80/443).

**Install Nginx:**

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx

```

**Configure Site:**
`sudo nano /etc/nginx/sites-available/transformerlab.conf`

```nginx
server {
  listen 80 default_server;
  server_name _;
  location / {
    proxy_pass http://127.0.0.1:8338;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}

```

**Activate & Secure:**

```bash
# Enable config
sudo ln -s /etc/nginx/sites-available/transformerlab.conf /etc/nginx/sites-enabled/transformerlab.conf
sudo rm -f /etc/nginx/sites-enabled/default

# Verify and Restart
sudo nginx -t
sudo systemctl restart nginx

# Generate SSL Cert (Follow prompts)
sudo certbot --nginx -d lab.vectorinstitute.ai

```

You can now access your cluster at `https://lab.vectorinstitute.ai` (or your configured domain).