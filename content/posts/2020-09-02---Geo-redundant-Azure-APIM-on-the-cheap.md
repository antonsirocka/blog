---
title: Building geo-redundant Azure API Management on a budget
date: "2020-09-02T22:40:32.169Z"
template: "post"
draft: false
slug: "buiding-geo-redundant-azure-apim-on-a-budget"
category: "Azure"
tags:
  - "Azure"
  - "API Management"
  - "Geo redundancy"
  - "Cost efficiency"
description: "Azure API management is a useful product in Azure offering, but the pricing structure makes it unattractive for building low cost geo-redundant infrastructure with no single point of failure."
socialImage: "../../media/apim-geo-redundancy-on-budget.jpg"
---

Azure API management is a useful product in Azure offering, but the pricing structure makes it unattractive for building low cost geo-redundant infrastructure with no single point of failure.

- [What is Azure API Management for](#what-is-azure-api-management-for)
- [The price of managed geo-redundancy](#the-price-of-managed-geo-redundancy)
- [DIY geo-redundancy](#diy-geo-redundancy-yourself)
- [Solving the synchronisation issue](#solving-the-synchronisation-issue)
- [The unintended benefits of building your own geo-redundancy](#the-unintended-benefits-of-building-your-own-geo-redundancy)

## What is Azure API Management for

Azure API management is a PaaS (platform-as-a-service) product in Azure designed to sit in front of your main API cluster/servers. It provides useful features such as:
- detailed analytics data
- authentication/athorisation
- hide sensitive APIs while exposing only the ones you want (we all had that one API we didn't want anyone to see, but were too lazy to secure it properly)
- out-of-the-box developer portal giving users overview of all API operations
- API management is `not` a load balancer, use Azure Front Door or Application Gateway for load balancing

## The price of managed geo-redundancy

One of the drawbacks of API management is the relative cost of a multi-regional deployment. If full geo-redundancy is required for your infrastructure, your only option is to use the highest `premium tier`, which costs an eye-watering £2083/month (in West Europe region). Not only that, you'll need to double this figure, since you need a premium instance in two separate regions. If you don't need other premium features, such as handling 8000 request/sec (2 x 4000) or a virtual network support, you'll be sending a lot of money to Microsoft every month without getting much in return.

The `basic tier` still handles around 1000 request/sec, which is more than sufficient for most solutions and costs just over £100/month. So, can you actually build the geo-redundancy into API management yourself? Yes. The only problem you have to solve is synchronisation of your APIs, users subscriptions and products across the instances.

## DIY geo-redundancy

Assuming the infrastructure consists of a "blue" and "green" side with an API management instance sitting in front of it, the diagram would look like this:

![apim-bluegreen.jpg](/media/apim-bluegreen.jpg)

After introducing a second API management instance in WestUK region and a traffic manager, the diagram would look like this:

![apim-geo-redundant-bluegreen.jpg](/media/apim-geo-redundant-bluegreen.jpg)

The thing to bear in mind is: Azure Traffic Manager is a DNS-based routing tool. It is capable of switching traffic manually between the API Management instances, however a more common configuration utilises a health probe. In a "priority" routing configuration, the decision to route traffic to an instance is done based on the endpoint's priority and health status. Thefore it is useful to have a root API operation configured within API Mananagement, which has a single responsibility - to serve a `200 OK` response to the traffic manager probe. Here's a sample configuration for such an API operation:

![apim-health-probe-api.jpg](/media/apim-health-probe-api.jpg)

We need the response to be served by API Management instance itself without involving our backend infrastructure (unlike normal API operations which would call the backend infrastructure when serving incoming requests), so we also need to create a simple policy for this API operation to facilitate the behaviour:

![apim-health-probe-policy.jpg](/media/apim-health-probe-policy.jpg)

This policy takes any inbound request and responds with a 200 (OK) response. It also adds an "instance name" header with a value of "weu-apim". This is useful if we need to send a request to the traffic manager ourselves to identify where the traffic is being routed. The response header will indicate the response was served by API management instance in West Europe (weu).

If there is a regional outage, or the API management instance becomes unvailable for any other reason (deleted by mistake or intentionally by an intruder), the traffic manager will become aware of this via the health probe and can automatically route traffic to the second instance with lower priority.

## Solving the synchronisation issue

The hardest part about doing DIY geo-redundancy yourself is setting up the synchronisation between multiple API Management instances. A company that goes through the effort of building a fully geo-redundancy infrastructure would be expected to following IaC (infrastructure-as-code) practices. There should be no manual changes done to APIs, products or policies within an API Management instance. Everything should be defined in code (terraform or similar). Management of users and subscriptions/api keys should also be done by an external micro-service, which needs to send the same request to multiple instances. For example, if a user "Adam" is created on API Management instance 1, the same user also needs to be created on instance 2. Error handling and logging is essencial for early detection of issues (what if instance 1 updates successfully, but instance 2 fails?).

## The unintended benefits of building your own geo-redundancy

Apart from cost savings, there are some unintended benefits of having separate instances fully under our control, rather than opting for a built-in geo-redundancy in Premium tier. Let's say there are risky configuration changes about to go out into production environment. There's always that possibility in cloud engineering... "What if things go wrong? Do I have a rollback strategy?" Having that backup API Management instance ready on-hand to pick up traffic should things go wrong is a significant safety asset that should not be underestimated.