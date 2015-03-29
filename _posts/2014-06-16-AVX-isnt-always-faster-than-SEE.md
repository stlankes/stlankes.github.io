---
layout: post
title: "AVX isn’t always faster than SEE"
date: 2014-06-16 
thumb: thumbnail-pi.png
tags: [x86]
share: true
comments: true
---

To explain vectorization via *Streaming SIMD Extensions* (SSE) I mostly use the simple calculation of π.
I calculate π by determining the integral between 0 and 1 of the function f(x) = 4 / (1+x*x).
The integral is the surface area below f(x) and could be approximated by the accumulation of the surface areas of small rectangles, which are drawn below f(x).

<figure>
<img src="/images/pi.png">
</figure>

The C-code is extremely simple.
In principal the heights of all rectangles will be accumulated and finally multiplied with the width, which is fix for all rectangles (1 / number of rectangles).

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>

double width;
double sum;
int num_rects = 1000000000;

void calcPi(void)
{
        float x;
        int i;

        for (i = 0; i < num_rects; i++) {
                x = (i + 0.5) * width;
                sum += 4.0 / (1.0 + x * x);
        }
}

int main(int argc, char **argv)
{
        struct timeval start, end;

        if (argc > 1)
                num_rects = atoi(argv[1]);
        if (num_rects < 100)
                num_rects = 1000000;
        printf("\nnum_rects = %d\n", (int)num_rects);

        gettimeofday(&start, NULL);

        sum = 0.0;
        width = 1.0 / (double)num_rects;

        calcPi();

        gettimeofday(&end, NULL);

        printf("PI = %f\n", sum * width);
        printf("Time : %lf sec\n", (double)(end.tv_sec-start.tv_sec)+(double)(end.tv_usec-start.tv_usec)/1000000.0);

        return 0;
}
{% endhighlight %}

