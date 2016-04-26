---
layout: post
title: "HermitCore - A Unikernel for Extreme Scale Computing"
date: 2016-04-26
thumb: devnull.jpg
tags: [HermitCore, link post]
link: http://www.hermitcore.org/
share: true
comments: true
---

I just open the GitHub repository of our project [HermitCore](http://www.hermitcore.org/), which is a new unikernel targeting high-performance computing.
HermitCore extends on the multi-kernel approach with unikernel features to provide better programmability and scalability for hierarchical systems.
By starting HermitCore applications, cores will be split off from the Linux system and the applications run bare-metal on these cores.
This approach achieves a lower OS jitter and a better scalability. HermitCore applications and the Linux system can communicate via an IP interface (e.g. inter-kernel communication).

HermitCore is the result of a research project at RWTH Aachen University and is currently an experimental approach, i.e. not production ready. Please use it carefully.
A first paper with preliminary results will be published at the [International Workshop on Runtime and Operating Systems for Supercomputers (ROSS 2016)](http://www.mcs.anl.gov/events/workshops/ross/2016/).
