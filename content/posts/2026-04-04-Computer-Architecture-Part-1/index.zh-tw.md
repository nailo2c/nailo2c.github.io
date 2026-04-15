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

1. Structural hazards：硬體資源衝突
2. Data hazards：資料相依衝突
3. Control hazards：控制流程不確定

Microcoded: 把ISA指令在CPU內拆成更多小步驟  
e.g. `ADD R1, R2, R3`
1. 讀出 R2
2. 讀出 R3
3. 送進 ALU
4. 做加法
5. 把結果寫回 R1

一個 cycle 的定義是一個 clock period  
1 GHz = 每秒 10^9 個 cycle  
此時 period = 1 ns

+ pipelined processor vs unpipelined control  
    + 前者可以多條指令存在不同的stages，例如第 1 條 instruction 在 EX 時，第 2 條可以在 ID，第 3 條可以在 IF  
    + 後者沒有 instruction overlap，例如第 1 條 instruction 走 IF  
        + 再走 ID  
        + 再走 EX  
        + 再走 MEM  
        + 再走 WB  
        + 全部做完後，第 2 條 instruction 才開始

每個instruction會執行的cycles可能都不同

## Structural Hazards

三種解決辦法: Schedule、Stall、加更多資源

+ 範例
LW : F D X M W  
ADD:   F D X M W  
ADD:     F D X M W  
ADD:       - F D X M W  

上述的第三個ADD需要stall一個cycle，因為Fetch跟Memory的load是不能同時操作的。

+ 多加一個Memory
ADD: F D X M0 M1 W  
ADD:   F D X  M0 M1 W  
LW :     F D  X  M0 M1 W  
LW :       F  D  X  -  M0 M1 W  
ADD:          F  D  -  X  M0 M1 W  

第一個stall: W, M1, M2 會因為搶memory datapath而產生structure hazard  
第二個stall: 由於LW2的X被stall了，卡住了ADD3的X，因此ADD3的X需要等LW2的X執行完才能執行  

## Data Hazards

e.g.  
I1: add r1, r2, r3  
I2: sub r4, r1, r5  
I2需要I1的r1  

資料相依問題發生時，通常採用以下四種策略
1. Schedule: 在instruction間安插一個不相關的instruction
2. Stall: 先等一下讓資料產生
3. Bypass: I1的X算出結果後直接先給I2，因此I2不必等I1的M跟W
4. Speculate: 先假設沒問題，照跑。萬一猜錯了就把錯誤丟回去重跑

有專門的一個元件在偵測hazard/stall，課堂花很多篇幅介紹這個元件的原理，例如他會需要知道每種 instruction 到底讀哪些 register、寫哪個 register 之類的。

Memory address dependency會造成stall

架構中可以加Bypass，也就是將比較後面stage的output提前送回ALU，可以解決某些情況下的stall/data hazard
