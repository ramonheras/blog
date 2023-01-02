---
title: "Automate HLS IP Core Deployment"
date: "2021-03-01"
tags: [Xilinx-HLS, C++, DevOps, Deployment, GitHub-Workflows, HLS Works]
categories: [Xilinx High-Level Synthesis, HLS Works]
author: "Ramon Heras"
draft: false
Toc: false
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# Automate HLS IP Core Deployment

In the last post, we introduced our new CI system and its remarkable features, such as automated testing and code coverage report generation. However, that was just the beginning step in our journey to automate most processes to save time and focus on what matters, providing better-quality code to our customers. In this second stage, we have automated the deployment process. In our case, this boils down to pushing repository changes to our customers' git server.

We understand that each organization uses unique processes that do not necessarily match ours; therefore, the main goal was to isolate our customers' processes from ours. To accomplish this task, we used two separated repos: one in our git server provider, and another hosted by our customer. In our repo, we keep track of all source code and configuration files. During the deployment, all the outputs are generated, from the IP core to the HTML reports and documents. Next, all these outputs are moved to a "clean repository." Finally, the clean repository is pushed to the customer's repository using a provided SSH key.

The customer owns a repository in which there is no need to run any script to read reports and documentation or import the IP core to the Vivado IP catalog. Depending on the project (for most projects), we also include the source code. Most projects involve designing custom libraries; hence, it does not make sense to export the IP core alone. Instead, we include the underlying library used to build the IP core, and, if it is requested, all related code such as external included files and tests.

{{< figure src="/images/dev-ops-2/pipeline.jpg" alt="Deployment Pipeline" position="center">}}
**Figure 1** Deployment Pipeline  
