---
title: "AssetAndTimeScheduler實作過程思考與紀錄（Part 3）"
date: "2026-05-28T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["Airflow"]
categories: ["Open Source"]

math:
  enable: true
---

沒料到這系列可以出到 Part 3，這是我在 Airflow 挑戰的第一個無中生有的 feature，沒想到比想像中複雜一些。

原本的實作是先創造 placeholder QUEUED run 然後在 scheduler 裡無限 loop 等 asset 進才跑。用 dagrun_timeout -> FAILED 這樣hack的方式處理超時，破壞 DagRun 狀態機。

新版實作是 時間 + asset 兩個條件都到位後才建立 DagRun，靠 ADRQ composite PK 與 `next_dagrun_create_after` 不前進雙重防堆積，無 hack、不違反狀態機。

簡單來說就是 Gating 從「DagRun 建立之後」搬到「DagRun 建立之前」。

## 新舊流程 mermaid 圖比較

+ 舊版流程
```mermaid
flowchart TD
    A([Schedule time arrives]) --> B[Create DagRun in QUEUED<br/>= placeholder waiting for asset]
    B --> C{Next scheduler loop<br/>_start_queued_dagruns}
    C --> D{Asset condition met?}
    D -->|Yes| E[Consume ADRQ<br/>Promote QUEUED to RUNNING]
    D -->|No| F{dagrun_timeout<br/>exceeded?}
    F -->|No| G[Stay in QUEUED]
    G --> C
    F -->|Yes| H[Set DagRun = FAILED<br/>BUT TI stays in 'none']

    style B fill:#fdd,stroke:#c33
    style H fill:#fdd,stroke:#c33
    style G fill:#ffd,stroke:#a80
```

+ 新版流程
```mermaid
flowchart TD
    A([Schedule time arrives]) --> B{dags_needing_dagruns<br/>ADRQ has row?}
    B -->|No| Z[Skip — no DagRun created<br/>next_dagrun_create_after stays put]
    B -->|Yes| C{Evaluate asset_condition}
    C -->|Not met| Z
    C -->|Met| D[_create_dag_runs_asset_gated:<br/>Lock ADRQ + re-evaluate HA guard]
    D --> E[Create DagRun in QUEUED<br/>+ delete consumed ADRQ rows<br/>+ extend consumed_asset_events<br/>+ update exceeds_max_non_backfill]
    E --> F{Normal _start_queued_dagruns<br/>max_active_runs slot free?}
    F -->|Yes| G[Promote QUEUED to RUNNING]
    F -->|No| H[Stay in QUEUED<br/>= original semantics:<br/>waiting for executor capacity]

    style Z fill:#dfd,stroke:#3a3
    style E fill:#dfd,stroke:#3a3
    style H fill:#dfd,stroke:#3a3
```

## 新流程的細節

+ `_create_dag_runs_asset_gated`
    + 拿到候選 DagModel → 取 serialized Dag → lock 對應 queue rows → re-check asset condition → 建 DagRun → 消費 queue rows

新流程的好處是用了較多原始airflow的機制，因此不需要一直 hack 去處理各種狀況。目前即使 Producer 跟 Consumer 不同步也能 handle。

+ Case A: Producer 10min 跑一次 / Consumer 20min 跑一次
| Time | Producer | Consumer | Note |
| --- | --- | --- | --- |
| T=00 | v | v | |
| T=10 | v | | 寫入 ADRQ |
| T=20 | v | v | T=20 不會被寫入，因為 ADRQ 最多存一筆。建立 run，消費 T=10 寫的 ADRQ row，但 T=20 producer 的 AssetEvent 仍存在 asset_event table。 |
| T=30 | v | | |
| T=40 | v | v | |
| T=50 | v | | |
| T=00 | v | v | |

+ Case B: Producer 20min 跑一次 / Consumer 10min 跑一次
| Time | Producer | Consumer | Note |
| --- | --- | --- | --- |
| T=00 | v | v | 寫入 ADRQ + 建立 run |
| T=10 | | v | ADRQ 為空，因此不建立 run |
| T=20 | v | v | |
| T=30 | | v | |
| T=40 | v | v | |
| T=50 | | v | |
