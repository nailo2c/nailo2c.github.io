---
title: "LFX Mentorship W2 筆記"
date: "2026-06-17T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["LFX Mentorship"]
categories: ["Open Source"]

math:
  enable: true
---

第一週嘗試重現bug，但有些行為已經跟issue當時回報的不同，我會從那些不同的地方開始。

## 1. exit code 71 是什麼

透過加 log 在 `lib/host/wasi/wasifunc.cpp` 中，可以觀察到 exit code 71 是客戶自己加的

舉例:
```cpp
Expect<uint32_t> WasiArgsGet::body(const Runtime::CallingFrame &Frame,
                                   uint32_t ArgvPtr, uint32_t ArgvBufPtr) {
  spdlog::warn("[LFX] args_get ArgvPtr=0x{:x}, ArgvBufPtr=0x{:x}, misaligned={}"sv,
               ArgvPtr, ArgvBufPtr, isMisaligned<uint8_t_ptr>(ArgvPtr));

  // Alignment checks
  if (unlikely(isMisaligned<uint8_t_ptr>(ArgvPtr))) {
    spdlog::warn("[LFX] args_get returns __WASI_ERRNO_ADDRNOTAVAIL ({})"sv,
                 static_cast<uint32_t>(__WASI_ERRNO_ADDRNOTAVAIL));
    return __WASI_ERRNO_ADDRNOTAVAIL;
  }

  ...
}
```

log如下:
```console
[2026-06-17 15:24:23.204] [warning] [LFX] args_get ArgvPtr=0x10009, ArgvBufPtr=0x1002e, misaligned=true
[2026-06-17 15:24:23.205] [warning] [LFX] args_get returns __WASI_ERRNO_ADDRNOTAVAIL (4)
[2026-06-17 15:24:23.205] [warning] [LFX] proc_exit ExitCode=71
```

雖然 #2733 中的 hang 沒做出來（可能被handle掉了），但至少從 log 中能證明有 memory misaligned 的存在。

#2881 跟 #2694 還不知道為什麼 current master 用 testcase 重現不出 issue 中提到的症狀。

ok現在知道了，三個 issue 都有 misaligned WASI guest pointer，只是可能有某些 handle 將這個 misaligned 隱藏起來。

- #2694：fd_write misalignment → ADDRNOTAVAIL (4) → guest proc_exit(0)
- #2733：args_get misalignment → ADDRNOTAVAIL (4) → guest proc_exit(71)
- #2881：args_get misalignment → ADDRNOTAVAIL (4) → guest proc_exit(71)

## 2. alignment-behavior map

Wasm / WASI 有好幾種 alignment 情境，且規則不同，不能因為 address 沒對齊就一律報錯。

| Category | 例子 | Misaligned 時應如何處理 |
| --- | --- | --- |
| WASI guest pointer | args_get(argv) / fd_write(iovs) | typed pointer 通常需要 alignment check |
| Ordinary Wasm load/store | i32.load / i64.store | 即使 address unaligned，只要沒 OOB 仍應成功 |
| Alignment immediate | i32.load align=8 | 宣告的 alignment 超過 instruction 允許值時，validation 應失敗 |
| Atomic instruction | i32.atomic.load | effective address misaligned 時應 trap |
| SIMD/vector memory | v128.load 等 | 需要按 instruction 規格確認 |
| Compiled execution path | JIT / AOT | 應和 interpreter 實現同一種 observable behavior |

(Note: SIMD = Single Instruction, Multiple Data)

我們的目標是整理一張表，說明 WebAssembly 各種「讀寫記憶體」的指令遇到位址沒有對齊時，正確行為應該是什麼。

