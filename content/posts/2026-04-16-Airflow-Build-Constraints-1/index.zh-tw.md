---
title: "Airflow --build-constraints 實作思考"
date: "2026-03-29T00:00:00-00:00"
draft: true
description: "記錄如何幫Airflow複雜的建置流程中增加 --build-constraints 來增加build的穩定性"
featuredImage: ""

tags: ["Airflow"]
categories: ["Open Source"]

math:
  enable: true
---

# 概觀

issue: [#54394](https://github.com/apache/airflow/issues/54394)

主要問題是Airflow的依賴套件雖然都有控制住版本號，但我們難以保證其套件的上游套件會不會做什麼更動導致跟現行的套件衝突。Airflow目前相依套件超過六百多個，更加深了這個問題的發生可能性。

過往一些PR有做一些優化例如改用`uv`並善用`uv`的能力例如`uv sync`、`uv.lock`、以及產生的`pyproject.toml`來控制住各個相依套件的版本號。但有些套件是沒有`.whl`的（例如`setuptools`），Python會將他們的Source Distribution（簡稱sdist）下載到一個沙盒環境中並安裝，但預設是下載最新的版本。

之前`pip`並不支援對sdist的版本控制，但`uv`支援，使用flag `--build-constraints`可辦到。因此這個issue的目標是將`--build-constraints`整合進現行Airflow build的流程中。

# 挑戰

1. 相依套件太多（Algorithm）
    - 上面提到Airflow有超過六百個相依套件，要如何整理出sdist相關的套件並產生一份給`--build-constraints`的`build-constraints-*.txt`是個挑戰，同時還要考慮掉CI的效能
2. 安裝路徑過多（Coverage）
    - Airflow有多種安裝方式（只裝core、只裝ctl、用docekr裝...等等），要全面覆蓋是個巨大的工程
3. 缺乏文件（Documentation）
    - 
4. 驗證

# 實作規劃

# 實作細節

## `run_generate_constraints.py` (Entry Point)

### 收集 (Collection)

####  Step 1: _collect_workspace_build_reqs

目的
+ 找出repo內所有的`pyproject.toml`，然後只看`build-system`的部分
+ 會跳過不相關的目錄例如`.venv`, `.git`, `node_modules`
+ 只找build time constraints，也就是上述提到的sdist
+ 最後會產出一個dict，key是套件名稱，value是對套件的要求，例如 `{setuptools: {"setuptools>=70.0", "setuptools>=68.2"}}`
    + 同一個套件會有不同要求是因為可能不同的`pyproject.toml`對套件的要求不同，這邊先全部搜集起來，之後再處理

#### Step 2: _collect_upstream_build_reqs

目的
+ 找出repo內所有的`uv.lock`，然後只看沒有`.whl`的套件（使用`_parse_uv_lock`檢查）
+ 使用Multi-thread pool來加速下載sdist的流程
+ 下載完的sdist資訊會進Github Action cache，之後main branch建好cache後，之後所有的PR就不需要對之前下載過的sdist再下載一次

### 合併 (Merging)

+ Step 3: 把Step 1跟Step 2的檔案合併成類似這樣 `{setuptools: {"setuptools>=70.0", "setuptools>=68.2"}}`

### 解析 (Resolution)

#### Step 4: 

目的
+ 定義版本
    + 如果只定義了下界例如 `setuptools>=70.0`，則會安裝最新版本（實作是用 "--resolution", "highest"）
    + 如果上下界都定義了，也是安裝最新版本
    + 如果版本衝突了，則不將該套件寫入build-constraints-*.txt，並發出warning
+ Exact Pin: 如果套件版本已經穩定則不參與解析
    + 如果版本號已經寫死，例如 `{setuptools: {"setuptools==70.0"}}`，則不會把它寫入build-constraints-*.txt
    + 因為寫死的話，在沙盒隔離安裝中，本來就會安裝該版本
    + 原始issue關心的是沒有被寫死的sdist，會自動安裝最新版本而壞掉
        + 這邊我一開始覺得矛盾，因為我們在定義版本那邊說會安裝最新版本
        + 但在Airflow的`pyproject.toml`中有定義`exclude-newer = "4 days"`，代表只會抓出超過四天的套件
        + 我們的工具又是掃`pyproject.toml`，因此大大降低抓到有重大bug的新版本的機率
        + 原始版本是直接就download最新版了，很有可能因此壞掉
+ 確保環境一致
    + 解析時考慮特定 Python 版本

#### Step 5: 

+ 把`BUILD_CONSTRAINTS_PREFIX`跟解析過後的build constraints寫進build-constraints-*.txt
