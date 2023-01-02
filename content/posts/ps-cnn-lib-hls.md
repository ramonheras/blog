---
title: "Project showcase - CNN layer library in HLS"
tags: [AI, CNN, convolutional-neuronal-networks, DNNDK, Vitis, C++, High-Level-Synthesis, HLS Works]
categories: [Product Showcase, HLS Works]
author: "Ramon Heras"
draft: false
Toc: false
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# Project showcase - CNN layer library in HLS
 
The main goal of this project was to create an HLS library for implementing CNN's convolutional layers. Our client used this library to study FPGA performance against other alternatives. 


1. *Pixel per clock per clock*: The generated IP core is able to process one pixel per clock 99% of the time. That's way better than the Xilinx HLS OpenCV library. 

2. *360FPS*: The tested IP core reached approximately 130MHz. In conjunction with a throughput of one pixel per clock, the IP can process more than 360 HD images per second. 

3. *Generic C++*: The library is designed to be generic, so it's possible to create several layers, changing the number of kernels and their content and size. 

## The library  

The library is written in C++. It is similar to Xilinx OpenCV in terms of code organization and naming (functions, objects, and files), but it is based on a different processing schema that reduces intermediate buffer sizes and unactive hardware generation. This library outperformed the Xilinx equivalent. 

Our library gathers typical operations used in convolutional layers. All operations support mixing types. For example, it works when the image's pixels are integers, and the filter pixels are fixed-point. 

Some other features include… 

- Kernel convolution. 
- Several activation functions like parametric ReLU and custom leaky ReLU. 
- Essential operations like crop, scale, insert padding. 

This project proposed a scalable architecture unlike [DNNDK](https://www.xilinx.com/support/documentation/user_guides/ug1327-dnndk-user-guide.pdf) (Deep Neural Network Development Kit), which is a fixed-size neural network accelerator. The resource usage and computing power of this project grows alongside the neural network; in contrast, the DNNDK is based on several fixed-size accelerators that do not increase in resources and performance. 

Implementing CNN's convolutional layers by using the library "hls_video" (now known as [Vitis Vision Library](https://github.com/Xilinx/Vitis_Libraries/tree/master/vision)) is tricky since the library only supports up to 3 channels, which is not enough. The number of channels increases when going deeper into the CNN. A solution could be process data as monochrome images in parallel. This solution would increase the resources used with the number of channels. But on CNNs, the data-to-process doesn't grow very fast since several techniques are used to avoid it (e.g., by adding pooling layers, etc.). In summary, although resource usage grows fast, the amount of data to be processed does not increase that fast, so most hardware remains inactive. Other comparable solutions rely on huge buffers that choke memory interfaces or use too much internal RAM between Block, Ultra, and distributed. 