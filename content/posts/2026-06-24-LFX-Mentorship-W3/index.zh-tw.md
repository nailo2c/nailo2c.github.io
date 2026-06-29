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

## Task 2 - 實作最小重現wat檔

`args_get` aligned 的 `.wat` 的最小重現範例如下:
```wat
;; wat2wasm LFX/w3/args_get-aligned.wat -o LFX/w3/args_get-aligned.wasm
;; build-lfx-repro/tools/wasmedge/wasmedge --force-interpreter LFX/w3/args_get-aligned.wasm
;; echo "exit=$?"
;; JIT:
;;   build-lfx-repro/tools/wasmedge/wasmedge --enable-jit LFX/w3/args_get-aligned.wasm
;;   echo "aligned-jit exit=$?"
;; AOT:
;;   build-lfx-repro/tools/wasmedge/wasmedgec LFX/w3/args_get-aligned.wasm LFX/w3/args_get-aligned.aot.wasm
;;   echo "aligned-aot exit=$?"
(module
    (import "wasi_snapshot_preview1" "args_get"
        (func $args_get (param i32 i32) (result i32)))

    (import "wasi_snapshot_preview1" "proc_exit"
        (func $proc_exit (param i32)))

    (memory (export "memory") 1)

    (func (export "_start")
        i32.const 4
        i32.const 64
        call $args_get
        call $proc_exit
    )
)
```

`args_get` misaligned 的 `.wat` 的最小重現範例如下:
```wat
;; wat2wasm LFX/w3/args_get-misaligned.wat -o LFX/w3/args_get-misaligned.wasm
;; build-lfx-repro/tools/wasmedge/wasmedge --force-interpreter LFX/w3/args_get-misaligned.wasm
;; echo "exit=$?"
;; JIT:
;;   build-lfx-repro/tools/wasmedge/wasmedge --enable-jit LFX/w3/args_get-misaligned.wasm
;;   echo "misaligned-jit exit=$?"
;; AOT:
;;   build-lfx-repro/tools/wasmedge/wasmedgec LFX/w3/args_get-misaligned.wasm LFX/w3/args_get-misaligned.aot.wasm
;;   echo "misaligned-aot exit=$?"
(module
    (import "wasi_snapshot_preview1" "args_get"
        (func $args_get (param i32 i32) (result i32)))

    (import "wasi_snapshot_preview1" "proc_exit"
        (func $proc_exit (param i32)))

    (memory (export "memory") 1)

    (func (export "_start")
        i32.const 5
        i32.const 64
        call $args_get
        call $proc_exit
    )
)
```

`fd_write` aligned 的 `.wat` 的最小重現範例如下:
```wat
(module
    (import "wasi_snapshot_preview1" "fd_write"
        (func $fd_write (param i32 i32 i32 i32) (result i32)))

    (import "wasi_snapshot_preview1" "proc_exit"
        (func $proc_exit (param i32)))

    (memory (export "memory") 1)

    (data (i32.const 16) "\40\00\00\00\03\00\00\00")  ;; buf pointer = 64
    (data (i32.const 32) "\00\00\00\00")  ;; initial nwritten = 0
    (data (i32.const 64) "hi\0a")  ;; write bytes = "hi\n"

    (func (export "_start")
        i32.const 1   ;; Fd = stdout
        i32.const 16  ;; IOVsPtr = aligned ciovec address
        i32.const 1   ;; IOVsLen = one ciovec
        i32.const 32  ;; NWrittenPtr = aligned write-back address
        call $fd_write
        call $proc_exit
    )
)
```

`fd_write` misaligned 的 `.wat` 的最小重現範例如下:
```wat
(module
    (import "wasi_snapshot_preview1" "fd_write"
        (func $fd_write (param i32 i32 i32 i32) (result i32)))

    (import "wasi_snapshot_preview1" "proc_exit"
        (func $proc_exit (param i32)))

    (memory (export "memory") 1)

    (data (i32.const 16) "\40\00\00\00\03\00\00\00")  ;; buf pointer = 64
    (data (i32.const 32) "\00\00\00\00")  ;; initial nwritten = 0
    (data (i32.const 64) "hi\0a")  ;; write bytes = "hi\n"

    (func (export "_start")
        i32.const 1   ;; Fd = stdout
        i32.const 17  ;; IOVsPtr = misaligned ciovec address
        i32.const 1   ;; IOVsLen = one ciovec
        i32.const 32  ;; NWrittenPtr = aligned write-back address
        call $fd_write
        call $proc_exit
    )
)
```

## Task 3 - 測試其他 categories

根據 Week 2 的 alignment-behavior map ，其他四個 categories 也做類似 Task 2 的事情：實作 `wat`，測試 Interpreter/JIT/AOT。

測試結果是其他四個 categories 都沒問題，不會有 misaligned 然後返回不預期結果的狀況，因此下一週會 focus 在寫 `args_get` 跟 `fd_write` 的 test 以及修復。
