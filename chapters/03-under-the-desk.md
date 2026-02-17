# The "Under-the-Desk" Server (Single Node, Multiple Users)

**The Scenario:** Your team has a single, powerful workstation—perhaps a 4x A6000 or an 8x H100 "DevBox"—tucked under a desk or in a corner of the office.

**The Challenge:** Multiple researchers need to run experiments. Without a system in place, "Researcher A" might start a training run that consumes all VRAM, causing "Researcher B's" debugging session to crash instantly with a `CUDA: Out of Memory` error. Worse, someone might upgrade a system-level driver that breaks everyone else's environments.

<img src="./images/configurations/2-multuser-single-workstation.png" width="180">

---

### 1. Avoid Running on "Bare Metal"

It's usually a bad idea to allow researchers to install libraries (`pip install ...`) directly on the host operating system. This leads to **Dependency Hell**, where one project requires specific version of CUDA, or someone needs Python 3.10 and another requires 3.12, eventually rendering the machine unusable.

**The Golden Rule:** The host OS should only have three things installed:

1. **NVIDIA/GPU Drivers**
2. **Docker / NVIDIA Container Toolkit**
3. **The Orchestrator (SkyPilot)**

Everything else—Python, PyTorch, CUDA libraries live inside a container or a virtual environment managed by the orchestrator.

If you really can't or don't want to use containerization, then using **uv environments** is an alternative that will at least separate out Python dependencies but won't prevent users from altering drivers or base components.

---

### 2. Using SkyPilot + k3s

While you *could* manually assign GPUs using `CUDA_VISIBLE_DEVICES=0,1`, this requires human coordination (e.g., a "GPU-Use" Slack channel). Instead, in this guide, we'll use **SkyPilot** to turn the workstation into a private, automated cloud.

SkyPilot acts as the "Traffic Controller." It handles the queue: if the GPUs are full, the next job waits until a resource is freed.

#### Why this works:

* **Virtualization:** It uses **k3s**. This provides the isolation of a cluster without the pain of managing a more complex Kubernetes installation.
* **Zero-Config for Researchers:** Researchers define their requirements in a simple YAML file (e.g., `accelerators: A100:1`). They don't need to know which GPU index is free.
* **Cloud Parity:** The exact same command used to run a job locally (`sky launch`) can be used to run the job on AWS or GCP later.



### 3. The Interface: Transformer Lab

Transformer Lab works with SkyPilot to make the entire research experience easier.

* **The Hub:** Install Transformer Lab as a persistent service on the machine.
* **Job Submission:** Researchers log into the web interface, select their model and dataset, and click "Run."
* **Queue Management:** Transformer Lab passes the job to SkyPilot. If Researcher A is using all GPUs, Researcher B sees their job marked as "Queued" in the dashboard, preventing a collision.


### 4. Handling Data & Artifacts

On a single node, you have the luxury of fast, local NVMe storage. However, you should still act like a cluster:

* **Data Isolation:** Create a standard directory structure (e.g., `/data` and `/shared`).
* **Mounting:** When SkyPilot Lab launches a container, it "mounts" these directories. This ensures that even if the container is deleted after the run, the model weights and logs persist on the physical disk.
* **Transformer Lab Global Storage:** If you use Transformer Lab, it handles storing artifacts, models, and logs in a common place to make it easier for researchers to share and collaborate.


### The "Under-the-Desk" Pro-Tips

> [!TIP]
> **The Thermal Reality:** > Even a single 8x GPU server can exhaust the heat of a small office in under an hour. If the "Under-the-Desk" server is literally under a desk, the researcher sitting there will be miserable. Ensure there is at least 3 feet of clearance behind the fans and never block the intake.
> **The "Zombie" Problem:**
> Use SkyPilot's `sky status` command weekly to find "zombie" interactive sessions—containers that were started for debugging but never shut down, silently hogging 24GB of VRAM.


[Ready to set up a machine as a shared computer for multiple researchers? Read this Guide -->](./03-01-install.md)