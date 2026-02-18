# Building a High-Performance Slurm Cluster

**Stack:** Slurm Workload Manager + Munge + NFS

This document lays out a path to build a traditional HPC (High Performance Computing) cluster using **Slurm**. We will configure a **Control Node** to schedule jobs and manage resources, and a set of **Compute Nodes** to execute the workloads.

Please note that, unlike the other documents in this guide, this document was AI generated and we have not tested all the steps. Please help us out by verifying this document or improving it if you can.

### Architecture Overview

1. **Control Node:**
* Runs **slurmctld** (The central controller daemon).
* Runs **Munge** (Authentication service).
* Hosts the **Shared Storage (NFS)**.
* Hardware: CPU-optimized, high memory for scheduling logic.


2. **Compute Nodes:**
* Runs **slurmd** (The compute node daemon).
* Executes user jobs.
* Hardware: GPU-accelerated or High-Core Count CPUs.



---

# Phase 1: Infrastructure Prep (All Nodes)

*Perform these steps on **ALL** machines (Control Node and all Compute Nodes).*

## 1. OS & Networking

**OS Baseline:** Ensure all nodes use **Ubuntu 22.04 LTS**.

**Update & Install Basics:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y munge libmunge-dev build-essential git mariadb-server \
  libssl-dev python3 python3-pip

```

**Set Hostnames:**
Run this on every node to ensure unique identification.

```bash
# On the Control Node
sudo hostnamectl set-hostname control-01

# On Compute Nodes (repeat for node-02, etc.)
sudo hostnamectl set-hostname node-01

```

**Resolution (/etc/hosts):**
Slurm relies heavily on DNS. If you don't have an internal DNS server, edit `/etc/hosts` on **every single node**.

```bash
sudo nano /etc/hosts

# Add lines like:
# 192.168.1.10  control-01
# 192.168.1.11  node-01
# 192.168.1.12  node-02

```

## 2. Synchronize Users

*Slurm requires the `slurm` user (and any actual human users) to have the exact same UID/GID across the entire cluster.*

```bash
# Create the Slurm user with a specific UID (e.g., 1100)
sudo groupadd -g 1100 slurm
sudo useradd -m -u 1100 -g 1100 -s /bin/bash slurm

# (Optional) specific UID for your admin user if creating fresh:
# sudo useradd -m -u 2001 -s /bin/bash researcher

```

## 3. Configure Munge (Authentication)

*Munge allows nodes to trust each other. They must share the exact same secret key.*

**A. Generate Key (On Control Node ONLY):**

```bash
sudo apt install -y munge
sudo /usr/sbin/create-munge-key

```

**B. Distribute Key (From Control to ALL Compute Nodes):**
Copy the key securely to every worker.

```bash
# Run from Control Node
sudo scp /etc/munge/munge.key ubuntu@node-01:/tmp/munge.key
# Repeat for all nodes...

```

**C. Install Key (On ALL Compute Nodes):**

```bash
# Move key to correct location
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key

# Restart Munge
sudo systemctl enable munge
sudo systemctl restart munge

# Verify Munge is working
munge -n | unmunge

```

---

# Phase 2: Shared Storage (NFS)

*Slurm requires a shared directory (usually `/home` or `/opt`) so that jobs can access data and scripts regardless of which node they run on.*

## 1. Configure Server (Control Node)

```bash
sudo apt install nfs-kernel-server -y
sudo mkdir -p /storage/slurm_data
sudo chown slurm:slurm /storage/slurm_data
sudo chmod 755 /storage/slurm_data

# Export the directory
echo "/storage/slurm_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

sudo exportfs -a
sudo systemctl restart nfs-kernel-server

```

## 2. Configure Clients (Compute Nodes)

```bash
sudo apt install nfs-common -y
sudo mkdir -p /storage/slurm_data

# Mount the share
echo "control-01:/storage/slurm_data /storage/slurm_data nfs defaults 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# Verify
df -h | grep slurm_data

```

---

# Phase 3: Install Slurm Controller

*Perform this ONLY on the **Control Node** (`control-01`).*

## 1. Install Slurmctld

```bash
sudo apt install -y slurmctld slurm-client

```

## 2. Create Configuration (`slurm.conf`)

This is the most critical step. We define the cluster topology.

**Create the config directory:**

```bash
sudo mkdir -p /etc/slurm-llnl
sudo cp /usr/share/doc/slurm-client/examples/slurm.conf.simple.gz /etc/slurm-llnl/
sudo gzip -d /etc/slurm-llnl/slurm.conf.simple.gz
sudo mv /etc/slurm-llnl/slurm.conf.simple /etc/slurm-llnl/slurm.conf

