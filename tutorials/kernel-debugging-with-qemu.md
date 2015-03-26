---
layout: page
title: "Part II: Kernel debugging with qemu"
date: 2013-09-09T14:31:00.362000+01:00
modified: 2015-03-22T23:06:00.362000+01:00
image:
  feature: ritchie.jpg
  credit: Peter Hamer [CC BY-SA 2.0]
  creditlink: https://en.wikipedia.org/wiki/Unix
---

Before I continued my kernel tutorial, I would like to explain the easiest form of kernel debugging.
I used [QEMU](http://www.qemu.org/) as test platform, which is an open source machine emulator and virtualizer for a general-purpose computer architectures.
Therefore, it is an ideal platform  to debug basic components of an operating system kernel.

Usally, I would explain kernel debugging based on my minimal kernel, which I presented in the [first part of my kernel tutorials](/tutorials/smallest-helloworld-of-the-world-or-not.html).
However, in this tutorial I used aggressive compiler flags to reach an excellent performance.
For debugging it is better to use less aggressive optimization flags.
Especially the flag `-fomit-frame-pointer` should not use, whereas the base pointer register (`EBP`) is not used to save, set up and restore frame pointers.
This makes an extra register available for compiler optimization.
However is also makes debugging impossible on some machines.
Therefore – for debugging purpose – I change the compiler flags in my Makefile from

	# Compiler options for the final code
	CFLAGS = -g -m32 -march=i586 -Wall -O2 -fno-builtin -fstrength-reduce -fomit-frame-pointer -finline-functions -nostdinc $(INCLUDE) -fno-stack-protector

to

	# Compiler options for kernel debugging
	CFLAGS = -g -O -m32 -march=i586 -Wall -fno-builtin -DWITH_FRAME_POINTER -nostdinc $(INCLUDE) -fno-stack-protector

In later tutorials, I have to manipulate the stack and its entries.
For this reason, I avoid compiler specific stack manipulations (e.g. `-fno-stack-protector`), which I am not able to forecast, and define the macro `WITH_FRAME_POINTER` to signalize that the debug version uses the EBP as frame pointer.

qemu offers the possibility to debug the system over a TCP/IP connection.
With the start option -s qemu accept incoming TCP/IP connections from a gdb at port 1234. I use also the option -S, which freeze the CPU at startup.
This provides enough time to start a gdb session and to set up its connection.

Furthermore, I have to separate the debug information from the OS image.
My Makefile does this per default. In principle, the Makefile compiles all files with the option -g, which add debug information to the object code.
After linking the object files to one OS image, I have one file, which contains the executable and its debug information.
Now, I copy the debug information with following command to a new file:

	objcopy –only-keep-debug eduos.elf eduos.sym

Afterwards, I remove the debug information from the OS image:

	objcopy –strip-debug eduos.elf

Finally, I try to debug my kernel. At first, I start qemu with the options -s -S:

	qemu-system-i386 -monitor stdio -s -S -kernel eduos.elf

Afterwards, I start the gdb in the directory, where the file with the debug information (eduos.sym) is located. In the gdb shell, I load the debug information with following command:

	(gdb) symbol-file eduos.sym

In my case, qemu and gdb runs on the same machine.
Therefore, I am able to establish a TCP/IP connection to the local host and set a break point at the symbol main.

	(gdb) target remote localhost:1234
	Remote debugging using localhost:1234
	0x0000fff0 in ?? ()
	(gdb) break main
	Breakpoint 1 at 0x101000: file kernel/main.c, line 55.

Finally, I continue the stopped machine with command continue (acronym c), which I prompt in the gdb shell. 
At the break point, the virtual machine stops again. It is possible to debug the kernel step by step with the commands step (acronym s) and next (acronym n). 
Therefore, kernel debugging is identical to debugging a normal user-level application and based on the same commands.
The following screenshot shows an example for debugging a kernel.

<figure>
<img src="/images/kernel_debugging.jpg">
</figure>

It is also possible to use a GUI as front-end to the gdb. For instance, the DataDisplay Debugger (DDD) is very easy to use.
The DDD shows the gdb shell in the lower part of its windows.
Therefore, you have a direct access to the gdb und could reuse this tutorial.
