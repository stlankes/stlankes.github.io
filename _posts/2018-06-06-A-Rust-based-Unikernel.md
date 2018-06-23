---
layout: post
title: "A Rust-based Unikernel"
date: 2018-06-06
thumb: devnull.jpg
tags: [Unikernels, Rust, link post]
link: https://hermitcore.org/2018/06/06/A-Rust-based-Unikernel/
share: true
---

Rust is an extremely interesting language for the development of system software.
It promises a secure memory handling enabled and a zero-cost abstraction.
Projects like Redox and this blog posts from Philipp Oppermann show the feasibility of writing an OS kernel with Rust in an expressive, high level way without worrying about undefined behavior or memory safety.
Rust features like iterators, closures, pattern matching, option and result, string formatting, and the ownership system are still usable for a kernel developers.

Today, we published our version of HermitCore, where the kernel is written in Rust.
More information are published at [HermitCore's web site](https://hermitcore.org/2018/06/06/A-Rust-based-Unikernel/).
