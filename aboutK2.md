---
title: About K2
keywords:
last_updated: July 3, 2016
toc: false
permalink: aboutK2.html
---
[K2](https://github.com/samsung-cnct/k2) will create a production grade Kubernetes cluster on a range of platforms using its default settings. When you are ready to optimize your cluster for your own environment and use case, K2 provides a rich set of configurable options.  

Samsung CNCT developed the first iteration of K2 out in the open via the [Kraken](https://github.com/samsung-cnct/kraken) project. We took the lessons learned from a year's worth of experience and carried forward best practices to K2. We continue to use Ansible and Terraform as the basis of Kubernetes cluster management because we believe these tools provide flexible and powerful abstractions at the right layers.

For those of you who are familiar with Kraken, K2 provides the same functionality with much cleaner internal abstractions.

## What is K2 for
K2 is targeted at operations teams that need to support Kubernetes, a practice becoming known as "Cluster Operations". K2 provides a single interface where you can manage your Kubernetes clusters across all environments.

K2 uses a single file to drive cluster configuration, providing two major benefits:

- It simplifies keeping your cluster configuration under version control as you promote changes from dev through to production. This applies to existing cluster configurations and brand new ones.

- It enables Continuous Integration for applications running on sandboxed and transient Kubernetes clusters.

We believe this simplified configuration management facility is essential for effectively and efficiently operating a Kubernetes based infrastructure.

## K2 Support Projects
K2 is supported by a variety of other tools, including:

- [k2cli](k2cli) - a command-line interface wrapper
- [k2-charts](k2-helm-charts) - Helm charts for a variety of useful services
- [logging tools](k2-logging) - tools for centralized logging

These tools are available on GitHub.
