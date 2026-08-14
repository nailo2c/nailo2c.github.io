---
title: "LFX Mentorship W3 筆記"
date: "2026-06-29T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["LFX Mentorship"]
categories: ["Open Source"]

math:
  enable: true
---

目前行為：

guest calls args_get
-> WasmEdge detects misaligned ArgvPtr
-> args_get returns __WASI_ERRNO_ADDRNOTAVAIL (4)
-> guest code 繼續跑
-> guest 可以自己決定 proc_exit(4) 或忽略這個 errno

trap 行為：

guest calls args_get
-> WasmEdge detects misaligned ArgvPtr
-> host function returns an error/trap to WasmEdge runtime
-> guest code 不會收到 args_get 的 return value
-> 後面的 Wasm instructions 不會繼續執行
-> CLI 顯示 execution failed / host function failed / 某個 ErrCode diagnostic
