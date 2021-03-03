---
title: Migrating Kubernetes cluster with zero downtime
date: "2021-03-03T14:48:32.169Z"
template: "post"
draft: false
slug: "migrating-kubernetes-cluster-with-zero-downtime"
category: "Kubernetes"
tags:
  - "Azure"
  - "AKS"
  - "Kubernetes"
  - "Migration"
  - "LetsEncrypt"
description: "When a company decides to use Kubernetes clusters in production and maintain high level of availability/SLA, sooner or later time will come to perform a full cluster migration with zero downtime. There are many reasons why a full cluster migration may be required. For example moving to a new hosting platform, implementing zone-redundancy for worker nodes, changing cluster configuration to use managed identity (as opposed to managing the service principal yourself) or simply performing a risky cluster upgrade (AKS 1.17 -> 1.18 changes Ubuntu OS version, 1.18 -> 1.19 changes default container runtime interface). Doing a zero-downtime migration is definitely possible, but requires careful planning."
socialImage: "../../media/kubernetes-migration.jpg"
---

When a company decides to use Kubernetes clusters in production and maintain high level of availability/SLA, sooner or later time will come to perform a full cluster migration with zero downtime. There are many reasons why a full cluster migration may be required. For example moving to a new hosting platform, implementing zone-redundancy for worker nodes, changing cluster configuration to use managed identity (as opposed to managing the service principal yourself) or simply performing a risky cluster upgrade (AKS 1.17 -> 1.18 changes Ubuntu OS version, 1.18 -> 1.19 changes default container runtime interface). Doing a zero-downtime migration is definitely possible, but requires careful planning.

- [Cluster design with migration in mind](#cluster-design-with-migration-in-mind)
- [Abstract deployment credentials](#abstract-deployment-credentials-from-code)
- [Transfer LetsEncrypt certificates](#transfer-letsencrypt-certificates)
- [End to end testing prior to migration](#end-to-end-testing-prior-to-migration)
- [Flipping the traffic manager switch](#flipping-the-traffic-manager-switch)
- [Post migration testing and clean up](#post-migration-testing-and-clean-up)
- [Summary](#summary)

## Cluster design with migration in mind
This article assumes all resources are hosted in Azure, CI/CD pipelines run via Azure DevOps services and TLS encryption is built around Let's Encrypt certificates, but the general principles apply with any other set of tools or environment.

It may sound counter-intuitive, but it is imperative to design the architecture with migration in mind from day 1. You will generally want to move all components that handle traffic redirection away from your cluster. It may be tempting to implement everything in-cluster with service meshes, but this area evolves fast. What may be a good mesh today is possibly an absolete concept tomorrow. Few things are worse than relying on a service mesh for migration, when it is the service mesh solution itself that is being replaced by migration.

It is therefore a good idea to rely on a core networking concept, like DNS, or another type of front-door service, to serve as an entry point into the cluster:

1. Leveraging DNS for migration

![AKS migration with DNS](/media/aks-migration-traffic-manager.jpg)

Traffic Manager in Azure is basically an abstracted DNS resolution service. The clear advantage here is that the request payload doesn't flow through the traffic manager. The actor checks with the traffic manager, as it would with any other DNS request, and traffic manager tells it where to send the actual payload. No need to solve encryption, high-availability or throughput issues. The disadvantage: It is entirely up to the actor how often it checks with the traffic manager. Checking too often will cause additional latency. Checking once an hour could cause traffic to go to the old destination for up to an hour when we perform a migration. Traffic Manager gives an indication to the actor how often it should be checking via TTL (time to live) setting, but it is up to the actor to decide if it will respect that. I have come across clients that have decided to simply cache a DNS record indefinitely.

2. Leveraging API Management/front-door for migration

![AKS migration with DNS](/media/aks-migration-front-door.jpg)

Handling traffic routing via a front-door rather than DNS is preferrable. However, as this will most likely operate on layer 6 or 7, it is more expensive and we have to maintain high availability, sufficient throughput for the client payload and implement TLS termination. Using a front-door for the sole purpose of making migration possible at some point in the future is simply not feasible. If your organisation already uses a front-door outside of the Kubernetes cluster for other reasons, it may be a good place to put the logic that decides which cluster to route traffic to.

## Abstract deployment credentials

Another important aspect of migration (especially if you plan to migrate large number of services) is to make it easier for CI/CD pipelines to deploy to the new cluster. If you use Azure DevOps for all CI/CD purposes and make use of pipelines-as-code defined in YAML (also known as multi-stage pipelines), make sure not to hard-code cluster details into the pipelines. Make use of [Azure DevOps environments](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops) instead. The pipeline yaml defintion will only store the environment name while cluster details will be abstracted to the environment as a resource.

## Transfer LetsEncrypt certificates

Transferring Let's Encrypt certificate from live cluster to the migration candidate is only required if you use DNS for switching traffic and both clusters use the same ingress host. The production candidate will not be able to complete the HTTP-01 challenge and issue new certificates until the switch takes place and all traffic is routed to the new cluster. This will inevitably cause downtime, which can range from few seconds to several hours (if Let's encrypt is not configured properly on the new cluster). It also means you are effectively unable to test services on the new cluster, because end-to-end encryption is not possible without the TLS certificate.

It is possible to transfer certificates issued by Let's Encrypt manually between clusters. There is nothing inherently restrictive in the certificate itself that ties it with the cluster it was issued on. The process is not very straight-forward, so I won't go into details here, but I'll write a separate article on it at a later date.

## End to end testing prior to migration

When using DNS for migration, end-to-end testing can be done by overriding DNS resolution locally (via `C:\Windows\System32\drivers\etc\hosts` file on Windows). Simply point a DNS record of the host at the IP address of the ingress controller in the new cluster. This will also test end-to-end encyption prior to live traffic hitting the new cluster.

When using a front-door/proxy/API management, you will need to set up a policy rule which redirects traffic to the new cluster based on certain conditions (for example a special header in the request that only you know about). 

## Flipping the traffic manager switch

If you're satisfied everything on the new cluster works as expected, it is time to flip the switch. The aim here is to achieve zero-downtime, so make sure to perform end-to-end testing and check TLS encryption works prior to sending live traffic to the new cluster.
Services like [Datadog](https://www.datadoghq.com/) offer synthetic tests that will continually hit a specific endpoint from multiple locations and alert you if you suffer from downtime.

## Post migration testing and clean up

Once the migration is complete, it is advisable to keep the old cluster running for at least couple of hours. You can then stop the cluster completely (using [az aks stop](https://docs.microsoft.com/en-us/azure/aks/start-stop-cluster) in Azure) and keep it around for few more weeks just in case a late rollback is needed.

## Summary

It's definitely possible to perform Kubernetes migrations with zero downtime, but early planning is essential. Choosing the right technology for proxy/front-door, or simply slapping a DNS based traffic manager in front of a cluster, should be the first thing on architect's mind when considering moving production workloads to Kubernetes.