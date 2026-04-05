---
title: "Computer Architecture (Part 1)"
date: "2026-04-04T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["Study"]
categories: ["Study"]

math:
  enable: true
---

Princeton Computer Architecture課程的快速筆記

# Module 1

Computer Architecture就是學習Application到Physics之間的架構，因為我們需要很多抽象層來讓Application能方便地達到Physics。

|                                   |                                        |
|-----------------------------------|----------------------------------------|
| Application                       |                                        |
| Algorithm                         |                                        |
| Programming Language              |                                        |
| Operating System/Virtual Machines | Computer Architecture focus 在這些層級  |
| Instruction Set Architecture      | Computer Architecture focus 在這些層級  |
| Microarchitecture                 | Computer Architecture focus 在這些層級  |
| Register-Transfer Level           |                                        |
| Gates                             |                                        |
| Circuits                          |                                        |
| Devices                           |                                        |
| Physics                           |                                        |

根據摩爾定律，我們的電晶體會越來越強，此時會需要更好的Computer Architecture來讓他發揮應有的效能。

三種加速Processor的方法:
1. Parallelism
2. 精簡工作量
3. Cache

## Parallelism

1. Instruction Level Parallelism
    1. Superscalar
    2. Very Long Instruction Word (VLIW)
2. Long Pipelines (Pipeline Parallelism)
3. Data Level Parallelism
    1. Vector
    2. GPU
4. Thread Level Parallelism
    1. Multithreading
    2. Multiprocessor
    3. Multicore
    4. Manycore

## Architecture and Microarchitecture

早期IBM四個型號的產品架構都不同，因此也完全不相容。因此推出了 IBM 360 這個 Architecture，讓大家共享 ISA (Instruction Set Architecture)，後續四個架構就都能自己實作各自的datapath, decode, pipeline之類。

1 byte = 8-bits 是當年IBM 360發明的。

IBM發展出來的就是大名鼎鼎的x86架構。

