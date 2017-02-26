---
layout: post
title: "The 'smallest' HelloWorld of the World (or not)"
date: 2015-03-20
thumb: thumbnail-eduos.jpg
tags: [Tutorials]
comments: true
share: true
---

I want to show how a kernel can be loaded into the memory and
print some messages on the screen. At this point, the kernel is only a
*HelloWorld application*, which runs bare-metal on the hardware. The
example code is published as branch *stage0* at the GitHub repository
[https://github.com/RWTH-OS/eduOS](https://github.com/stlankes/eduOS).
Therefore, it is possible to get access to the code with the following
commands:

{% highlight bash %}
git clone https://github.com/RWTH-OS/eduOS eduOS
cd eduOS
git checkout stage0
{% endhighlight %}

It is an extremely difficult task to load code from a hard disk into the
memory without the support of an operating system (with its hard disk
drivers) is an extremely difficult task. However, a boot loader like
[grub](http://www.gnu.org/s/grub/) simplifies the procedure. The easiest
way is to use grub’s [Multiboot
Specification](http://www.gnu.org/software/grub/manual/multiboot/),
which is also supported by the generic machine emulator and virtualizer
[qemu](http://www.qemu.org).

The main task of the Multiboot Specification is the definition of boot
loader/OS image interface. The following three main aspects are defined
by the specification:

1.  The format of an OS image as seen by a boot loader.
2.  The state of a machine when a boot loader starts an operating
    system.
3.  The format of information passed by a boot loader to an operating
    system.

The boot loader already initializes the machine into the 32bit protected
mode (**without** paging). Our OS image uses the Executable and Linkable
Format (ELF), which is a common standard file format for executables,
object code and shared libraries. An advantage of the using ELF is the
possibility to analyze the image with standard tools like `objdump`.
However, the boot loader
has to detect that the image is a valid OS kernel and not a normal
executable. The Multiboot Specification claims a Multiboot header in the
first 8192 bytes of the image. The header contains at least three 32bit
numbers, which are a magic *multi boot* number (0x1BADB002) , a flag to
specifiy features that the OS image requests and a check sum. In
assembler the creation of  such a header is relatively simple and could
be realized as follows:

{% highlight nasm %}
[BITS 32]
SECTION .mboot
global start
start:
jmp stublet

; This part MUST be 4 byte aligned
ALIGN 4
mboot:

; Multiboot macros to make a few lines more readable later
MULTIBOOT_PAGE_ALIGN   equ 1 << 0
MULTIBOOT_MEMORY_INFO  equ 1 << 1 MULTIBOOT_HEADER_MAGIC equ 0x1BADB002 MULTIBOOT_HEADER_FLAGS equ MULTIBOOT_PAGE_ALIGN | MULTIBOOT_MEMORY_INFO MULTIBOOT_CHECKSUM     equ -(MULTIBOOT_HEADER_MAGIC + MULTIBOOT_HEADER_FLAGS) ; This is the GRUB Multiboot header => A boot signature
dd MULTIBOOT_HEADER_MAGIC
dd MULTIBOOT_HEADER_FLAGS
dd MULTIBOOT_CHECKSUM

SECTION .text
ALIGN 4
stublet:
{% endhighlight %}

Later, the label *start* will be the entry point of our kernel. The
lines 19-21 define the three 32bit numbers, which specify the image as
multiboot image. The feature flag defines that the kernel has to align
to a 4K boundary (`MULTIBOOT_PAGE_ALIGN`) and the
size of the available memory will be stored in the multiboot information
structure (`MULTIBOOT_MEMORY_INFO`)

Finally, we have to guarantee that the OS image begins with the section mboot
and consequently that the
magic *multi boot* number is located in the first 8192 bytes. Linker
scripts give us the possibility to describe how the sections in the
input files should be mapped into the output file, and to control the
memory layout of the output file. We use the following linker script for
our OS image:

{% highlight text %}
OUTPUT_FORMAT("elf32-i386")
OUTPUT_ARCH("i386")
ENTRY(start)
phys = 0x00100000;

SECTIONS
{
  kernel_start = phys;
  .mboot phys : AT(ADDR(.mboot)) {
    *(.mboot)
  }
  .text ALIGN(4096) : AT(ADDR(.text)) {
    *(.text)
  }
  .rodata ALIGN(4096) : AT(ADDR(.rodata)) {
    *(.rodata)
    *(.rodata.*)
  }
  .data ALIGN(4096) : AT(ADDR(.data)) {
    *(.data)
  }
  bss_start = .;
  .bss ALIGN(4096) : AT(ADDR(.bss)) {
    *(.bss)
  }
  bss_end = .;
  kernel_end = .;
}
{% endhighlight %}

Line 3 specifies that the symbol `start` is the entry point of the OS image, while line 4 defines the address where the OS image has to be loaded.
The structure `SECTIONS` (lines 6-28) describes the order of the sections and guarantees that `mboot` is located at the beginning of the image.
Furthermore, the linker script specifies the variable `kernel_start`, `kernel_end`, `bss_start` and `bss_end`.
With the addresses of the variable, I am later able to determine the location and the size of the OS image and its BSS segment (zero-valued statically-allocated variables).

Finally, we want to enter *normal* C code as soon as possible. This
requires to initialize a stack, which is needed by the C calling
convention:

{% highlight nasm %}
SECTION .text
ALIGN 4
stublet:
; initialize stack pointer.
    mov esp, default_stack_pointer
; interpret multiboot information
    extern multiboot_init
    push ebx
    call multiboot_init
    add esp, 4

; jump to the boot processors's C code
    extern main
    call main
    jmp $  ; endless loop

; Here is the definition of our stack. Remember that a stack actually grows
; downwards, so we declare the size of the data before declaring
; the identifier 'default_stack_pointer'
SECTION .data
    resb 8192               ; This reserves 8 KBytes of memory here
default_stack_pointer:
{% endhighlight %}
	
Line 21 reserves 8 KByte of memory, which is used as stack.
Therefore, the stack pointer (`esp`) is accordingly initialized with the stack (line 5).
Afterwards, we could enter any C function.
Before the kernel will enter the `main` function, the function `multiboot_init` is called. 
The Multiboot Specification defines a data structure, which contains e.g. the size of the available memory.
The boot loader creates the structure at boot time and stores its address in `ebx`. Therefore, `multiboot_init` saves the register to a global variable.
Finally, the kernel enters its main function.

{% highlight c %}
/* 
 * Note that linker symbols are not variables, they have no memory allocated for
 * maintaining a value, rather their address is their value.
 */
extern const void kernel_start;
extern const void kernel_end;
extern const void bss_start;
extern const void bss_end;

static int eduos_init(void)
{
        // initialize .bss section
        memset((void*)&bss_start, 0x00, ((size_t) &bss_end - (size_t) &bss_start));

        koutput_init();

        return 0;
}

int main(void)
{
        eduos_init();

        kprintf("This is eduOS %s\n", EDUOS_VERSION);
        kprintf("Kernel starts at %p and ends at %p\n", &kernel_start, &kernel_end);
        kprintf("\nHello World!\n");

        return 0;
}
{% endhighlight %}

We have almost reached our goal now: In `eduos_init`, we have to initialize the BSS segment.
The location and size of this segment is determined by the global variables, which are specified in the linker script.
The implementation of `kprintf` is derived from BSD Unix.
I will not explain the details of the implementation because parsing of the format string is not a kernel specific problem.
Finally, the messages have to be printed on the screen.
Fortunately, a VGA video card makes this a simple task.
It gives us a chunk of memory that we write characters on in order to show information on the screen.
I will be explain the details in another tutorial, which is focused on I/O handling.


Next, we need to compile our kernel.
I added a Makefile to the source code, which simplifies the building process.
One compiler flags is not typical for user-level applications.
A kernel is not able to use standard C library functions. Therefore, `-nostdinc` and `-fno-builtin` are used to avoid these functions.
If we need *standard* functions like `memset` (see above), we have to rewrite the function by ourselves.

Now, we could test our *HelloWorld* kernel. I use qemu as a generic machine emulator to test my approach. With

{% highlight bash %}
qemu-system-i386 -monitor stdio -kernel eduos.elf
{% endhighlight %}

the kernel will start on a virtual machine and dump *HelloWorld* on the
screen.

I hope that this first tutorial has given you a deeper understanding of designing low-level software.
