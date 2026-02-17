# Step 2. Installing Skypilot

This next phase connects your Kubernetes cluster to **SkyPilot**, an orchestrator that makes running AI workloads much easier. I’ve organized this to ensure your environment variables and permissions are handled correctly from the start.


> [!TODO]
> Update these docs to install skypilot globally in the base env and update the docs to install as services so they live after reboot

---

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

### Troubleshooting Tip:

If you close your terminal and return later, remember to re-activate your environment:

```
source .venv/bin/activate
```