---
title: "Learning pills - Array as HLS top function parameter"
date: "2021-02-01" #ToDo
tags: [Xilinx-HLS, Vitis, C++, High-Level-Synthesis, HLS Works]
categories: [Xilinx High-Level Synthesis, HLS Works]
author: "Ramon Heras"
draft: false
Toc: false
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# Learning pills - Array as HLS top function parameter 

Type: Blog post 
Length: 524 words 

This post's idea is to explain how to set an array as the top function parameter in a simple way.  You may think it is as simple as writing `foo(Type args[SIZE])`, and you’d be right! Not so fast; what would be the point of this post then? In HLS, we are designing hardware, and the top function wraps that hardware. Therefore, the top function parameters are interfaces. So the point is that an array can result in different interfaces depending on what we tell to the HLS tool. 

HLS will consider the array "args" as a memory interface (`ap_memory`) by default; this interface includes the following signals: `data` (one on each direction `out`/`in`, for reading and writing), `enable`, `address`, and `write enable`. HLS may optimize and remove one of the data signals and the write enable. For example, if we only read the array, HLS won't generate the `data in` (data write) because this port wouldn't be used.  

A memory interface is a typical interface for an array, but sometimes the array represents a single piece of data that we want to process at once. How can we specify it to the HLS tool?. The answer is not as straightforward as it may appear; the first idea people usually came up with is changing how we access the array instead of changing the interface. For example, inserting the unroll pragma. But it won't operate the entire array at once, and even worse, the pragma will mess up the interfaces. For the code below, HLS generated two memory interfaces per parameter—really? 

 
```cpp
const uint32_t CTE = 12; 

void core(int32_t in[N], int32_t out[N]){ 
  for(int i=0; i<N; i++){ 
    #pragma HLS unroll 
    out[i] = in[i] * CTE; 
  } 
} 
```
 

Another idea is using the interface pragma on the array, which lets us tell the compiler that the array is actually a data stream (axis). Therefore the tool will generate an AXI-Stream interface. We can also specify other AXI interfaces such as AXI-Lite or AXI4. But for this example, AXI4 is overkill and AXI-Lite won't be adequate either. AXI-Lite consumes more resources and is meant to access core registers from a processor. By using this pragma, we are specifying that "in" and "out" are registers. Using AXI-Lite is a pretty slow access pattern. In this case, AXI-Stream is the best, but the core still can't perform all the operations at once. 

```cpp
#pragma HLS INTERFACE axis       port=in   // AXI Stream 
#pragma HLS INTERFACE m_axi      port=in   // AXI4 memory maped 
#pragma HLS INTERFACE s_axilite  port=in bundle=control  // AXI Lite  
```

What about specifying other modes on the interface pragma? } 


`ap_none`, `ap_vld`, `ap_ack`, `ap_hs`: None of them work and the resulting interface is ap_memory 

`ap_fifo`: This will generate a standard FIFO interface with data, full, and empty signals. Each time we access an array element, we perform a read operation on that interface. 

`bram`: This one will result in a standard BRAM interface, which is similar to ap_memory but includes CLK and reset signals.  

 

In both axis and `ap_fifo` interfaces, the access pattern must be sequential. In contrast, ap_memory and bram allow random access patterns. 

 

Finally, what's the way to execute all the operations simultaneously, then? It is actually pretty simple:, just pack the array into a structure and use the interface pragma on that parameter, as follows. 

 
```cpp
#include <ap_utils.h> 
#include <cstdint> 
#include "core.hpp" 
 
const uint32_t CTE = 12; 

template<typename T, unsigned SIZ> 
struct Arg_t { 
    T arr[SIZ]; 
}; 

 
using MyArg_t = struct Arg_t<int32_t, N>; 


void core(MyArg_t in, MyArg_t out){ 
  for(int i=0; i<N; i++){ 
    #pragma HLS unroll 
    out.arr[i] = in.arr[i] * CTE; 
  } 
} 
```

In the pictures below, you can see the different interface summaries to compare the presented interfaces. 

{{< figure src="/images/lp-array-top-param/1_AxiLite.jpg" alt="Test pipeline" position="center">}}
*Figure 1* AXI Lite  

{{< figure src="/images/lp-array-top-param/2_Axi4Mm.jpg" alt="Test pipeline" position="center">}}
*Figure 2* AXI4 MM  

{{< figure src="/images/lp-array-top-param/3_AxiStream.jpg" alt="Test pipeline" position="center">}}
*Figure 3* AXI Stream  

{{< figure src="/images/lp-array-top-param/4_Bram.jpg" alt="Test pipeline" position="center">}}
*Figure 4* BRAM

{{< figure src="/images/lp-array-top-param/5_Fifo.jpg" alt="Test pipeline" position="center">}}
*Figure 5* FIFO  