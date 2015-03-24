---
layout: page
title: "Tutorials"
date: 2013-09-09T14:31:00.362000+01:00 
modified: 2015-03-22T23:06:00.362000+01:00
excerpt:
image:
  feature: sample-image-6.jpg
  credit: WeGraphics
  creditlink: http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/
---

With [Bran’s](http://www.osdever.net/tutorials/view/brans-kernel-development-tutorial) and [James’](http://www.jamesmolloy.co.uk/tutorial_html/index.html) kernel development tutorials there already exists a variety of  great guides on how to build a UNIX-clone operating system for the x86 architecture.
However, in the next few weeks I will publish a set of tutorials, which tries to clear open questions and to give another perspective on the topic.

The tutorials are practical guidance.
It is important to note that the implemented kernel is a teaching kernel.
The used algorithms are not optimal and mostly space inefficient.
They are normally chosen for their simplicity and ease of understanding.

#### Prerequisites

The following tools are required to build the sample codes:

* nasm
* gcc and binutils, which are able to create and to deal with the [Executable and Linking Format](http://refspecs.linuxbase.org/elf/elf.pdf) (ELF)
* gdb
* git
* [qemu](http://www.qemu.org/)
* a text editor (e.g. vi)

All examples are published via GitHub at [https://github.com/RWTH-OS/eduOS](https://github.com/RWTH-OS/eduOS).

#### Table of Contents

1. [The *smallest* HelloWorld of the World (or not)](/tutorials/smallest-helloworld-of-the-world-or-not.html)
2. Kernel debugging with qemu
3. to be continued
