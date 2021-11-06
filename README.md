---
permalink: /
---

Today in Kubernetes, the exposure of secrets for connecting application workloads to external services such as REST APIs, databases, event buses, and many more is manual and bespoke.  Each service provider suggests a different way to access their secrets, and each application developer consumes those secrets in a custom way to their workloads.  While there is a good deal of value to this flexibility level, large development teams lose overall velocity dealing with each unique solution.  To combat this, we already see teams adopting internal patterns for how to achieve this workload-to-service linkage.

This project specifies a Kubernetes-wide specification for communicating service secrets to workloads in an automated way.  It aims to create a widely applicable mechanism but _without_ excluding other strategies for systems that it does not fit easily.  The benefit of Kubernetes-wide specification is that all of the actors in an ecosystem can work towards a clearly defined abstraction at the edge of their expertise and depend on other parties to complete the chain.

## User Guides
To get started, please check out the guide for the appropriate role

* [Application Developer](/application-developer/)
  Expects secrets to be exposed consistently and predictably
* [Service Provider](/service-provider/)
  Expects secrets to be collected consistently and predictably
* [Application Operator](/application-operator/)
  Expects secrets to be transferred from services to workloads consistently and predictably

## Specification
* Core
  * [1.0.0-rc3](/spec/core/1.0.0-rc3/) (pre-release)
  * [1.0.0-rc2](/spec/core/1.0.0-rc2/) (pre-release)
  * [1.0.0-rc1](/spec/core/1.0.0-rc1/) (pre-release)
