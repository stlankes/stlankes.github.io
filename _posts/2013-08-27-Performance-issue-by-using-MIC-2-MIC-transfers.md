---
layout: post
title: "Performance issue by using MIC-2-MIC transfers"
date: 2013-08-27T14:31:00.362000+01:00 
tags: [MIC]
thumb: thumbnail-netio_mic2mic.png
share: true
comments: true
---

Intel’s [Symmetric Communication Interface (SCIF)](http://www.intel.com/content/dam/www/public/us/en/documents/product-briefs/xeon-phi-software-developers-guide.pdf) is the low-level communication backbone between the host and the MIC.
It looks like a mixture of BSD sockets and DAPL.
Like DAPL, SCIF is available in user and kernel space (with minimal changes) and offers the possibility of data transfers via remote direct memory access (RDMA) or a direct remote memory access (RMA).

I wrote a benchmark to evaluate the bandwidth between host and MIC.
In the post "IO Performance on Intel’s MIC Architecture", I already showed that SCIF is able to reach the expectable bandwidth via RDMA.
The benchmark could also be used to evaluate the bandwidth between to two MICs, which are plugged into the same host.
In this case (see figure below), the bandwidth is suboptimal. It seems that the integrated PCIe fabric of the host processor (Intel Xeon E5-2650) has a problem forwarding the messages with the expectable bandwidth.

<figure>
<img src="/images/netio_mic2mic.png">
</figure>

However, this approach use RDMA transfers, which doesn’t stress the host processor.
If a higher bandwidth is required, a proxy could be started on the host, which actively manages the data transfers between the MICs.
A similar approach is used by [MVAPICH2](http://mvapich.cse.ohio-state.edu/publications/ofa_apr13_mvapich2_mic.pdf), which solves the problem in the MPI layer.
However, such an approach wastes processor cycles on the host side.
