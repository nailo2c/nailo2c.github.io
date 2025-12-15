---
title: "AssetAndTimeScheduler實作過程思考與紀錄（Part 1）"
date: "2025-11-16T00:00:00-00:00"
draft: false
description: "此篇文章記錄了個人思考與開發 Issue #58506 所要求 feature 的過程"
featuredImage: "airflow_feat_implementation.png"

tags: ["Airflow"]
categories: ["Open Source"]

math:
  enable: true
---

## 背景介紹

此篇文章記錄了個人思考與開發 [issue #58506](https://github.com/apache/airflow/issues/58056) 所要求 feature 的過程。

## 研究

我在 Airflow Core 方面沒有經驗，為了實作出這個功能，我參考了既有的 AssetOrTimeScheduler，發現此功能是依靠 `SchedulerJobRunner` 的能力而來，因此了解 `SchedulerJobRunner` 的運作機制變成了我的首要課題。

### SchedulerJobRunner 與 AssetOrTimeSchedule

我的理解是 `SchedulerJobRunner` 啟動後，會定期掃描 Queue 與 Database 來做各種事。執行 `airflow scheduler` 後，`_run_scheduler_job` function 會執行 `SchedulerJobRunner._execute`，裡頭會執行 `_run_scheduler_loop` 並呼叫 `EventScheduler` 來運作定期掃描。

而 `AssetOrTimeSchedule` 被傳入 Dag 的 schedule 後，timetable 與 assets 的資訊會被 serialize 進 metadata database，應該是會存在 `dag`（欄位 `timetable_summary` 與 `asset_expression`）、`dag_schedule_asset_reference` 與 `dag_schedule_asset_uri_reference` 裡，後兩張 tables 應該是為了支援 asset triggered 用。

以下為官網範例
```python
from airflow.timetables.assets import AssetOrTimeSchedule
from airflow.timetables.trigger import CronTriggerTimetable

@dag(
    schedule=AssetOrTimeSchedule(
        timetable=CronTriggerTimetable("0 1 * * 3", timezone="UTC"), assets=(dag1_asset & dag2_asset)
    )
    # Additional arguments here, replace this comment with actual arguments
)
def example_dag():
    # Dag tasks go here
    pass
```

當上游的 Dag 產生 asset 時，`asset_manager.register_asset_change` 會尋找所有該 asset 的下游，並寫入 `asset_dag_run_queue` 等待 `SchedulerJobRunner._create_dagruns_for_dags` 來檢查。最終 `SchedulerJobRunner._create_dag_runs_asset_triggered` 建立必須執行的 DagRun 後，再將這些 Dag 從 `asset_dag_run_queue` 中刪除。

至於 Cron 的部分，`next_dagrun_info` function 會更新 metadata table 讓 scheduler 知道何時該 create DagRun，最後由 `SchedulerJobRunner._create_dagruns_for_dags` query 出 non_asset_dags 跟 asset_triggered_dags。如果上游 asset 還沒準備好，此時 Dag 就會在 non_asset_dags 裡，最終被 `SchedulerJobRunner._create_dag_runs` 執行。

## AssetAndTimeScheduler

### 設計

根據 issue 內容，先定義出最小可用版的 Acceptance Criteria，OP所說的 backfill 功能先不予考慮。

+ AC1: Impletement `AssetAndTimeSchedule` Class and API, interface is similiar with `AssetOrTimeSchedule`
+ AC2: 新增單元測試
+ AC3: 新增文件
+ AC4: UI？

#### SchedulerJobRunner._create_dagruns_for_dags

它會將 dags 區分為 non_asset_dags 跟 asset_triggered_dags，並把他們放進 table 中，狀態為 DagRunState.QUEUED。

non_asset_dags 由 `SchedulerJobRunner._create_dag_runs` 處理，依賴時間排程相關的 metadata，關心「時間到了沒？`active_runs`還夠嗎？」如果是，就建立 `SCHEDULED` run。

asset_triggered_dags 由 `SchedulerJobRunner._create_dag_runs_asset_triggered` 處理，依賴事件相關的 metadata，關心「哪些 asset_event 應該觸發新的run？這次 run 要吃掉哪些 `AssetEvent`？」然後建立 `ASSET_TRIGGERED` run。

決定他們會不會變成 RUNNING 的是 `SchedulerJobRunner._start_queued_dagruns`。

#### SchedulerJobRunner._start_queued_dagruns

這個 function 的作用是找出所有狀態為 queued 的 dags，並決定是否將它們轉為 running。

### 實作

下一篇將會講述實作細節。

## Reference

[Airflow Metadata Database Schema](https://airflow.apache.org/docs/apache-airflow/stable/database-erd-ref.html)

## tmp

generate_run_id
1. 當 DagRunType 不是 ASSET_TRIGGERED 時
    1.1. 手動：manual__2023-10-27T10:00:00+00:00
    1.2. 排程：scheduled__2023-10-27T01:00:00+00:00
    1.3. 回填：backfill_job__2023-10-26T00:00:00+00:00
2. 當 DagRunType 是 ASSET_TRIGGERED 時：
    2.1. asset_triggered__2023-10-27T11:30:00+00:00
