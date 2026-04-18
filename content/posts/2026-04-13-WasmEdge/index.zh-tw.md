---
title: "WasmEdge初探"
date: "2026-03-29T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["LXF Mentorship"]
categories: ["Open Source"]

math:
  enable: true
---

根據GSoC的結論，我覺得應該不會上，競爭太激烈的，於是我決定先來佈局LXF Mentorship。

我的目標依舊是就業導向，我想增加C++跟偏硬體的能力，於是我選中了WasmEdge專案。

# WasmEdge是什麼?

想先了解WasmEdge，就先得了解WebAssembly。

我們都知道不同架構的CPU之下編譯出來的machine code不能互通，而WebAssembly則是一種通用的二進位指令格式，不管是C、C++、Rust編譯出來的`.wasm`檔案，只要有Wasm runtime就能執行，不用去管底下的OS或硬體。

而WasmEdge則是把WebAssembly針對雲端與邊緣運算場景做了最佳化。

# 底層技術

1. Ahead-of-Time Compiler
    - WasmEdge基於LLVM的AOT編譯器，讓它成為市面上最快的WebAssembly runtime之一。
2. Sandbox Isolation
    - Wasm程式不能直接碰到你的檔案系統、網路、或記憶體中的其他程式。跟Docker container的隔離概念相似，但Wasm的隔離是在指令層級就做好，底層更輕量。
3. WASI (WebAssembly System Interface)
    - Wasm原本是給瀏覽器用的，WASI讓Wasm程式可以安全地存取作業系統功能。

# 跟Docker比較

1. Docker container
    1. containerd 呼叫 runc
    2. 建立一整組 Linux namespace, cgroup, mount - 一個完整的root filesystem
    3. 啟動一個Linux process
    4. 上述可能幾十到幾百MB
2. Wasm container
    1. containerd 呼叫 runwasi
    2. 啟動 WasmEdge runtime
    3. 直接執行 `.wasm`
    4. 上述可能幾MB甚至幾百KB

# 現實世界的應用

1. LlamaEdge
    - 在個人電腦與邊緣裝置上運行生成式AI，因為如果用PyTorch來運行太肥了(> 5GB)，改用LlamaEdge runtime不到30MB
2. Docker + Wasm
    - 目前WasmEdge已經與Docker有著不錯的整合
3. 區塊鏈智能合約
    - Wasm的沙盒特性讓它適合跑智能合約，因為你不希望別人寫的智能合約能存取你節點上的檔案
4. Kubernetes
    - Second State（WasmEdge 背後的公司）協助客戶在K8s邊緣從集中部署WasmEdge微服務

# 然後呢?

把 [Getting Started](https://wasmedge.org/docs/start/overview/) 摸一摸，發現了新手教學文件有些地方過時了順便修一修。

看了一下 [Contributing Guide](https://wasmedge.org/docs/contribute/)，順便修了一些typo跟排版。

接下來應該就等 term 2 出來，看有沒有適合我的題目，說不定 term 2 根本不會有 WasmEdge，我白忙一場（昏）。
