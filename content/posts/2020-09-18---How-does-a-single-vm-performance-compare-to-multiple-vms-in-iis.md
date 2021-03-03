---
title: How does a single VM performance compare to multiple VMs? (Windows Server/IIS)
date: "2020-09-18T22:40:32.169Z"
template: "post"
draft: false
slug: "how-does-a-single-vm-performance-compare-to-multiple-vms-in-iis"
category: "Performance"
tags:
  - "Azure"
  - "Virtual Machines"
  - "Performance"
  - "Load Balancing"
description: "Having you ever wondered if 2 VMs would provide the same level of performance in IIS as a single VM of double the size? The following analysis will attempt to answer the question."
socialImage: "../../media/single-vm-vs-two-vms.jpg"
---

Having you ever wondered if 2 VMs would provide the same level of performance in IIS as a single VM of double the size? The following analysis will attempt to answer the question.

- [The argument for multiple VM instances](#the-argument-for-multiple-vm-instances)
- [Infrastructure setup](#infrastructure-setup)
- [Performance test with a single VM instance](#performance-test-with-a-single-vm-instance)
- [Performance test with two VM instances](#performance-test-with-two-vm-instances)
- [Conclusions](#conclusions)

## The argument for multiple VM instances

Running multiple instances of identical virtual machines is nearly always a good idea and became de-facto a standard in modern infrastructures running on Kubernetes. The concept of `cattle vs pets` describes these identical nodes as disposable `cattle`. We are encouraged to kill off nodes at will and replace them very quickly. Thus, increasing the resiliency of the infrastructure. On the other hand, once we start making custom modifications on individual nodes, they will slowly turn into `pets`. Something we need look after, care for and under no circumstances want to lose.

In Kubernetes the `cattle` concept is enforced from the ground up. However, most production workloads these days still run on IIS or similiar web servers sitting behind a load balancer. All infrastructure engineers have come face-to-face with such setup at some point and the inevitable questions come up: How to size up virtual machine to get optimal performance? Is it better to use a large number of small instances or a small number of large instances?

### Advantages of running many small instances
- Higher resiliency to failures. For example, if 1 out of 10 instance shuts down, 90% of the infrastructure resources remain unaffected. Compared that to a case where 1 out of 4 instances goes down, only 75% of infrastructure remains up and running.
- Potentially better resource utilisation as the increments are smaller. Not necessarily always the case though, as shown below.
- Potentially higher disk throughput. For example 5 separate premium 128 GiB SSDs will provide a much higher throughput than a single 640 GiB disk.

### Disadvantages of running many small instances
- Baseline resource wastage - every instance has to dedicate a portion of resources for operating system and any kernel operations. 
- Spinning up new instances is time consuming - new instances need to be configured and software packages deployed before the instance can serve traffic. A lot of this can be automated, but it does introduce another potential point where issues can creep up.

There tends to be a sweetspot and gut feeling tells us that running 8 instances with a single vcore is not going to provide optimal results. If you run Windows Server 2016+ and IIS, the minimum you should consider for an infrastructure handling large volume of traffic is a VM instance with 4 vcores. Consider your memory requirements to decide between a memory-optimised or a general-purpose instance.

The difference in real IIS performance between a single 8 vcore VM and two 4 vcores VMs however is not so obvious. I've run some performance tests using JMeter in an attempt to answer this question.

## Infrastructure setup

The infrastructure setup consists of an Azure API management instance, followed by an application gateway that serves as a load balancer and TLS termination point. The traffic is load balanced to a virtual machine scaleset, which contains either 1 or 2 VM instances (in this case a DS12_v2 size). I've used two JMeter profiles. One that generates a medium level of traffic (load profile A). I've then created a second load profile B, which generates traffic levels sufficient enough to saturate the instances to a point when service degradation could occur.

Infrastructure setup with a single VM instance:
![2-vm-configuration.jpg](/media/2-vm-configuration.jpg)

Infrastructure setup with a two VM instances:
![1-vm-configuration.jpg](/media/1-vm-configuration.jpg)

## Performance test with a single VM instance

### Load profile A

![1-instance-load-profile-A.jpg](/media/1-instance-load-profile-A.jpg)

![1-instance-tps-load-profile-A.jpg](/media/1-instance-tps-load-profile-A.jpg)

![1-instance-response-times-load-profile-A.jpg](/media/1-instance-response-times-load-profile-A.jpg)

### Load profile B

![1-instance-load-profile-B.jpg](/media/1-instance-load-profile-B.jpg)

![1-instance-tps-load-profile-B.jpg](/media/1-instance-tps-load-profile-B.jpg)

![1-instance-response-times-load-profile-B.jpg](/media/1-instance-response-times-load-profile-B.jpg)

## Performance test with two VM instances

### Load profile A

![2-instances-load-profile-A-1.jpg](/media/2-instances-load-profile-A-1.jpg)
![2-instances-load-profile-A-2.jpg](/media/2-instances-load-profile-A-2.jpg)

![2-instances-tps-load-profile-A.jpg](/media/2-instances-tps-load-profile-A.jpg)
![2-instances-response-times-load-profile-A.jpg](/media/2-instances-response-times-load-profile-A.jpg)

### Load profile B
![2-instances-load-profile-B-1.jpg](/media/2-instances-load-profile-B-1.jpg)
![2-instances-load-profile-B-2.jpg](/media/2-instances-load-profile-B-2.jpg)

![2-instances-tps-load-profile-B.jpg](/media/2-instances-tps-load-profile-B.jpg)
![2-instances-response-times-load-profile-B.jpg](/media/2-instances-response-times-load-profile-B.jpg)

## Conclusions

In conclusion, it looks like there indeed isn't much difference in terms of raw IIS performance between the two setups.
The measured response times, transactions per second, CPU profiles and process queue lengths are very similar. The configuration with 2 VMs produced fewer responses times between 500ms and 1500ms, but a higher number over 1500ms. The configuration with a single VM on the other hand produced more responses between 500ms and 1500ms, but very few over 1500ms.

Therefore, the IIS performance should not be the deciding factor when choosing the size of a VM. The performance seems to scale reasonably well and the decision should instead be focused around overall infrastructure resiliency requirements and how quickly and reliably can new instances be spun up when your infrastructure really needs them.