```

**Edit `/etc/slurm-llnl/slurm.conf`:**

Use the settings below as a template. You **must** adjust `CPUs`, `RealMemory`, and `Sockets` to match your actual hardware (use `lscpu` and `free -m` to check).

```ini
# CONTROL NODE CONFIG
ClusterName=ai-cluster
SlurmctldHost=control-01
# SlurmUser must match the user created in Phase 1
SlurmUser=slurm
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
StateSaveLocation=/var/lib/slurm-llnl/slurmctld
SwitchType=switch/none
MpiDefault=none
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack/cgroup
ReturnToService=1

# TIMERS
SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0

# SCHEDULING
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# LOGGING
SlurmctldDebug=3
SlurmctldLogFile=/var/log/slurm-llnl/slurmctld.log
SlurmdDebug=3
SlurmdLogFile=/var/log/slurm-llnl/slurmd.log

# COMPUTE NODES DEFINITION
# REPLACE values below with your actual hardware specs!
# run 'slurmd -C' on a worker to get the exact configuration string
NodeName=node-[01-20] CPUs=64 RealMemory=128000 Sockets=2 CoresPerSocket=16 ThreadsPerCore=2 State=UNKNOWN

# PARTITION DEFINITION
PartitionName=debug Nodes=ALL Default=YES MaxTime=INFINITE State=UP

```

## 3. Create Cgroup Configuration

Slurm uses Linux cgroups to contain processes and manage resources (RAM/CPU) strictly.

Create `/etc/slurm-llnl/cgroup.conf`:

```ini
CgroupAutomount=yes
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAM=yes
ConstrainSwap=yes

```

## 4. Allow Required Ports

Open ports for the Slurm daemon communication.

```bash
sudo ufw allow 6817/tcp  # slurmctld
sudo ufw allow 6818/tcp  # slurmd
sudo ufw reload

```

## 5. Start the Controller

```bash
sudo mkdir -p /var/log/slurm-llnl
sudo chown slurm:slurm /var/log/slurm-llnl
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
sudo systemctl status slurmctld

```

---

# Phase 4: Install Compute Workers

*Perform this on **ALL Compute Nodes**.*

## 1. Install Slurmd

```bash
sudo apt install -y slurmd slurm-client

```

## 2. Sync Configuration

The `slurm.conf` and `cgroup.conf` must be **identical** across the entire cluster.

**On Control Node:**

```bash
# Copy configs to worker
scp /etc/slurm-llnl/slurm.conf ubuntu@node-01:/tmp/
scp /etc/slurm-llnl/cgroup.conf ubuntu@node-01:/tmp/

```

**On Compute Node:**

```bash
sudo mv /tmp/slurm.conf /etc/slurm-llnl/
sudo mv /tmp/cgroup.conf /etc/slurm-llnl/
sudo chown root:root /etc/slurm-llnl/*.conf

```

## 3. Start the Daemon

```bash
sudo systemctl enable slurmd
sudo systemctl start slurmd
sudo systemctl status slurmd

```

> **Verification:** The node should now attempt to contact `control-01`.

---

# Phase 5: Verification & Testing

*Perform these checks on the **Control Node**.*

## 1. Check Cluster State

Run `sinfo` to see the state of your partition and nodes.

```bash
sinfo

# Expected Output:
# PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
# debug* up   infinite      2   idle node-[01-02]

```

If State is `down` or `drain`, check the logs at `/var/log/slurm-llnl/slurmctld.log`.
A common fix for new nodes is to manually update their state to IDLE:

```bash
sudo scontrol update NodeName=node-01 State=RESUME
sudo scontrol update NodeName=node-02 State=RESUME

```

## 2. Run a Test Job

Create a test script `test_job.sh` on the shared storage:

```bash
cd /storage/slurm_data
nano test_job.sh

```

**Content:**

```bash
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=result.out
#SBATCH --nodes=1
#SBATCH --ntasks=1

echo "Hello from $(hostname)"
sleep 10
echo "Job Complete"

```

**Submit the job:**

```bash
sbatch test_job.sh

```

**Monitor the queue:**

```bash
squeue

```

**Check results:**

```bash
cat result.out
# Should print: "Hello from node-01"

```

## 3. Troubleshooting

If nodes are not joining:

1. **Time Sync:** Ensure clocks are synced (`date`). Slurm fails if clocks drift > 5 mins.
2. **Munge:** Run `munge -n | ssh node-01 unmunge` to verify authentication works across the network.
3. **Hardware Specs:** Run `slurmd -C` on the worker. If the CPU/Memory output does not match `slurm.conf` exactly, the node will drain.