---
title: "Learning pills - Inferring a register in Xilinx High-Level Synthesis (HLS)"
date: "2021-02-01" #ToDo
tags: [Xilinx-HLS, Vitis, C++, High-Level-Synthesis, HLS Works]
categories: [Xilinx High-Level Synthesis, HLS Works]
author: "Ramon Heras"
draft: false
Toc: false
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# Learning pills - Inferring a register in Xilinx High-Level Synthesis (HLS) 

A year ago, I was designing a relatively simple core and the synthesis report was surprising. The latency was equal to zero; I was not happy with that, so I thought about forcing a register. The following code is the idea I came up with. 

But first: why do we have to bother with inferring registers in HLS? The point of HLS is to abstract the designer from low-level details such as the clock signal and register-related stuff. Instead, it is recommended to specify pipelines and their characteristics using the unroll, pipeline, and latency pragmas. 

Anyway, the trick is pretty simple. We just have to use the pragma “HLS Protocol”. This pragma is commonly used to force HLS to schedule operations in a certain way. This pragma allows us to specify in which order the operations are executed. It also allows us to insert “clock waits”, which is where the core waits for a clock cycle before executing the next operation. This last feature is what I used for our purpose. 

By inserting a “clock wait”, we are forcing the synthesis engine to use a register since it has to store the data somewhere until the next clock cycle. I packaged the register functionality into a template function "Reg", which accepts any type. Notice that the template arguments are deduced automatically. 

`File: core.cpp`

```cpp
#include <ap_utils.h> 
#include <cstdint> 

template<typename T> 
T Reg(T in){ 
    #pragma HLS protocol 
    T reg; 

    reg = in; 
    ap_wait(); 
    return reg; 
} 
 

void core(int32_t in, int32_t &out){ 
    #pragma HLS pipeline // Important to not increase interval 
    out = Reg(in); 
} 
```
 

`File: core.hpp`


```cpp

#include <cstdint> 


void core(int32_t in, int32_t &out); 

```


`File: test.cpp`

```cpp
#include "core.hpp" 
#include <iostream> 


#define N (10) 
int main(){ 
    for(int i=0; i<N; i++){ 
        int out; 
        core(i, out); 
        std::cout << i << ", " << out << std::endl; 
    } 
}
```

The software simulation does not produce any valuable information. 

```
 0, 0 
 1, 1 
 2, 2 
 3, 3 
 4, 4 
 5, 5 
 6, 6 
 7, 7 
 8, 8 
 9, 9 
```
 

In contrast, the synthesis report and the cosim wave view show that the register is working fine! 


{{< figure src="/images/lp-infer-reg-hls/1_synthReport.jpg" alt="Test pipeline" position="center">}}
**Figure 1** Synthesys Report 

{{< figure src="/images/lp-infer-reg-hls/2_simReport.jpg" alt="Test pipeline" position="center">}}
**Figure 2** Simulation Report 

{{< figure src="/images/lp-infer-reg-hls/3_waveView.jpg" alt="Test pipeline" position="center">}}
**Figure 3** Simulation Wave View (Zoom)

{{< figure src="/images/lp-infer-reg-hls/4_waveViewFull.jpg" alt="Test pipeline" position="center">}}
**Figure 4** Simulation Wave View (Full)

## Original Cover

{{< figure src="/images/lp-infer-reg-hls/cover.jpg" alt="Test pipeline" position="center">}}