| Category | Alignment example | Spec rule | WasmEdge code | Status |
|---|---|---|---|---|
| Ordinary non-atomic load/store | `i32.load` from address `1` | Unaligned in-bounds access must succeed. [Execution](https://webassembly.github.io/spec/core/exec/instructions.html#memory-instructions), [validation](https://webassembly.github.io/spec/core/valid/instructions.html#memory-instructions) | 1. [**Validation**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/validator/formchecker.cpp#L290-L315)<br>2. [**Interpreter**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/include/executor/engine/memory.ipp#L13-L49)<br>3. [**JIT/AOT**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/llvm/compiler.cpp#L4789-L4871) | Needs testing in Interpreter/JIT/AOT |
| Bulk memory operations | `memory.copy` with source address `1` and destination address `3` | Byte offsets need not be aligned; OOB ranges trap. [`memory.copy`](https://webassembly.github.io/spec/core/exec/instructions.html#exec-memory-copy), [`memory.fill`](https://webassembly.github.io/spec/core/exec/instructions.html#exec-memory-fill), [`memory.init`](https://webassembly.github.io/spec/core/exec/instructions.html#exec-memory-init) | 1. [**Validation**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/validator/formchecker.cpp#L1256-L1280)<br>2. [**Interpreter**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/executor/engine/memoryInstr.cpp#L37-L128)<br>3. [**JIT/AOT**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/llvm/compiler.cpp#L1765-L1818) | Needs testing in Interpreter/JIT/AOT |
| SIMD/vector memory instructions | `v128.load` from address `1` | Unaligned in-bounds access must succeed. [Validation](https://webassembly.github.io/simd/core/valid/instructions.html#memory-instructions), [execution](https://webassembly.github.io/spec/core/exec/instructions.html#memory-instructions) | 1. [**Validation**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/validator/formchecker.cpp#L1476-L1516)<br>2. [**Interpreter**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/include/executor/engine/memory.ipp#L52-L180)<br>3. [**JIT/AOT**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/llvm/compiler.cpp#L4816-L4882) | Needs testing in Interpreter/JIT/AOT |
| Atomic memory instructions | `i32.atomic.load` from address `1` | Natural alignment is required; runtime misalignment traps. [Validation](https://webassembly.github.io/threads/core/valid/instructions.html#atomic-memory-instructions), [execution](https://webassembly.github.io/threads/core/exec/instructions.html#atomic-memory-instructions) | 1. [**Validation**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/validator/formchecker.cpp#L301-L315)<br>2. [**Interpreter**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/include/executor/engine/atomic.ipp#L15-L105)<br>3. [**JIT/AOT**](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/llvm/compiler.cpp#L3989-L4001) | Needs testing in Interpreter/JIT/AOT |
| WASI typed guest pointers | `args_get` with misaligned `argv`; `fd_write` with misaligned `iovs` | Typed pointers must be aligned; misalignment shall trap. [WASI pointer rule](https://github.com/WebAssembly/WASI/blob/v0.2.0/legacy/tools/witx-docs.md#pointers), [Basic C ABI](https://github.com/WebAssembly/tool-conventions/blob/main/BasicCABI.md#data-representation) | 1. [`args_get`](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/host/wasi/wasifunc.cpp#L374-L383)<br>2. [`fd_write`](https://github.com/WasmEdge/WasmEdge/blob/e0c2f83edb5b5fc151e1a94aeb7b81cc4852d858/lib/host/wasi/wasifunc.cpp#L1156-L1167) | Current master detects all three issue testcases and returns `__WASI_ERRNO_ADDRNOTAVAIL`; trap/diagnostic expectation needs confirmation |

+ Interpreter 入口: `engine.cpp` 的 opcode dispatch
+ JIT/AOT: `lib/llvm/compiler.cpp` 編譯成 native code
+ WASI host calls: 三種mode最後都進入 `wasifunc.cpp`，因此三種mode都會看到 `args_get`/`fd_write`

## 3. Week 2 總結

有了 Behavior Map 後，能知道各個行為的 code 入口在哪，同時了解這些行為。

當然 WasmEdge 不只這些 Behaviors，接下來會請教 mentors 要先做哪個。另外 Week 3 的目標是將此次 Mentorship 提到的三個 issue 精簡成一個 minimal testcase 能穩定重現，最後看能否轉為正式的 regression test。
