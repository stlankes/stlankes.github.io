---
layout: post
title: "xhyve - Lightweight Virtualization on OS X"
date: 2015-07-05
thumb: devnull.jpg
tags: [OS X, Hypervisor]
share: true
comments: true
---

Maybe you can recognize from my former posts, that - beside Linux - OS X is my favorite operating system.
On OS X, I use also [Qemu](http://www.qemu.org) to test my kernel extensions.
To improve the performance, Qemu is able to use [KVM](http://www.linux-kvm.org), which is Linux's kernel framework to support Intel's and AMD's hardware virtualization extensions (e. g. nested page tables).

Since OS X 10.10 (Yosemite), a hypervisor framework is integrated, which could be used to support similar features.
[xhyve](http://www.pagetable.com/?p=831) is a lightweight virtualization solution, which supports this framework and is at least capable to run Linux and FreeBSD on OS X.
In principle, it is a port of FreeBSD’s [bhyve](http://bhyve.org/).

*xhyve* works very well.
I tested it with the Linux kernel, which I built in my [kernel guide](https://techblog.lankes.org/2015/05/01/My-Memo-to-build-a-custom-Linux-Kernel-for-Qemu/).
I created and started the virtual machine with the following command:

{% highlight bash %}
	xhyve -m 1G -c 2 -s 0:0,hostbridge -s 31,lpc -l com1,stdio -f kexec,./bzImage,./myinitrd.cpio,"earlyprintk=serial root=/dev/ram0 rootfstype=ramfs init=init console=ttyS0"
{% endhighlight %}

The flag `-m 1G` specifies the memory size of the VM to 1 GByte, while `-c 2` defines the number of cores to 2.
The Linux kernel is configured to use the serial port (`ttyS0`) as standard communication interface.
To establish a valid serial port and to redirect the output of the virtual machines to the console of my host system, the flags `-s 0:0,hostbridge -s 31,lpc -l com1,stdio` are used.
The final flag `-f` specifies the location of the kernel, the location of the intial ramdisk and the kernel parameter.

That’s it! *xhyve* is simple and it works!
