---
title: "Enabling Burst Transactions on an AXIM Interface in HLS"
date: "2021-06-01"
tags: [Xilinx-HLS, AXI-Master, C++, Burst-Transactions]
categories: [Xilinx High-Level Synthesis]
author: "Ramon Heras"
draft: false
mathjax: true
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# Enabling Burst Transactions on an AXIM Interface in HLS

Burst transactions are essential to take full advantage of the AXIM interface. After performing many tests and reading the documentation several times, I must say that there is not a well-defined behavior of Vitis HLS. In this post, I summarize my thoughts and experimentation results on optimizing burst transactions. You can find an introductory blog to this topic here.

Unfortunately, the only way to ensure bursting is by diving into the wave viewer to check for it yourself. In Vitis 2020.2, there is a section specifically to report burst-related issues, the *AXI_M Burst Information" section located in the "Synthesis Summary." In older versions, you can still find this information by skimming through the synthesis warnings or checking the synthesis log on the console. Unfortunately, this report is not accurate enough and tends to throw many false messages. Still, I follow the tool's guidance, hoping that someday it'll be more accurate or at least consistent with the documentation.

## Access Pattern

The access pattern is the most critical aspect to make the burst optimization succeed. The access pattern must be sequential and incremental. The simplest way to ensure such an access pattern is by using an array as a top function parameter that is accessed from a single loop (to avoid overlapping).

 
```cpp
for(int i=0; i<ARRAY_SIZE; ++i)
    outputArray[i] = inputArray[i]+val;
```


## Top Parameter Alignment

When you tell HLS to use a particular interface (AXIM or AXI Lite), HLS just adds an IP core called an adapter that defines the bus functional model and takes care of all the interfacing logic (See pages 233-236). The alignment of the parameter that you put on the top function concerns the interfacing between the HLS core and the adapter. Types aligned to a smaller value than the interface width lead to a bottleneck. This is because the AXIM adapter is capable of reading up to 4 bytes per clock cycle (for a 32-bit wide interface), but the HLS core only reads from the adapter the number of bytes to which the top parameter is aligned, per clock cycle. For instance, if the top parameter is a char, access is limited to only a single byte per clock. 

## Quick Note on Structs 

As previously mentioned, when using structs, be careful with alignment since it limits the access in HLS. In the example below, the struct alignment is 1 byte, which limits access to a single byte of data per clock cycle. You might be tempted to use pragmas to solve this issue, perhaps partitioning the inner array  or aggregating the struct, but none of those solutions will work. Partitioning the inner array will not work because the only place to put this pragma is in the struct constructor, and since it is an interface, the pragmas inside of it are ignored. And aggregating the struct will not work neither because using the aggregate pragma on an AXI master interface is not allowed . In contrast, if you were after an AXI stream interface, the aggregation would be automatic. The correct solution is much simpler. Simply align the struct properly using the C++ keyword alignas to read the struct entirely in a single clock cycle.

```cpp
//Default alignment 1 byte
struct Sample_t{
    char data[4];
};

//Proper alignment 4 bytes
struct alignas(4) Sample_t{
    char data[4];
};
```


## Memcpy?

```cpp
void SimpleCore2b(int inputArray[ARRAY_SIZE], int outputArray[ARRAY_SIZE], ap_int<8> val){
#pragma HLS interface s_axilite port=return bundle=CONTROL
#pragma HLS interface s_axilite port=val bundle=CONTROL
#pragma HLS interface m_axi port=inputArray offset=slave bundle=GMEM0
#pragma HLS interface m_axi port=outputArray offset=slave bundle=GMEM0


int localBuffer[ARRAY_SIZE];
memcpy(localBuffer, (const int*)inputArray, sizeof(int)*ARRAY_SIZE);


for(int i=0; i<ARRAY_SIZE; ++i)
    outputArray[i] = localBuffer[i]+val;

}
```

In many examples, a local buffer is used along with memcpy to copy the data from the parameter (the interface) to the local buffer; see the code above. This approach is necessary when the burst optimization is not being applied. Sometimes, HLS cannot guess burst length due to a bad access pattern or other reasons. Using a local buffer effectively forces burst transactions, though this strategy has two main drawbacks. First, additional memory resources are needed to allocate the local buffer [either SLICEMs (to create distributed RAM), BRAMs, or URAMs]. Second, the burst length is of a fixed size, filling the entire buffer regardless of whether the entire buffer is needed.

## Volatile?

```cpp
    void fooA(volatile int inputArray[ARRAY_SIZE])
    // Or
    void fooB(int inputArray[ARRAY_SIZE])
```

Xilinx documentation is not very clear about whether or not to use the volatile qualifier on the top parameter. On pages 159 and 160, they state that volatile, following the standard C/C++ behavior, instructs the HLS tool not to optimize the variable to which it is applied. This may include no burst access, port widening, or dead code elimination. 

