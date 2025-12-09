---
title: "AssetAndTimeScheduler實作過程思考與紀錄（Part 2）"
date: "2025-11-21T00:00:00-00:00"
draft: true
tags: ["Airflow"]
categories: ["Open Source"]
---

如果 Asset 永遠不準備好，會發生什麼事？

## 實作

### AssetAndTimeSchedule

基本上與 `AssetOrTimeSchedule` 相同，不同點有二：
1. `AssetAndTimeSchedule` 不需被Asset Triggered，因此繼承 `Timetable`。
2. 生產 run_id 時只使用 timetable 的 generate_run_id。

```python
class AssetOrTimeSchedule(AssetTriggeredTimetable):
    ...
    def generate_run_id(self, *, run_type: DagRunType, **kwargs: typing.Any) -> str:
        if run_type != DagRunType.ASSET_TRIGGERED:
            return self.timetable.generate_run_id(run_type=run_type, **kwargs)
        return super().generate_run_id(run_type=run_type, **kwargs)

class class AssetAndTimeSchedule(Timetable):
    ...
    def generate_run_id(self, *, run_type: DagRunType, **kwargs: typing.Any) -> str:
        return self.timetable.generate_run_id(run_type=run_type, **kwargs)
```

### Scheduler

首先對 `DagModel.dags_needing_dagruns` 稍做修改，讓 `AssetAndTimeSchedule` 物件能不被加入到 `triggered_date_by_dag`，因為我們不依賴 Asset Triggered。

```python
for ser_dag in ser_dags:
    dag_id = ser_dag.dag_id
    statuses = dag_statuses[dag_id]

    # Add the code block below to prevent `AssetAndTimeSchedule` from being added to `triggered_date_by_dag`
    #####################################
    timetable = ser_dag.dag.timetable

    if not isinstance(timetable, AssetTriggeredTimetable):
        del adrq_by_dag[dag_id]
        continue
    #####################################

    if not dag_ready(dag_id, cond=ser_dag.dag.timetable.asset_condition, statuses=statuses):
        del adrq_by_dag[dag_id]
        del dag_statuses[dag_id]
```

新增檢查 assets 是否 ready 的邏輯在 `SchedulerJobRunner._start_queued_dagruns` 裡。

例如依賴兩個 assets，則 required_count 為 2，如果只有一個 asset ready，則 ready_count 為 1，此時會被我們的邏輯擋住而進入到 continue。

```python
# For AssetAndTimeSchedule, defer starting until all required assets are queued.
if isinstance(dag.timetable, AssetAndTimeSchedule):
    # Count required assets for this DAG's schedule
    required_count = session.scalar(
        select(func.count()).select_from(DagScheduleAssetReference).where(
            DagScheduleAssetReference.dag_id == dag_id
        )
    ) or 0

    if required_count > 0:
        ready_count = session.scalar(
            select(func.count())
            .select_from(DagScheduleAssetReference)
            .join(
                AssetDagRunQueue,
                and_(
                    DagScheduleAssetReference.asset_id == AssetDagRunQueue.asset_id,
                    DagScheduleAssetReference.dag_id == AssetDagRunQueue.target_dag_id,
                ),
            )
            .where(DagScheduleAssetReference.dag_id == dag_id)
        ) or 0

        if ready_count < required_count:
            # Not all assets are present; skip starting this run for now.
            self.log.debug(
                "Deferring DagRun until assets ready; dag_id=%s run_id=%s (ready=%s required=%s)",
                dag_id,
                run_id,
                ready_count,
                required_count,
            )
            # Do not increment active run counts; we didn't start it.
            continue

        # Consume queued asset events for this DAG so the next run gates on new events.
        session.execute(
            delete(AssetDagRunQueue).where(AssetDagRunQueue.target_dag_id == dag_id)
        )
```

當條件滿足後，會進入到 `_update_state(dag, dag_run)` 將 Dag 更新成 RUNNING 狀態，隨後執行。