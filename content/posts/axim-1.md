---
title: "The AXI Master interface in HLS (The Basics)"
date: "2021-05-01"
tags: [Xilinx-HLS, AXI-Master, C++, Basics]
categories: [Xilinx High-Level Synthesis]
author: "Ramon Heras"
draft: false
mathjax: true
---

> 📝 *This post was initially released on the HLS Works Blog in 2020. The post was moved to this website after HLS Works closed in Sep 2021.*

# The AXI Master interface in HLS (The Basics)

The AXI4 master is a powerfull interface that supports many features, but probably the most remarkable feature is support for burst transactions (Covered in detail in this post). An AXI Master (AXIM) interface is commonly used to access the DDR memory, though it can also be used to access other cores, such as BRAM or URAM. On the other side, an AXIM is complex and requires a lot of FPGA real estate. The operating frequency is also significantly reduced in comparison to different interfaces, such as the AXI stream.

This post addresses the basics of designing an AXIM-powered IP core. It is written with the assumption that the reader has some knowledge regarding this interface. If you do not, I recommend watching [these videos](https://www.youtube.com/watch?v=1zw1HBsjDH8&list=PLaSdxhHqai2_7WZIhCszu5PLSbZURmibN) or reviewing the [AMBA AXI specification documentation](https://developer.arm.com/documentation/ihi0022/e/AMBA-AXI3-and-AXI4-Protocol-Specification). Throughout this post, I will be referring to the HLS documentation quite often, in particular to [UG1399-v2020.2](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf). As I explain later on, some concepts in the documentation are not clear enough, in my opinion.

Let's start with some code. The code below is the simplest example using AXIM that I can think of. It takes an input array and adds the value of val to each element. Both val and the starting address from which the AXI master interface will read the array are set through an AXI-Lite slave interface.

```cpp
void SimpleCore( int inputArray[ARRAY_SIZE], int outputArray[ARRAY_SIZE], ap_int<8> val){
	#pragma HLS interface s_axilite port=return bundle=CONTROL
	#pragma HLS interface s_axilite port=val bundle=CONTROL
	#pragma HLS interface m_axi port=inputArray offset=slave bundle=GMEM0
	#pragma HLS interface m_axi port=outputArray offset=slave bundle=GMEM0

	for(int i=0; i<ARRAY_SIZE; ++i)
		outputArray[i] = inputArray[i]+val;
 };
```
## Offset=slave?

The AXIM is a master interface, so it is responsible for reading the data. The starting address from which to start reading is known as the slave address or offset. With the offset parameter, we specify how to tell our IP core the address. I have set it to slave, which causes the HLS tool to create a slave register to set the slave address. This register can either be read or written by your system's processor through an AXI-Lite slave interface that is also added to your IP core. 

There are other modes for this setting, such as "fixed" or "off," but these modes are rarely used. The "direct" mode tells the HLS tool to add an extra input to the core instead of using an AXI Lite register. Finally, the "off" option does not create any offset port nor an AXI-Lite register. In this last case, the starting address is configured in the IP Integrator window ("Base address of target slave," [page 241](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_2/ug1399-vitis-hls.pdf#page=241)) when instantiating the core in the block design in Vivado. Note that this last approach does not allow you to change the address at runtime.

## `int*` or `int[SIZE]`?

In C/C++, there is no difference between both options. However, I prefer the second option because it tells HLS the burst depth so that I do not have to specify it manually with the pragma option depth, this way you keep the pragma line cleaner. The tool might indicate that the depth is still 0 but the simulation is not going to fail, as it would do if you were not specified the size anywhere.

The depth parameter indicates the maximum burst length and is used in co-simulation. HLS uses the depth parameter to calculate the size of the FIFO that interfaces the C++ test bench with the RTL core under test. A FIFO is necessary for each streaming interface, including AXIM and AXI-Stream. If the depth parameter is not specified, the cosim is likely to fail since HLS usually cannot deduce the burst length from the code.

```cpp
 #pragma HLS interface m_axi port=outputArray depth=100 offset=slave bundle=GMEM0
```

## Top Parameter 

The top function parameter will most likely be an array or pointer (which, in practice, are the same), but of what type? As with other interfaces, you can either use (1) a C/C++ type (int, char, etc.…), (2) an arbitrary type (ap_int, ap_uint. Check out the post "Why use arbitrary types") or (3) a custom struct. Simplicity is best, especially with an AXI Master interface, so I tend to use an int32 because it is of the same size as the interface, 32 bits. Sometimes a struct might be more suitable, but it can be tricky due to alignment concerns—learn more about alignment in the Enabling Burst Transactions post.

## Last Notes When Using AXIM Powered IP Cores

One important precaution must be taken when setting the slave address. Make sure that this address is aligned with the interface width (normally 4 bytes). To check this, you must make sure that the starting address value modulo the interface byte width is 0 . Or, since the interface width is limited to a power of two by the standard, the first least significant bits must be 0.

$$
\\displaylines{StartingAddr \\% AximTdataByteWidth== 0 == StartingAddr[(AximDataByteWidth-1):0 ]\\\\
[a:b]→range\\ of\\ bits\\ from\\ \\mathbf{a}\\ to\\ \\mathbf{b}\\ boundaries\\ included.\\ Ej\ [2:0]→[b_2,b_1,b_0]}
$$

When dealing with burst transactions, there is another critical aspect to remember regarding burst lengths. Learn more about this issue by reading the article **Enabling Burst Transactions**.