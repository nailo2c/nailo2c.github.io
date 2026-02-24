---
title: "Azure VM Operator實作過程紀錄"
date: "2026-02-19T00:00:00-00:00"
draft: false
description: "此篇文章記錄了個人思考與開發 Issue #49796 的過程"
featuredImage: "airflow_feat_implementation.png"

tags: ["Airflow"]
categories: ["Open Source"]

math:
  enable: true
---

最近弄到了免費的Azure credit，想說趁機把Airflow上有關Azure的issue解一解。解了幾個簡單的後，看到了[#49796](https://github.com/apache/airflow/issues/49796)覺得滿有趣的，就把它pick起來增加開發Airflow feature的經驗。

### 準備

在開始弄之前，我先檢查這個需求是否合理。首先Azure VM對標AWS的EC2以及GCP的GCE，而後兩者都有這個feature，因此我認定這個需求是合理的。

### 實作

接著就請Claude Code參考EC2跟GCE的實作，很快就做了出來，但此時遇到了async的問題導致System Test在設定`deferrable=True`的情況下會卡死。正好對Airflow Trigger也不算太了解，可以趁機去理解一下。

### 問題

`deferrable=True`的目的是在等待的時候（例如Azure VM開關機需要時間），釋放worker slot來節約資源。初版的實作是在`AzureVirtualMachineStateTrigger.run`中自幹async，如下所示:

```python
def _get_power_state(self) -> str:
    from airflow.providers.microsoft.azure.hooks.compute import AzureComputeHook
    if not hasattr(self, "_hook"):
        self._hook = AzureComputeHook(azure_conn_id=self.azure_conn_id)
    return self._hook.get_power_state(self.resource_group_name, self.vm_name)

async def run(self) -> AsyncIterator[TriggerEvent]:
    try:
      while True:
          power_state = await asyncio.to_thread(self._get_power_state)
          if power_state == self.target_state:
              message = f"VM {self.vm_name} reached state '{self.target_state}'."
              yield TriggerEvent({"status": "success", "message": message})
              return
          await asyncio.sleep(self.poke_interval)
```

這邊的問題分兩個層面：

第一個問題：`_get_power_state` 裡使用了 lazy import，第一次執行時會 import `AzureComputeHook`，進而 import `ComputeManagementClient`。`ComputeManagementClient` 是一個非常重的 import，直接在 event loop 裡執行時會卡住整個 triggerer。

第二個問題：即使用 `asyncio.to_thread` 把 sync 呼叫移到 thread pool，Azure SDK 的 sync HTTP client 在處理 response 時（JSON parsing、OAuth token refresh）是 CPU-bound 的 Python bytecode，會持有 GIL（Global Interpreter Lock），讓 event loop thread 搶不到 CPU，導致 Triggerer IPC frame 漏接、觸發crash。

### 解法

改用 Azure 官方提供的 native async client：

```python
from azure.identity.aio import ClientSecretCredential as AsyncClientSecretCredential
from azure.mgmt.compute.aio import ComputeManagementClient as AsyncComputeManagementClient
```

`azure.mgmt.compute.aio` 底層使用 aiohttp，在等待網路 I/O 時會真正釋放 GIL，讓 event loop 可以繼續處理其他 trigger，不再有 blocking 的問題。
