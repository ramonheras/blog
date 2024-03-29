---
title: "Automate test execution HLS"
date: "2021-02-01"
tags: [Xilinx-HLS, C++, DevOps, Verification, GitHub-Workflows, HLS Works]
categories: [Xilinx High-Level Synthesis, HLS Works]
author: "Ramon Heras"
draft: false
Toc: false
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# Automate test execution HLS

Nowadays, continuous integration (CI) is a must, and including CI in our workflow has been a game-changer. We have automated many processes to save an enormous amount of time that can be spent on improving quality and delivering a better product to our customers.

To fully integrate dev-ops into our processes, we have automated test execution and code coverage report generation. Tests in HLS differ significantly from typical software development. In software development, a typical CI workflow compiles and executes tests in several different environments with distinct operating systems and configurations. In HLS, executing the software is just the beginning of the pipeline C simulation (C SIM). A software CI workflow lacks synthesis, co-simulation, and IP exportation. Checking all of these can be tedious and time-consuming, but with the help of automation and CI tools, this process becomes much easier!

There are many challenges to obtaining this workflow, from building in-house tools to hosting the entire CI system on our own servers. This is due to execution time constraints and the large amount of memory taken by Xilinx tools.
What Have We Automated?

We execute a pipeline of operations on each commit to ensure that our code is not broken at any stage:

1. The server runs the tests as regular software (C Simulation or csim for short). We use this step to find early errors as soon as possible since this is the fastest part of the pipeline.

2. The server re-runs the tests but enabling code coverage. With code coverage, we check that each line of code has been executed. It does not ensure the quality of the tests, but it is an essential check.

3. Finally, the server executes unit and integration tests running C synthesis, co-simulation, and IP exportation for each test.

4. If the test pipeline completes successfully, the deployment pipeline is triggered.

Step 3 is the most time-consuming and complex. It can be split into three sub-processes:

1. C Synthesis is executed to verify synthesizability. Identifying non-synthesizable code as soon as possible is crucial to don't waste time later. Next, synthesizable code is translated to RTL, leaving the remaining code as C/C++. The RTL code can be either VHDL or Verilog.

2. In co-simulation, C/C++ code runs as normal software, whereas the RTL hardware description is simulated. Vitis HLS inserts several interfaces to enable data transfers between the software part and the RTL simulation. This step is the slowest due to the RTL simulation.

3. Ip exportation does not simply package the core and the drivers. It also runs extra checking in Vivado to verify functionality, i.e., place and route, among other tests. Those tests culminate in a final report on resource utilization, throughput, max clock frequency, etc. This report is more accurate than the one generated by C Synthesis.

Detecting any breaks in the pipeline at commit granularity has saved our customers a lot of time and money. Figure 1 illustrates the described flow.
Figure 1 Test pipeline

{{< figure src="/images/dev-ops-1/pipeline.jpg" alt="Test pipeline" position="center">}}
**Figure 1** Test pipeline 

Our automated system executes the pipeline described above (C simulation, synthesis, co-simulation, and IP exportation) in various Vitis HLS versions (2020.1 and 2020.2) and operating systems (Windows 10 and Ubuntu 20.04|18.4), running all tasks in parallel to improve execution times.  

## This Is Only the Beginning.

The HLS Works operations roadmap includes much more automation, specifically in the field of deployment operations. From documentation generation and deployment to code deployment, all these topics [KA1] will be covered in a later post.
