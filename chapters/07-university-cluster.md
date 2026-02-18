# Chapter 5: The "University Cluster" (10–100 Nodes)

**The Scenario:** You have transitioned beyond a single-server setup. You now have a dedicated server room or a caged row in a colocation facility. You are managing 10 to 100 compute nodes—likely a mix of dense training nodes (e.g., 8x H100s) and lighter inference or development nodes.

**The Assumption:** This guide assumes your hardware is racked, power is balanced, and your 100GbE or InfiniBand fabric is operational. This chapter focuses on the **Software Control Plane** required to manage this infrastructure.

---

## 1. The Strategy: Modern vs. Traditional

At this scale, there are two primary architectural paths.

### Path A: The Traditional HPC Route (Slurm)

Commonly used in academic environments, Slurm is a stable workload manager that excels with MPI-based jobs. However, it lacks native container orchestration, making it difficult to host persistent services—such as vector databases or inference endpoints—alongside batch jobs.

### Path B: The "Cloud-Native" Route (Kubernetes + Kueue + SkyPilot)

**This is the recommended architecture.** Kubernetes is excellent at managing containers, but it lacks a native "Queue." By adding **Kueue**, you gain HPC-style job queuing and fair-share quotas on top of a modern API.

1. **Operations:** Utilizes industry-standard management (Kubernetes + Kueue) for health checks, multi-tenant resource quotas, and priority queuing.
2. **Researchers:** Use a simplified interface (SkyPilot) that abstracts the complexities of pods and submits jobs to the managed queues.

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
      │  │▒▒SkyPilot + Kueue + Rancher ▒▒▒▒│  │  │ │▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ │  │ │▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ │
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

### Step 1: Cluster Orchestration (Rancher & RKE2)

Use **Rancher** and **RKE2** to manage the distribution. This ensures the **NVIDIA GPU Operator** is deployed correctly to handle driver lifecycle and container runtimes across all nodes.

### Step 2: The Job Coordinator (Kueue)

Kubernetes’ default scheduler is "first-come, first-served," which leads to resource starvation in a multi-user cluster.

* **Kueue:** This is a cloud-native job queuing controller. It allows you to define **ResourceFlavors** (e.g., H100 nodes vs. A100 nodes) and **ClusterQueues** with specific quotas.
* **Why it matters:** If Researcher A submits a job that requires 64 GPUs but only 32 are available, Kueue holds the job in a pending state until the resources are free, rather than letting Kubernetes pods fail repeatedly.

### Step 3: The Entry Point (SkyPilot + Transformer Lab)

* **SkyPilot:** Acts as the high-level scheduler. It submits jobs to the **Kueue** interface.
* **Transformer Lab:** Provides the user-facing GUI for notebook and job management.

### Step 4: The Storage Strategy

Maintain a "Two-Tier" approach: **Hot Tier** (NFS/JuiceFS) for active training data and **Cold Tier** (S3/MinIO) for long-term checkpoint persistence.

---

## 3. Hardware Resilience

In a 100-node environment, hardware failure is a statistical certainty. At this scale, the Mean Time Between Failures (MTBF) necessitates a plan for GPU Xid errors, ECC bit-flips, and network disconnects.

### The Solution: Managed Recovery

The combination of **SkyPilot** and **Kueue** automates recovery:

1. **Detection:** SkyPilot monitors job status. If a node fails and the pod terminates, SkyPilot (working with Kueue) re-enqueues the job.
2. **Auto-Restart:** Once a healthy node is available, the job is automatically re-provisioned.
3. **Requirement:** Training scripts must be configured to save and load checkpoints from the shared storage tier to ensure continuity.

> [!TIP]
> **Preemption Support:** Kueue allows you to implement priority classes. High-priority training jobs can "preempt" lower-priority dev work, ensuring critical deadlines are met without manual intervention.

---

## 4. Advanced Considerations

### Advanced Storage Solutions

* **JuiceFS:** Recommended for datasets with millions of small files. It provides a POSIX-compliant interface on top of Object Storage.
* **Longhorn:** Provides distributed block storage for persistent volumes that require high-speed local performance with data replication.

### Observability & Networking

* **Monitoring:** Deploy **Prometheus** and **Grafana** with the `NVIDIA DCGM Exporter` for real-time visibility into GPU health and utilization.
* **Networking:** For multi-node training (32+ GPUs), use the **SR-IOV CNI plugin** to expose **RoCE** or **InfiniBand** directly to pods, minimizing latency during gradient synchronization.