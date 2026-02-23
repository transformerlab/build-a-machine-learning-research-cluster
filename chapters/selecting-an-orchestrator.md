# Selecting a GPU Scheduler for a Machine Learning Research Cluster

If you are building a new ML research cluster, the two most likely choices are **Slurm** and **SkyPilot**. Most teams should evaluate those two first, then consider alternatives if they have specific requirements.

---

## 1. The Two Most Common Choices: Slurm and SkyPilot

### Slurm

Slurm is the dominant scheduler in academic HPC and many enterprise on-prem environments.

**Where Slurm is strongest**

* Stable scheduling for fixed clusters with known users.
* Strong multi-user controls (partitions, priorities, fair-share, QoS, limits).
* Excellent fit for institutions that already run Slurm or need compatibility with existing HPC practices.
* Mature support for long-running batch workloads and strict policy enforcement.

**Operational tradeoffs**

* Primarily designed around static infrastructure.
* Cloud elasticity is possible but is not its default operating model.
* Setup and operations require familiarity with Slurm internals (controller, accounting, authentication, job policies).

### SkyPilot

SkyPilot is a modern resource and job launcher designed for cloud and hybrid environments.

**Where SkyPilot is strongest**

* Fast path to productive cluster usage with minimal scheduler administration.
* Dynamic provisioning across cloud providers and Kubernetes backends.
* Built-in support for cost-focused workflows (for example, spot instances and recovery).
* Good fit for teams that move between local nodes and cloud resources.

**Operational tradeoffs**

* Not a drop-in replacement for all Slurm policy controls in traditional HPC organizations.
* Some institutions with strict internal governance may still prefer Slurm’s long-established model.

### Slurm vs SkyPilot: Practical Comparison

| Dimension | Slurm | SkyPilot |
| --- | --- | --- |
| Typical starting point | Existing university or enterprise HPC cluster | New cloud-first or hybrid research platform |
| Resource model | Mostly static node pools | Dynamic provisioning + existing pools |
| Policy controls | Very strong native multi-tenant scheduling controls | Simpler at scheduler-policy layer; often paired with K8s policies |
| Cloud cost optimization | Possible, but not primary default | Core strength (spot + rescheduling patterns) |
| Day-2 operations | Mature but heavier HPC administration | Lighter to start; complexity depends on backend |
| Best initial fit | Labs with established HPC operations | Teams optimizing for velocity and cloud flexibility |

### Decision Guidance

Choose **Slurm** first if:

* You are integrating into an existing institutional HPC environment.
* You need strict fair-share, accounting, and policy controls that your org already understands.
* Your infrastructure is mostly on-prem and continuously available.

Choose **SkyPilot** first if:

* You are starting new and want one path across cloud and on-prem.
* You care about cloud cost controls and dynamic provisioning from day one.
* You want to reduce scheduler administration overhead while scaling quickly.

For many new labs, SkyPilot is the faster starting point. For many established HPC teams, Slurm remains the safest operational choice.

---

## 2. Other Alternatives Worth Evaluating

If Slurm and SkyPilot do not fit your constraints, the next set of options is usually one of the following.

### dstack

dstack provides an ML-focused workflow for running development and training tasks across cloud and on-prem resources. It avoids the complexity of Kubernetes.

### Kubernetes + Kueue Directly

This is the most flexible cloud-native path when you want direct control.

**What this gives you**

* Native Kubernetes operations and ecosystem compatibility.
* Queueing, admission, and quota control via Kueue.
* Fine-grained integration with GPU plugins, storage, networking, and observability stacks.

**What it costs**

* More platform engineering work than Slurm-only or SkyPilot-first setups.
* You must design your own developer-facing abstractions and workflows.

This is often the right choice for larger internal platform teams that already run Kubernetes at scale.

SkyPilot also works on top of Kueue + Kubnernetes, so this is the configuration we recommend for larger labs.

### Kubeflow

Kubeflow provides a broad ML platform on top of Kubernetes, including pipelines, notebook patterns, and training components.

**When to evaluate it**

* You want an integrated Kubernetes-native ML stack, not only a scheduler.
* You need end-to-end platform components under one ecosystem.

**Watch for**

* Higher operational complexity compared with thinner scheduler-oriented layers.
* Need for clear ownership: Kubeflow is a platform commitment, not only a job queue decision.

---

## 3. Different Paradigm: Workflow Orchestrators and Control-Plane-First Systems

Some teams choose a different paradigm entirely: treat the scheduler as a low-level resource layer and put a workflow control plane on top.

### Flyte

Flyte is a workflow orchestration system focused on reproducible, typed, and versioned ML/data workflows.

**Why teams pick it**

* Strong workflow abstraction and reproducibility model.
* Good for production-grade ML/data workflows with lineage and governance needs.

**What changes in architecture**

* Scheduling becomes one part of a larger workflow runtime.
* Teams invest in workflow definitions and platform conventions, not only queue setup.

### Other systems in this paradigm

* **Argo Workflows**: Kubernetes-native DAG execution; common in K8s platform teams.
* **Prefect**: Python-friendly orchestration with strong developer ergonomics.
* **Dagster**: Asset-centric orchestration model; useful when data asset management is central.
* **Apache Airflow**: Widely used orchestrator for data/ML pipelines, especially where existing data engineering standards already rely on it.
* **Metaflow**: Data-science-oriented workflow framework focused on developer experience and portability.

These are not direct equivalents to Slurm. They are higher-level orchestration systems that usually sit above Kubernetes, cloud APIs, or other compute backends.

---

## 4. Recommended Evaluation Order

For most research clusters, this sequence minimizes rework:

1. Decide first between **Slurm** and **SkyPilot** based on your operating model (institutional HPC vs cloud/hybrid velocity).
2. If neither is a fit, evaluate **dstack** or **Kubernetes + Kueue** based on your platform engineering capacity.
3. If your real need is workflow governance and reproducibility, evaluate **Flyte** or related workflow orchestrators as a higher-level control plane.

This approach keeps the decision grounded in operational reality: who runs the platform, where compute lives, and how strictly multi-tenant policy and cost controls must be enforced.