But then, on pages [169-171](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf#page=169) and [228-238](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf#page=228), Xilinx provides many examples using the volatile and explains that volatile is used on ports (top function parameters) that are read from or written to several times during the code execution, especially on pointers. This same section refers to "Optimizing Burst Transfers" ([page 315](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf#page=315)), where there is yet another example without the volatile keyword. Furthermore, using volatile throws the burst failure 5, [error [214-227]](https://www.xilinx.com/html_docs/xilinx2020_2/hls-guidance/214-227.html). One more caveat is that you cannot use an arbitrary precision (AP) type (such as ap_int or ap_fixed) along with the volatile qualifier since AP types do not support the volatile qualifier (and the code would not compile), see [page 160](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf#page=160). 

Well, that is the theory; in practice, using volatile does not prevent bursting every time. Generally, the impact on resource utilization is limited, but it can have a considerable effect on everything from resources to performance depending on the case. The only way to make sure that everything is working properly is by checking the wave window. I personally tend to follow the tool guidance and generally would not use the volatile qualifier in this scenario. Another caution that I take is not reading/writing the same element several times but instead, copying it to/from another variable before. That is not strictly necessary, but it is good practice, especially when writing/reading to variables instead of arrays. The code snippets below reflect what I've seen during code reviews.

```cpp
// Avoid writing to an output twice 
out[i] = 5;
if(something())
    out[i] = var; //second write


if(something()) out[i] = var; //Writing once
else            out[i] = 5; 


// Avoid reading from an input twice 
if(something(in[i])) var = in[i]; //Second read
else                 var = 0;

auto aux = in[i]; //Reading once
if(something(aux))  var = aux; 
else                var = 0;
```

## Just Checked the Wave Viewer, and the Burst Length Is Not Fully Optimal...

After making sure that the access pattern is perfect or even using memcpy to force it, you might find that the burst length is shorter than the optimal length. For instance, a for loop might be split into several bursts. This scenario is quite common because  Vitis HLS limits the maximum burst length by default to just 16 beats. To change this, increment the burst read/write size to the desired value (256 is the maximum for a 32-bit wide interface). You can do this in either of the following ways.

1. Add the options max_read_burst_length=256 or max_write_burst_length=256 to the interface pragma

2. Modify the default settings at `solution settings` > `general / configuration settings` > `config interface`

When setting up this value, there are two limitations:

1. The AW_LEN signal, which specifies the burst length (number of beats per transaction), is only 8 bits wide; therefore, the maximum value is 255. In AXI, the minimum length is one beat, so the transaction length is the AW_LENGTH value plus one. This way, a value of AW_LENGTH of 0 signifies a single beat transaction (the minimum). Therefore, the maximum burst size is really 256.

2. The limit for a burst is 4 KB per transaction; this is an AXI standard limitation. Therefore the maximum burst size depends on the channel TDATA width as well.

The following formula agglutinates the two limitations described above:

$$
maxReadBurstLength = min(4096/axiChannelByteWidth,\\ 256)
$$

## Optimizing A Bit More

By adding the latency option to the interface pragma, you can specify the expected latency. That might be a little confusing at first glance since there are two latencies involved with an AXIM burst. 

* Read latency: the number of clock cycles between requesting a read and receiving the first datum

* Write latency: the amount of time between the first write request and receiving the write acknowledgment

Specifying this latency option allows your IP to request access to the channel before it is needed. The best way to optimize your system is by monitoring all the transactions and synchronizing the different modules to share the memory bandwidth efficiently, rather than playing with the latency option alone. Note that if you choose a value too high, the IP core might have access to the channel before actually needing it, therefore not using it and wasting bandwidth. 

Similarly, if the latency value is too low, your core might stall waiting for access. As a good starting point, I would leave the default value as is. On [page 311](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf#page=311), there is a more detailed explanation of the implied latencies.

 
```cpp
    #pragma HLS interface m_axi port=outputArray offset=slave bundle=GMEM0 latency=<value>
```

## Last Notes When Using AXIM Powered IP Cores

In addition to the practices discussed in the previous section, another precaution must be taken when inferring burst transactions. A read or write burst must not cross the 4 KB boundaries; this rule is an AXI standard limitation. To verify this, the following statement must be true:

$$
\\displaylines{(StartingAddr\\ \\%\\ 4096)+BurstLength*BurstSize<4K=4096\\\\
BurstLength→Number\ of\\ transfers/beats\\\\\
BurstSize→Number\\ of\\ bytes\\ per\\ transfer/beat} 
$$

For the first example (100 beats of 4 bytes) and given an starting address of 0x7871D0, the statement is as follows:

$$
\\displaylines{(7893456\\ \\%\\ 4096)+100*4<4K=4096\\rightarrow 464+400<4K\\rightarrow864<4096 \\rightarrow OK!}
$$

The easiest way to avoid manually checking the above all the time is to use buffers already aligned to 4 KB; this method gives you the benefit of pushing the burst size to the 4 KB limit. See the examples below:


```cpp
// Instead of This
int buff[N];

// This (C++11)
alignas(4096) int p[N];

// This (C11 User space)
#include <stdlib.h>

int* buff = (int*)aligned_alloc(4096, sizeof(int)*N);

// This (C Kernel space)
#include <linux/slab.h>

struct kmem_cache_t
    *buff_cache = (kmem_cache_t*)kmem_cache_create(
        "buff_4k_cache",
        sizeof(int)*N,
        1<<12, /* 4K alignment */
        SLAB_HWCACHE_ALIGN); // Add | SLAB_POISON | SLAB_RED_ZONE for debugging
``` 