Usually I write SSE code with [NASM](http://www.nasm.us/) in assembler -- I simply like to program assembler and it’s a fair enough reason to do it this way.
However, at the same time it increases the understanding of current computer architectures.
That’s why I frequently look at the new extensions of the instruction set.

In principle, with SSE you are able to store two double-precision floating point numbers in one register.
Consequently, you could see this as a small vector with two values.
To accelerate the calculation of π, I determine two rectangles’ heights in one pass.

{% highlight nasm %}
SECTION .data
ALIGN 16
four DQ 4.0, 4.0
two DQ 2.0, 2.0
one DQ 1.0, 1.0
ofs DQ 0.5, 1.5

SECTION .text

; defined in our C code (see example above) 
extern width,sum,num_rects

global calcPi_SSE

calcPi_SSE:
    push ebp
    mov ebp, esp
    push ebx
    push ecx

    xor ecx, ecx             ; ecx = i = 0
    xorpd xmm0, xmm0         ; xmm0 = sum = 0
    movsd xmm1, [width]      ; set xmm1 to width
    shufpd xmm1, xmm1, 0x0
    movapd xmm2, [ofs]       ; set xmm2 to (0.5, 1.5)

L1:
    cmp ecx, [num_rects]     ; check termination condition
    jge L2
    ; calculate (i+0.5)*width
    movapd  xmm4, xmm1
    mulpd   xmm4, xmm2
    ; square the interim result
    ; and add one to it
    mulpd xmm4, xmm4
    addpd xmm4, [one]
    ; divide 4 by the interim result
    movapd  xmm3, [four]
    divpd   xmm3, xmm4
    ; accumulate heights of the rectangles
    addpd xmm0, xmm3
    ; increase loop counters and
    ; jump to the beginning of the loop
    addpd xmm2, [two]
    add ecx, 2
    jmp L1
L2:
    ; accumulate all element of xmm0 (= all heights)
    ; to one result (= first element of xmm3)
    xorpd xmm3,xmm3
    addsd xmm3, xmm0
    shufpd xmm0, xmm0, 0x1
    addsd xmm3, xmm0
    movsd [sum], xmm3

    pop ecx
    pop ebx
    pop ebp
    ret
{% endhighlight %}

I tried to update my example and moved to the modern *Advanced Vector Extensions* (AVX).
AVX increases the register size to 256 bit, which allows us to store four double-precision floating point numbers in one register.
AVX comes with a new instruction set.
However, the differences between AVX and SSE are small. The new prefix "v" is added to the instructions and they have now three (instead of two) operands.
The additional operand is the destination register of the calculation.

In theory, the calculation of π will be accelerated by factor of two, if we use AVX and double the number of calculations per iteration.
The one-to-one translation of the SSE to AVX code is as follows:
{% highlight nasm %}
SECTION .data
ALIGN 32
four DQ 4.0, 4.0, 4.0, 4.0
two DQ 2.0, 2.0, 2.0, 2.0
one DQ 1.0, 1.0, 1.0, 1.0
ofs DQ 0.5, 1.5, 2.5, 3.5

SECTION .text

extern width,sum,num_rects

global calcPi_AVX

calcPi_AVX:
     push ebp
     mov ebp, esp
     push ebx
     push ecx

     xor ecx, ecx               ; ecx = i = 0
     vxorpd ymm0, ymm0, ymm0    ; ymm0 = sum = 0
     vbroadcastsd ymm1, [width] ; set ymm1 to step
     vmovapd ymm2, [ofs]        ; set ymm2 to (0.5, 1.5, 2.5, 3.5)
     vmovapd ymm3, [four]       ; set ymm3 to (4.0, 4.0, 4.0, 4.0)

L1:
     cmp ecx, [num_rects]       ; check termination condition
     jge L2
     ; calculate (i+0.5)*width
     vmulpd  ymm4, ymm1, ymm2
     ; square the interim result
     ; and add one to it
     vmulpd ymm4, ymm4, ymm4
     vaddpd ymm4, ymm4, [one]
     ; divide 4 by the interim result
     vdivpd  ymm4, ymm3, ymm4
     ; accumulate heights of the rectangles
     vaddpd ymm0, ymm0, ymm4
     ; increase loop counters and
     ; jump to the beginning of the loop
     vaddpd ymm2, ymm2, ymm3
     add ecx, 4
     jmp L1
L2:
     ; accumulate all element of xmm0 (= all heights)
     ; to one result (= first element of xmm3)
     vperm2f128 ymm3, ymm0, ymm0, 0x1
     vaddpd ymm3, ymm3, ymm0
     vhaddpd ymm3, ymm3, ymm3
     vmovsd [sum], xmm3

     pop ecx
     pop ebx
     pop ebp
     ret
{% endhighlight %}

Unfortunately, the AVX code is not faster than the SSE code. 
To understand such performance issues, I suggest a look at [Agner Fog’s website](http://www.agner.org/).
Especially his [lists of instruction latencies, throughputs and micro-operation breakdowns](http://www.agner.org/optimize/instruction_tables.pdf) are great.
He analyzed that on Intel’s Haswell processor the instruction vdivpd (AVX) has a latency, which is twice as high as divpd (SSE).
In our example, it destroys any performance benefit of AVX.

This behavior surprised me. However, it is extremely simple to solve the performance issues.
Intel already described the solution in its 1999 document *Increasing the Accuracy of the Results from the Reciprocal and Reciprocal Square Root Instructions using the Newton-Raphson Method*. 
In principle, we have to estimate the reciprocal of (1+x*x) with the instruction vrcpps, which is extremely fast.
However, the result is only accurate in the 12 most significant bits of the mantissa, which is not accurate enough to get single-precision floating-point numbers -- and our aim is to get double-precision.
With the  Newton-Raphson Method we are able to increase the precision.
In our case, the iterative method has the following iteration loop, where x1 represents the new and more accurate value, x0 the value of the previous cycle or in the first pass the initial value from vrcpps, and d the double-precision result of (1 + x*x):

> x1 = x0 * (2 – d * x0) = 2 * x0 – d * x0 * x0

To get a double-precision result of 1/(1+x*x), we have to run through the iterative loop twice.
Consequently, we have to replace line 36 of the above AVX code by the following lines:

{% highlight nasm %}
vmovapd   ymm5, ymm4
vmovapd   ymm6, ymm4
vcvtpd2ps xmm4, ymm5  ; convert the result of (1+x*x) in single-percision number
vrcpps    xmm4, xmm4  ; estimate reciprocal 
vcvtps2pd ymm4, xmm4  ; convert result to a double-percision number
; first pass to the iterative loop
vmulpd    ymm5, ymm4
vmulpd    ymm5, ymm4
vaddpd    ymm4, ymm4
vsubpd    ymm4, ymm5
; second pass to the iterative loop
vmulpd    ymm6, ymm4
vmulpd    ymm6, ymm4
vaddpd    ymm4, ymm4
vsubpd    ymm4, ymm6
; multiply with 4 to determine 4 / (1+x*x)
vmulpd    ymm4, ymm3
{% endhighlight %}

Voilà, my AVX code is now twice as fast as my SSE code. :-)

> Time: 5.705789 sec (C code)
> Time: 2.213902 sec (SSE)
> Time: 1.172251 sec (AVX)
