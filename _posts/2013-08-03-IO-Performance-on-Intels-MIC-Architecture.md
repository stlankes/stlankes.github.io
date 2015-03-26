---
layout: post
title: "IO Performance on Intel’s MIC Architecture"
date: 2013-08-03T14:31:00.362000+01:00 
tags: [MIC]
thumb: thumbnail-netio_phi.png
share: true
comments: true
---

Over the last few months I have been working with Intel’s MIC architecture.
It is well described in many articles, basic information is available at [http://software.intel.com/en-us/mic-developer](http://software.intel.com/en-us/mic-developer).

I like the architecture.
However, it is hard to write code which utilizes all cores and its large vector units.
If your code does not fulfill these requirements, it is not suited for the current MIC architecture.

In this post, I would like to discuss the MIC architecture’s IO performance -- more precisely, I used the Intel Xeon Phi Coprocessor 5110P for my tests.
One of my test programs used the file system to temporarily store results.
The IO performance was extremely bad. This problem is very simple to recreate, for instance, with the following example:

	dd if=/dev/zero of=/dev/shm/foo count=16 bs=1M
	16+0 records in
	16+0 records out
	16777216 bytes (16.0MB) copied, 0.081972 seconds, 195.2MB/s

A bandwidth of 195 MB/s to write data into tmpfs is a poor result. For comparison only, the host (Intel Xeon E5-2650) reaches a bandwidth of 1.5 GB/s.
Admittedly, a [busybox system](http://www.busybox.net/) is used on the MIC architecture, which is not optimized for high IO performance.
Therefore, I also used [Bonnie](https://code.google.com/p/bonnie-64/) -- an IO benchmark -- to evaluate the performance and got similar results for the peak performance (~500 MB/s on the coprocessor vs. ~7400 MB/s on the host).

Furthermore, I evaluated the TCP/IP bandwidth via the [netio](http://www.ars.de/ars/ars.nsf/docs/netio) benchmark.
I started the server on the coprocessor while the client is running on the host.
The results (see figure) are also extremely bad (~200 MB/s), in particular if we consider that the TCP/IP connection is tunneled over PCIe x16, which provides a theoretical peak performance of

> number of lanes * number of transactions per lane * amount of data per transaction * encoding overhead = 16 lanes * 5 GT/s per lane * 1 Bit/T * 8Bit/10Bit = 64 * 10^9 Bit/s = 8 * 10^9 Byte/s = ~7629 MByte/s.

(PCIe V2.0 use an  8b/10b encoding, which has an overhead of 20%.
By the way, why does the Intel Xeon Phi 5110P not support PCIe V3.0, which promises a higher bandwidth?)
The figure shows also that we reach *nearly* the theoretical peak performance, if we use Intel’s [Symmetric Communication Interface](http://www.intel.com/content/dam/www/public/us/en/documents/product-briefs/xeon-phi-software-developers-guide.pdf) (SCIF).
SCIF is the low-level communication backbone between the host and the MIC in a heterogeneous computing environment.

![SCIF bandwidth](/images/netio_phi.png)

Admittedly, the single core performance of the current MIC architecture is poor, which could explain the bad IO performance.
However, does this explain everything?
I am not sure and have started a deeper analysis...
