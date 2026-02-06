# Chapter 4: The "Apple Silicon" Shop

**The Scenario:** Your lab is stocked with Mac Studios or high-end MacBook Pros. You want to pool these machines together so a researcher can "send" a job to an idle Mac in the corner rather than heating up their own laptop.

**The Challenge:** Apple Silicon is a masterpiece of hardware, but it is a "citizen of a different world." Applying traditional Linux-centric patterns to macOS results in the "Metal Air Gap."

---

## 1. Do Not Use Kubernetes

Kubernetes is a Linux-native technology. To run it on macOS, you must use a Virtual Machine (via Docker Desktop, OrbStack, or QEMU). Because of this abstraction, your GPUs will sit idle.

* **The Container Barrier:** Kubernetes uses pods, and there is currently no way to expose Apple Silicon (Metal) to those pods through the VM layer.
* **No Device Plugin:** In the NVIDIA world, a Pod requests `nvidia.com/gpu: 1`. There is no equivalent `apple.com/gpu` plugin because the Linux VM is blind to the host's Metal API for compute tasks.

To use the GPU on Apple Silicon, the process must run **natively on macOS**. You need an orchestrator that bypasses the container layer and talks directly to the host OS.

---

## 2. Option 1: The Modern Choice (dstack)

For most ML teams, **dstack** is the best way to pool Apple Silicon devices. It provides a "Cloud UX" without forcing you into a containerized overlay.

* **The Mechanism:** You run a `dstack` server on one machine and configure your fleet of Macs as an **"SSH Fleet."**
* **The Magic:** dstack SSHs into the Mac and runs your workload directly on the host shell. This grants your code direct, native access to **MPS (Metal Performance Shaders)**.
* **Research Workflow:** Since dstack handles the environment and job lifecycle, it is the primary tool for fine-tuning and experimental runs.

---

## 3. Option 2: The "Inference Hive" (Exo)

While not a primary research tool for training, your lab may want to run **Exo** ([exolabs.net](https://exolabs.net/)) to utilize the combined RAM of your fleet for large-scale inference.

* **The Mechanism:** Exo is a peer-to-peer distributed inference engine that allows you to run models (like Llama-3 70B) by partitioning them across multiple Macs over your local network.
* **Integration:** You can leverage Exo through **dstack** as a backend. This allows researchers to use dstack's familiar job-scheduling interface while taking advantage of Exo’s ability to "stitch" together the memory of multiple Macs into one giant virtual GPU.
* **Best For:** When the lab needs a shared, high-capacity inference endpoint for testing or local model evaluation.

---

## 4. Option 3: The HPC Choice (Slurm)

If you are running a formal university lab where "Fair Share" scheduling and multi-user priority queues are vital, **Slurm** remains the heavyweight option.

* **The Mechanism:** You compile Slurm from source using Homebrew dependencies.
* **The Magic:** You create a partition in `slurm.conf` called `apple_gpu`. Because the job runs as a native macOS process, it has full access to the unified memory pool.

---

## 5. Summary: Choosing Your Tool

| Feature | **dstack** | **Exo** | **Slurm** |
| --- | --- | --- | --- |
| **Primary Use** | Orchestration & Job Queuing | Distributed "Giant" Inference | Multi-user Resource Fairness |
| **GPU Access** | Native (via SSH) | Native (via Metal/p2p) | Native (via Process) |
| **Setup Time** | 15 Minutes | 2 Minutes | 2-4 Hours |
| **Research Utility** | High (Fine-tuning/Training) | Moderate (Inference only) | High (Resource Management) |

---

## The Verdict & Recommendation

Apple Silicon machines are **Inference Monsters**. Their high memory bandwidth makes them faster than many mid-range NVIDIA cards for large model weights.

* **Start with dstack.** It turns your office Macs into a private "Apple Cloud" with almost zero configuration and full support for training workflows.
* **Layer in Exo via dstack** when you need to run massive models that exceed the RAM of a single machine.
* **Switch to Slurm** only if you need to enforce strict "wait your turn" policies for a large department.

> [!TIP]
> **Network is the Bottleneck:** For both Exo and dstack remote jobs, your WiFi will be the bottleneck. For the best performance, plug all your Macs into a **10GbE Switch**.

---

**Would you like me to move on to Chapter 5: The "University Cluster" (10–100 Nodes) and tackle the networking complexities of a larger rack?**