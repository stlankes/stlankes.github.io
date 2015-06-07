---
layout: post
title: "My Memo to build a customized Linux Kernel for Qemu"
date: 2015-05-01
thumb: tux.png
tags: [Linux, Kernel]
share: true
comments: true
---

I frequently create a customized version of the Linux kernel and test it with the open source machine emulator [Qemu](http://www.qemu.org).
To create a small system, I use [BusyBox](http://www.busybox.net), which provides tiny versions of many common UNIX utilities in a single small executable.
This will be the base of my *initial ramdisk*.

In this memo, I later use the shared libraries of the host system (x86_64).
It is the fastest way to build a system from scratch, but it is not the correct way.
To avoid dependencies between the host and the target system, you should use [BuildRoot](http://buildroot.org) to create the *initial ramdisk*.
For my tests, I am pretty sure that there are no dependencies to my host system.
The memo is derived from [Humpherys' post](http://mgalgs.github.io/2012/03/23/how-to-build-a-custom-linux-kernel-for-qemu.html) and tailored for my requirements.

### Building BusyBox

Download and unpack the BusyBox source file:

{% highlight bash %}
wget http://busybox.net/downloads/busybox-1.23.2.tar.bz2
tar xf busybox-1.23.2.tar.bz2
cd busybox-1.23.2/
{% endhighlight %}
	
Configure BusyBox with `make menuconfig`.
I used the default configuration and changed only the following options:

	CONFIG_DESKTOP=n
	CONFIG_EXTRA_CFLAGS=-m64
	CONFIG_EXTRA_LDFLAGS=-m64

I used these options to be sure that a 64bit version (x86_64) of BusyBox will be built.
Furthermore, *desktop features* are not required to test the Linux extensions.

Building and installing BusyBox:

{% highlight bash %}
make
make install
{% endhighlight %}

Afterwards, the binaries are installed in the directory `_install`, which is created in the root directory of the source tree.

### Building the initial ramdisk

To build the *initial ramdisk*, the directory structure has to be created as follows:

{% highlight bash %}
mkdir initramfs
cd initramfs
mkdir -pv bin lib dev etc mnt proc root sbin sys tmp
	
cd dev
sudo mknod ram0 b 1 0  # all one needs is ram0
sudo mknod ram1 b 1 1
ln -s ram0 ramdisk
sudo mknod initrd b 1 250
sudo mknod mem c 1 1
sudo mknod kmem c 1 2
sudo mknod null c 1 3
sudo mknod port c 1 4
sudo mknod zero c 1 5
sudo mknod core c 1 6
sudo mknod full c 1 7
sudo mknod random c 1 8
sudo mknod urandom c 1 9
sudo mknod aio c 1 10
sudo mknod kmsg c 1 11
sudo mknod sda b 8 0
sudo mknod tty0 c 4 0
sudo mknod ttyS0 c 4 64
sudo mknod ttyS1 c 4 65
sudo mknod tty c 5 0
sudo mknod console c 5 1
sudo mknod ptmx c 5 2
sudo mknod ttyprintk c 5 3
	
ln -s ../proc/self/fd fd
ln -s ../proc/self/fd/0 stdin # process i/o
ln -s ../proc/self/fd/1 stdout
ln -s ../proc/self/fd/2 stderr
ln -s ../proc/kcore     kcore
cd ..
{% endhighlight %}

Copy the BusyBox system to the directory 'initramfs':

{% highlight bash %}
cp -avR ../_install/* .
{% endhighlight %}
	
The shared libraries are missing in our *initial ramdisk*.
With the command `ldd` we are able to determine the missing files:

{% highlight bash %}
$ ldd bin/busybox 
	linux-vdso.so.1 =>  (0x00007fffe80e4000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f38d6d74000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f38d69b3000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f38d708f000)
{% endhighlight %}
		
As I already mentioned, I use the shared libraries from my host system.
Consequently, I copy these files from my host file system to the directory *initramfs*

{% highlight bash %}
mkdir -pv usr/lib
ln -s lib lib64 # make lib and lib64 identical
cp -av /lib64/lib[mc].so.6 lib/
cp -av /lib64/lib[mc]-2.17.so lib/
cp -av /lib64/ld-2.17.so lib/
cp -av /lib64/ld-linux-x86-64.so.2 lib/
{% endhighlight %}
	
By running `sudo chroot . /bin/sh` in your directory *initramfs*, you can check if your shared libraries are imported correctly.

Now, we have to create `/init`, which will be started after the last stage of the boot process.
It is a shell script, which mounts some pseudo file systems, initiates the ethernet device and starts a shell.

{% highlight bash %}
#!/bin/sh
	
/bin/mount -t proc none /proc
/bin/mount -t sysfs sysfs /sys
/sbin/mdev -s
/sbin/ifconfig lo 127.0.0.1 netmask 255.0.0.0 up
/sbin/ifconfig eth0 up 10.0.2.15 netmask 255.255.255.0 up
/sbin/route add default gw 10.0.2.2

echo 'Enjoy your Linux system!'
	
/usr/bin/setsid /bin/cttyhack /bin/sh
exec /bin/sh
{% endhighlight %}

Finally, make `/init` executable and create the *initial ramdisk*:

{% highlight bash %}
chmod 755 init
find . -print0 | cpio --null -ov --format=newc > ../myinitrd.cpio
{% endhighlight %}
	
### Kernel Configuration

To create a customized version of Linux for this memo, I downloaded the [source code](https://www.kernel.org) from Linux 3.19.5 and unpacked the tarball into a directory, which I called `linux`.
In most cases I start with a minimal configuration and add only what I need:

{% highlight bash %}
cd linux
make allnoconfig
make menuconfig
{% endhighlight %}

For this memo, I used a configuration which creates a small - but not the smallest - version of Linux.
It fits most of my use cases.

	CONFIG_64BIT=y
	CONFIG_EMBEDDED=n
	CONFIG_PRINTK=y
	CONFIG_PCI_QUIRKS=y
	CONFIG_BLK_DEV=y	
	CONFIG_BLK_DEV_INITRD=y
	CONFIG_INITRAMFS_SOURCE=/pathto/myinitrd.cpio
	CONFIG_BLOCK=y
	CONFIG_SMP=y
	CONFIG_PCI=y
	CONFIG_MTTR=y
	CONFIG_BINFMT_ELF=y
	CONFIG_BINFMT_SCRIPT=y
	CONFIG_EXT2_FS=y
	CONFIG_INOTIFY_USER=y
	CONFIG_NET=y
	CONFIG_PACKET=y
	CONFIG_UNIX=y
	CONFIG_INET=y
	CONFIG_WIRELESS=n
	CONFIG_ATA=y
	CONFIG_NETDEVICES=y
	CONFIG_NET_VENDOR_REALTEK=y
	CONFIG_8139TOO=y
	CONFIG_8139CP=y (unchecked all other Ethernet drivers)
	CONFIG_WLAN=n
	CONFIG_DEVKMEM=y
	CONFIG_TTY=y
	CONFIG_TTY_PRINTK=y
	CONFIG_SERIAL_8250=y
	CONFIG_SERIAL_8250_CONSOLE=y
	CONFIG_PROC_FS=y
	CONFIG_PROC_KCORE=y
	CONFIG_SYSFS=y
	CONFIG_X86_VERBOSE_BOOTUP=y
	CONFIG_EARLY_PRINTK=y
	
At last, I built the Linux kernel with the following command:

{% highlight bash %}
make
{% endhighlight %}

### Booting the customized kernel

Now we are ready to boot our virtual machine!

{% highlight bash %}
qemu-system-x86_64 -smp 2 -kernel /path-to/bzImage -initrd /path-to/myinitrd.cpio -append "root=/dev/ram0 rootfstype=ramfs init=init console=ttyS0" -net nic,model=rtl8139 -net user  -net dump -nographic
{% endhighlight %}

Instead of creating a dedicated window, I redirected the output of the virtual machines to the console of my host system.
This is realized by using the serial port (`ttyS0`) as the communication interface to the Linux system.
Consequently, I set the kernel paramater `console` to `ttyS0` and disabled the graphical output with `-nographic`.

That's it! Hopefully the tutorial about [kernel debugging](https://techblog.lankes.org/tutorials/kernel-debugging-with-qemu/) is interesting enough to start you in kernel development.
