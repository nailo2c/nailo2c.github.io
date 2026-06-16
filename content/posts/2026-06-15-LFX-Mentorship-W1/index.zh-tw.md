---
title: "LFX Mentorship W1 筆記"
date: "2026-06-15T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["LFX Mentorship"]
categories: ["Open Source"]

math:
  enable: true
---

很幸運的入選了 LFX Mentorship WasmEdge 項目，簡單紀錄 Week 1 的一些心得。

首先這個 project 是 async 溝通的，不會有線上會議。會要求 mentee 創一個 Workspace issue，然後以那個 issue 為主要討論的媒介。

在開始做 issue 前我需要對 WasmEdge 有基本的了解。

## 1. WasmEdge 是什麼

WasmEdge 是一個 runtime，他能執行各種語言（C/C++、Rust、Swift、etc）編譯成的 `.wasm`。

WasmEdge 根據 Edge Computing 做了最佳化以及高安全性，能妥善隔離作業系統資源。根據官方測試，它是市面上執行速度最快的 Wasm Runtime 之一。

啟動速度極快、體積小、安全，因此現在很流行將 WasmEdge 取代 Docker 跑在雲端環境中。

## 2. WasmEdge 內部的 Compiler

WasmEdge 雖然是 Runtime，但它內部有 Interpreter / Compiler 負責 執行/編譯`.wasm`。

1. Interpreter (Default) - 直接讀取 Wasm bytecode 然後執行對應的 C/C++ 等底層語言實作邏輯。
2. AOT (Ahead-of-Time) - 呼叫 LLVM 後端把 `.wasm` 編譯成對應的CPU架構（x86_64/ARM64/etc）的machine code。
3. JIT (Just-in-Time) - 先用 Interpreter 啟動，執行過程中，Runtime分析哪部分被頻繁呼叫，然後在背景把這些熱點程式編譯成machine code。

## 3. Wasm 是什麼

Wasm = Web Assembly。它是虛擬的 Assembly，傳桶 Assembly 是給機器看的，而 Web Assembly 是給「虛擬機器」看的，類似JVM的概念可以跨平台。

Assembly 不跨平台，在 x86 編譯出的 machine code 是不能跑在 ARM 上的。

Web Assembly 則是跑在 Runtime 上（例如 WasmEdge），因此不論底層是 Windows x86 或是 Linux ARM，都能跑同一份 `.wasm`。

最初 Web Assembly 是給 Web 用的，開發者把 C/C++ 編譯成 `.wasm` 後，可以丟給瀏覽器，讓瀏覽器的 Wasm 引擎編譯 + 執行。

後來Wasm的優點（跨平台、安全性、啟動速度快、體積小），被大家拿去應用在雲端伺服器或IoT的邊緣運算上。

## 4. 目前的問題

此次 LFX Mentorship 要解:

1. issue 2694 - misaligned pointer 發生後 WasmEdge 會繼續執行並印出 checksum。預期不應該 silent success，應在對應的 WASI pointer / alignment 檢查處 fail 或 trap
2. issue 2733 - 舊版 WasmEdge 會 hang 住，但新版本會返回 exit=71，需要調查目前的output是否符合預期
3. issue 2881 - AOT mode 可能會報 OOB 或是 hang，預期要行為一致成報 misaligned pointer error

(WASI = WebAssembly System Interface)

Local re-produce issue 後的結果如下:

| Issue   | Mode       | Input | Exit | Timeout | stdout | stderr / diagnostic | Notes |
|---------|------------|-------|-----:|---------|--------|---------------------|-------|
| #2694 | Interpreter| mutated.wasm | 0 | no | empty | empty | 舊版 checksum 現象未重現，但仍有 silent success |
| #2694 | JIT        | mutated.wasm | 0 | no | JIT compile logs only | empty | 沒有 checksum/trap，有 AOT/JIT exception-handling warning |
| #2694 | AOT        | mutated.aot.wasm | 0 | no | empty | empty | AOT compile exit 0; compile stderr 有 lld version warning |
| #2733 | Interpreter | mutated_file_28840.wasm | 71 | no | empty |empty | hang 未重現，最新的 master 快速 exit=71 |
| #2733 | JIT | mutated_file_28840.wasm | 71 | no | JIT compile logs only |empty | 執行成功但有 JIT exception-handling warning |
| #2733 | AOT | mutated_file_28840.aot.wasm | 71 | no | empty | empty | 執行成功，但快速 exit=71 |
| #2881 | Interpreter | 71 | no | empty | empty | OOB/hang 未重現，最新的 master 快速 exit=71 |
| #2881 | JIT | 71 | no | JIT compile logs only | empty | 執行成功但有 JIT exception-handling warning |
| #2881 | AOT | 71 | no | empty | empty | 原本會報 OOB 的那個檔案現在以 exit code 71 結束 |
| #2881 | AOT | 71 | no | empty | empty | 原本會 hang 的那個檔案現在以 exit code 71 結束 |

## 5. 展望下一週

有些原始 issue 提到的問題都沒有順利復現，例如 OOB 或 hang，反而都很快地返回 exit code 71。

下一週會嘗試找出原因，以及 exit code 71 所代表的意思。
