# Chapter 5: The "University Cluster" (10–100 Nodes)

**The Scenario:** You have transitioned beyond a single-server setup. You now have a dedicated server room or a caged row in a colocation facility. You are managing 10 to 100 compute nodes—likely a mix of dense training nodes (e.g., 8x H100s) and lighter inference or development nodes.

**The Assumption:** This guide assumes your hardware is racked, power is balanced, and your 100GbE or InfiniBand fabric is operational. This chapter focuses on the **Software Control Plane** required to manage this infrastructure.

---

## 1. The Strategy: Modern vs. Traditional

At this scale, there are two primary architectural paths.

### Path A: The Traditional HPC Route (Slurm)

Commonly used in academic environments, Slurm is a stable workload manager that excels with MPI-based jobs. However, it lacks native container orchestration, making it difficult to host persistent services—such as vector databases or inference endpoints—alongside batch jobs.

### Path B: The "Cloud-Native" Route (Kubernetes + SkyPilot)

**This is the recommended architecture.** Layering SkyPilot on top of Kubernetes provides a balanced environment for both operations and research:

1. **Operations:** Utilizes industry-standard management (Kubernetes) for health checks, networking, and storage.
2. **Researchers:** Use a simplified interface (SkyPilot) that abstracts the complexities of pods, services, and ingresses.

---

## 2. The Recommended Stack

At this scale, your goal is to provide a **Single System Image**, where researchers interact with a unified pool of compute resources rather than individual IP addresses.

### The Infrastructure Architecture

```text
┌─────────────────┐  ┌──────────────┐   ┌──────────────┐                                   
│                 │  │              │   │              │                                   
│  Researcher A   │  │ Researcher B │   │ Researcher C │                                   
│  (Web UI)       │  │ (CLI/API)    │   │ (Notebooks)  │                                   
└─────────┬───────┘  └─┬────────────┘   └─────────┬────┘                                   
          └─────┐      │                          │                                        
                │      │                          │                                        
                │  ┌───┘             ┌────────────┘                                        
                │  │                 │                                                     
      ┌─────────┼──┼─────────────────┼────────┐  ┌──────────────────┐  ┌──────────────────┐
      │         │  │ Control Plane   │        │  │  Worker Node 1   │  │  Worker Node 100 │
      │  ┌──────▼──▼─────────────────▼─────┐  │  │ (RKE2 + GPUs)    │  │ (RKE2 + GPUs)    │
      │  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│  │  │                  │  │                  │
      │  │░░░░░░░░░Transformer Lab░░░░░░░░░│  │  │                  │  │                  │
      │  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│  │  │                  │  │                  │
      │  └─────────────────────────────────┘  │  │ ┌──────────────┐ │  │ ┌──────────────┐ │
      │  ┌─────────────────────────────────┐  │  │ │▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ │  │ │▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ │
      │  │▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│  │  │ │▒▒▒GPU Oper.▒▒│ │  │ │▒▒▒GPU Oper.▒▒│ │
      │  │▒▒▒SkyPilot + Rancher (RKE2)▒▒▒▒▒│  │  │ │▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ │  │ │▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ │
      │  │▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│  │  │ └──────────────┘ │  │ └──────────────┘ │
      └─────────────────────────┬─────────────┘  └────────┬─────────┘  └───────┬──────────┘
                                └──────┐           ┌──────┘          ┌─────────┘           
                                       │           │                 │                     
           ┌───────────────────────────▼───────────▼─────────────────▼────────────────┐    
           │██████████████████████████████████████████████████████████████████████████│    
           │████████████████  Two-Tier Storage: NFS / JuiceFS / S3  ██████████████████│    
           │██████████████████████████████████████████████████████████████████████████│    
           └──────────────────────────────────────────────────────────────────────────┘    

```

### Step 1: The Base Layer (Rancher & RKE2)

Avoid manual Kubernetes installations with `kubeadm` at this scale due to the high administrative overhead.

* **Recommendation:** Use **Rancher** for cluster management and **RKE2** as the distribution.
* **Why:** RKE2 is secure by default and simplifies the deployment of the **NVIDIA GPU Operator**, which automates driver and container runtime synchronization across the cluster.

### Step 2: The Entry Point (SkyPilot + Transformer Lab)

Standard Kubernetes manifests (`.yaml` files) are often impractical for research workflows.

* **SkyPilot:** Operates within the Kubernetes cluster as a scheduler, translating job requests into Kubernetes Pods.
* **Transformer Lab:** Provides a graphical interface for launching tasks, Jupyter notebooks and other jobs without requiring command-line interaction.

### Step 3: The Storage Strategy

At 100 nodes, storage performance is a critical factor. A "Two-Tier" approach is recommended:

1. **Hot Tier (NFS/JuiceFS):** A high-performance shared filesystem for active datasets, mounted to `/home`.
2. **Cold Tier (Object Store):** S3 or MinIO for long-term model checkpoints and logs.

---

## 3. Hardware Resilience

In a 100-node environment, hardware failure is a statistical certainty. At this scale, the Mean Time Between Failures (MTBF) necessitates a plan for the following:

* GPU Xid errors (silent data corruption or process hangs).
* ECC Memory bit-flips.
* Network interface card (NIC) disconnects.
* Power supply unit (PSU) failures.

### The Challenge of Node Failure

In a manual setup, if a node fails during a multi-day training run, the job typically crashes. The researcher must then manually identify a healthy node, migrate the data, and restart the process from the last saved state.

### The Solution: Managed Recovery

The **SkyPilot + Transformer Lab** stack automates recovery from these failures:

1. **Automatic Detection:** SkyPilot monitors job health. If a Kubernetes Pod terminates due to node failure, SkyPilot identifies the event immediately.
2. **Auto-Restart:** SkyPilot automatically provisions a new Pod on an available healthy node and restarts the job.
3. **Checkpointing Requirements:**
* This system requires the user to implement periodic checkpointing.
* The training script must be configured to load the most recent checkpoint from shared storage (NFS/S3) upon startup.
* **Result:** The job resumes with minimal loss of progress.



> [!TIP]
> **Cloud Bursting and Spot Instances:** This auto-recovery logic also supports the use of "Spot" or "Preemptible" instances when bursting to the cloud. This can reduce compute costs by up to 70%, as SkyPilot will automatically relocate the job if the cloud provider reclaims the instances.

---

## 4. Advanced Considerations

### Advanced Storage Solutions

* **JuiceFS:** Standard NFS performance often degrades when handling datasets containing millions of small files. JuiceFS creates a POSIX-compliant file system on top of Object Storage (S3/MinIO), providing the scalability of S3 with the access patterns of a local drive.
* **Longhorn:** For workloads requiring fast, private block devices, Longhorn (by Rancher) provides distributed block storage for Kubernetes, ensuring data replication across nodes.

### Observability & Networking

* **Monitoring:** Deploy **Prometheus** and **Grafana** with the `NVIDIA DCGM Exporter`. This provides visualization for GPU temperatures, power consumption, and process health across the cluster.
* **Networking:** For multi-node training (e.g., 32+ GPUs), ensure that the **RoCE (RDMA over Converged Ethernet)** or **InfiniBand** fabric is exposed to Kubernetes pods via the **SR-IOV CNI plugin** to prevent network bottlenecks during data synchronization.
