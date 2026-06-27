---
title: "LFX Mentorship W3 筆記"
date: "2026-06-24T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["LFX Mentorship"]
categories: ["Open Source"]

math:
  enable: true
---

本週的目標是將各個 issue 遇到的問題收斂成可重複執行的 regression test。

## Task 1 - Understand `args_get` and `fd_write`

第一步就是先 trace code，`args_get` 於 `lib/host/wasi/wasifunc.cpp`。

+ `WasiArgsGet` 有兩個參數
    + `ArgvPtr` 指向一個 32-bit pointer array; 每個元素指向參數字串
    + `ArgvBufPtr` 指向實際存放參數字串的 byte buffer
    + 該 function 目前 misaligned 時會回傳 `ADDRNOTAVAIL(4)`，理論上應該要 trap
+ 設計 aligned control case
    + 我選擇 ArgvPtr=4, ArgvBufPtr=64
    + Address: 4  5  6  7 (4-byte pointer, 因為是 WASM32, 32 bits = 4 bytes)
    + Value  : 64 0  0  0
    + 例子:
    + addresses 4-7  -> value 64 -> address 64-69 -> "hello\0"
    + addresses 8-11 -> value 70 -> address 70-74 -> "world\0"
