# The Definitive Guide to Building a Machine Learning Research Platform 🚀

![Cover Image](./chapters/images/cover.png)

This guide is for **academic institutions and research labs** ready to establish their first cohesive machine learning research platform. We’ve designed these chapters for the IT directors, principal investigators, and system architects tasked with building a unified environment where every researcher—from the first-year student to the seasoned postdoc—can train models efficiently. Whether you are starting with a single "under-the-desk" GPU server or scaling to a university-wide cluster, this guide provides the technical blueprints and strategic philosophy needed to turn raw hardware into a world-class discovery engine.

In this guide, we are not attempting to list every possibility but, rather, to offer the most common "tried and tested" configurations, with a bias toward modern, simple tooling that is open source and easy to maintain.

> **Note:** This is a "living book" written on GitHub. We are looking for contributions from industry and academic experts to make recommendations. Use the links below to navigate the chapters. Found a mistake? Open an **Issue** or submit a **Pull Request**.


## Table of Contents

### Background

* **[Chapter 1: Philosophy & Components](chapters/01-foundation.md)**
    * Defining the stack: Drivers, Orchestration, and Storage.

### Configurations

<img src="./chapters/images/configurations/1-singleuser.png" width="100">

* **Chapter 2: The Single User AI Workstation (1 Node, 1 User)**
    * 2.1 [Overview and Recommendations](chapters/02-single-user-single-workstation.md)
    * 2.2 Step-by-Step OS Installation:
        * [Ubuntu 22.04 Server with CUDA Support](/chapters/02-02-01-ubuntu-cuda.md)
        * [Ubuntu with AMD ROCm Support](/chapters/02-02-02-ubuntu-rocm-amd.md)
        * [Which Mac to Buy for Machine Learning]( /chapters/02-02-03-macos.md)

<img src="./chapters/images/configurations/2-multuser-single-workstation.png" width="100">

* **Chapter 3: The "Under-the-Desk" Server (1 Node, Multiuser)**
    * 3.1 [Overview and Recommendations](chapters/03-under-the-desk.md)
    * 3.2 [Step-by-Step Install Instructions](/chapters/03-01-install.md)


<img src="./chapters/images/configurations/3-multiusers-multi-workstation.png" width="100">

* **Chapter 4: The "Closet Cluster" (2–5 Nodes)**
    * [Overview and Recommendations](chapters/03-closet-cluster.md)
    * Step-by-Step Install Instructions (Coming Soon)


* **[Chapter 5: The "Mac Silicon Cluster"](chapters/05-apple-silicon.md)**

<img src="./chapters/images/configurations/5-largecluster.png" width="150">

* **[Chapter 6: The "University Cluster" (10–100 Nodes)](chapters/06-supercomputer.md)**
* **Chapter 7: The "Supercomputer" (1000+ Nodes)** | Coming Soon

<img src="./chapters/images/configurations/4-single-cloud.png" width="100">

* **Chapter 8: Cloud Compute Cluster** | [Read Chapter](chapters/07-cloud-cluster.md)

<img src="./chapters/images/configurations/6-hybrid.png" width="150">


* **Chapter 9: Hybrid Cloud Cluster** | Coming Soon

* **Chapter 9: The Air-Gapped Systems** | Coming Soon