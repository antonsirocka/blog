---
title: Optimal VM for building code?
date: "2020-02-27T22:40:32.169Z"
template: "post"
draft: false
slug: "optimal-vm-for-building-code"
category: "Azure"
tags:
  - "Azure"
  - "VM size"
  - "build times"
  - "F-series"
description: "Every company runs into this challenge sooner or later. How to optimally build growing code base as efficiently as possible. Azure offers VMs with varying CPU families. This article attempts to answer which one provides the best value for money."
socialImage: "media/build-times-xeon.jpg"
---

Every company runs into this challenge sooner or later. How to optimally build growing code base as efficiently as possible. Azure offers VMs in varying CPU families. This article attempts to answer which one provides the best value for money.

- [The test setup](#the-test-setup)
- [Testing methodology](#testing-methodology)
- [Results](#results)
- [Conclusions](#conclusions)

## The test setup

The test will consist of 4 different configurations. Prices are approximate for West Europe Azure region at the time of writing this document.

- B2ms (2 vcores, burstable) - £52.23/month
- B4ms (4 vcores, burstable) – £104.46/month
- F4s_v2 (4 vcores, compute optimised) - £105.55/month
- D4s_v3 (4 vcores, general purpose) - £130.58/month

`B-series` are cost-efficient VMs suitable for workloads that don't require high network/disk throughput. When they're not fully utilised they will bank up CPU credits and utilise them later when needed. This makes them an attractive candidate for build agents (build up a credit bank overnight and utilise the computing resources during the day). More info on these VMs is [here](https://azure.microsoft.com/en-gb/blog/introducing-b-series-our-new-burstable-vm-size/).

`Fsv2-series` VMs run on Intel Xeon Platinum CPUs with high clock speeds and performance in floating point operations. They come with lower memory allocation than burstable VMs, but this is not necessarily a problem for build agents. They are also priced very competitevly. More info on these bad boys [is here](https://docs.microsoft.com/en-us/azure/virtual-machines/fsv2-series).

`D4s_v3` are suitable for most general purpose workloads. They come with a higher memory allocation than F-series VMs. They are classed as a good all-around VMs making them an attractive default choice for users who are just trying things out. Optimisation tends to be an after-thought, which rarely materialises. More info on Dsv3 series [here](https://docs.microsoft.com/en-us/azure/virtual-machines/dv3-dsv3-series).

## Testing methodology

The build process will run on a self-hosted Azure DevOps build agents in AKS (Azure Kubernetes Service). A separate worker node pool is created for each VM size as follows:

![build-times-k8s-pools.jpg](/media/build-times-k8s-pools.jpg)

The resource requests and limits for Kubernetes pods hosting the build agents are set up as follows:

![build-times-k8s-limits.jpg](/media/build-times-k8s-limits.jpg)

 I won't go into details about the nature of the code being built or the steps in the build process to keep this short and focused. But it does involve the sort of stuff you'd normally expect: Restoring nuget packages, running "dotnet build", unit testing and pushing the final image to ACR (Azure Container Registry).

## Results

Each build run is executed twice. The first run is noticably longer due to docker images being pulled from the internet. These images are then cached for the 2nd run. I've also timed the compile time and overall build time separately to get a better indication how the CPU affects the compilation part of the process. 

### B2ms

- Overall build time: 15m 19s
- Compile task: 12m 47s
- CPU profile:
 ![build-times-b2ms.jpg](/media/build-times-b2ms.jpg)

 ### B4ms

- Overall build time: 9m 27s
- Compile task: 8m 41s
- CPU profile:
![build-times-b4ms.jpg](/media/build-times-b4ms.jpg)

### F4s_v2

- Overall build time: 7m 28s
- Compile task: 6m 54s
- CPU profile:
![build-times-f4s.jpg](/media/build-times-f4s.jpg)

### D4s_v3

- Overall build time: 8m 41s
- Compile task: 8m 3s
- CPU profile:
![build-times-d4s.jpg](/media/build-times-d4s.jpg)

### Memory consumption profile

The profile was very similar across all tested configurations. I've therefore only included a single graph. 

![build-times-memory-allocation.jpg](/media/build-times-memory-allocation.jpg)

The profile suggests an AKS node needs between 1 – 2.5GB of memory to host a single build agent. B4ms and D4s_v3 both come with 16 GB of RAM, but 8GB is sufficient to host 2 agents on a single node. The additional RAM is therefore not utilised.

### Disk performance

All tests were done on nodes utilising a single 127GB premium disk (100MBps max throughput). I haven’t tested using a larger disk with higher throughput, because the throughput here will be limited by VM size. D4s_v3 and F4s_v2 have a maximum throughput of 96MBps. 

## Conclusions

The measured results suggest `F4s_v2` is the optimal SKU to achieve lowest build times/cost ratio. Hosting 2 agents on each node appears to be a balanced approach. F4s_v2 is also the lowest SKU that supports accelerated networking (you can read more about accelerated networking [here](https://azure.microsoft.com/en-gb/blog/maximize-your-vm-s-performance-with-accelerated-networking-now-generally-available-for-both-windows-and-linux/)).

Utilising general-purpose D4s_v3 nodes provides similar peformance, but at significantly higher price due to the additional memory, which is not utilised.

One may consider de-allocating VMs in out-of-office hours and starting them again first thing in the morning. However B-series VMs are not suitable for this, because they rely on banked CPU credits, which are lost when a VM is de-allocated. I would therefore recommend to either use Fs_v2 series VMs and automate the power-management or consider reserving the compute capacity for 3 years, providing significant cost saving. Any power management implementation will come with operational overheads and in both cases the cost savings are similar.