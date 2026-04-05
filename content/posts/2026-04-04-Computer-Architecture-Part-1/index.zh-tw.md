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

+ Stack Machine
(a + b * c) / (a + d * c - e) 可以轉化為:  
abc*+adc*+e-/

用stack的方式表達這個四則運算，難怪某些LeetCode要考這個資料結構

Stack的最大問題是理論上可以一直堆疊，但物理上很難做到那麼大

像上述的運算會變成:  

|                  |
|------------------|
| stack (size = 2) |
| R0               |
| R0 R1            |
| R0 R1 R2         |
| R0 R1            |
| R0               |
| R0 R1            |
| R0 R1 R2         |
| R0 R1 R2 R3      |
| R0 R1 R2         |
| R0 R1            |
| R0 R1 R2         |
| R0 R1            |
| R0               |


因為有缺點，所以架構才一路進化，Stack -> Accumulator -> Register-Memory -> Register-Register。

之前上的Nand2Tetris是接近Register-Register架構。

投影片介紹了一些 instruction 的定義（不同家的CPU會有不同定義）、Data Type、ISA Encoding的方法如Fixed Width跟Variable Length、

我認為主要在講歷史進程，以及當下的考量與trade-off。我不追求背起來，先留個印象以及當下設計的取捨。

ARM跟x86是不同的Architecture，擁有不同的ISA。而Apple的M系列晶片是做在ARM下的Microarchitecture，不自己做ISA的可能理由是生態系、相容性以及開發者友善不友善。

# Module 2

