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

分五大階段實作，部分階段會有相依性。五大階段的順序以及目的如下：

```
Phase 1 ──→ Phase 2 ──→ Phase 3a ──→ Phase 3e
                    ├──→ Phase 3b ──→ Phase 3e
                    └──→ Phase 3d
             Phase 3c（獨立，Python 腳本）
             Phase 3e 等 Phase 3a, 3b 完成後執行
             Phase 3f 等所有消費端就緒
             Phase 4 本地驗證通過後再接 CI
             Phase 5 最後
```

| 步驟 | 內容 | 依賴 | 理由 |
|------|------|------|------|
| **Phase 1** | 生成器（`run_generate_constraints.py`） | 無 | 基礎，產出檔案 |
| **Phase 2** | `common.sh` — `get_build_constraints_location()` | Phase 1 | 所有 shell 消費端的共用 helper |
| **Phase 3a** | `install_airflow_when_building_images.sh` | Phase 2 | 呼叫 common.sh helper |
| **Phase 3b** | `install_from_docker_context_files.sh` | Phase 2 | 呼叫 common.sh helper |
| **Phase 3c** | `install_airflow_and_providers.py` | Phase 1 | 獨立 Python 腳本，不依賴 common.sh |
| **Phase 3d** | `entrypoint_ci.sh` + `install_development_dependencies.py` | Phase 1 | 不走 common.sh，直接傳 build constraints 檔案路徑 |
| **Phase 3e** | Dockerfile inlined scripts 重新生成 | Phase 3a, 3b | `prek update-inlined-dockerfile-scripts`，需等 shell scripts 改完 |
| **Phase 3f** | Breeze CLI plumbing | Phase 2-3d | 把 env var 傳進容器，串接所有消費端 |
| **Phase 4** | CI artifacts 發布 | Phase 1 | 純 infra，本地驗證通過後再接 CI |
| **Phase 5** | 文件更新 | 全部 | 等一切穩定 |

# 實作細節

## Phase 1 - `run_generate_constraints.py` (Entry Point)

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

## Phase 2 - `common.sh`

### `get_build_constraints_location`

+ ~~第一個 if~~
1. ~~Install mode 為 "."，根據 `install_airflow_when_building_images` 中的邏輯，會跑`uv sync`。不是我們的目標因此跳過~~
2. ~~最後加上 `rm` 清理 `build-constraints.txt`，以免後續 get_build_constraints_install_flags 時會錯給 flag~~

這邊在Phase 3實作中反覆思考一陣後被刪除了，因為只有Source Install模式會不會用到build-constraints，因此只在這個模式的function `install_airflow_when_building_images` 將其移除。

+ 第二個if
1. 如果用戶手動設定constrints file的變數，則使用該file
2. 會進到if的例子(從網路下載):
    1. AIRFLOW_BUILD_CONSTRAINTS_LOCATION="https://raw.githubusercontent.com/user/repo/test-branch/build-constraints-3.12.txt"
3. 會進到else的例子(Docker or Breeze):
    1. AIRFLOW_BUILD_CONSTRAINTS_LOCATION="/docker-context-files/my-special-constraints.txt"
    2. AIRFLOW_BUILD_CONSTRAINTS_LOCATION="/files/constraints-3.12/build-constraints-3.12.txt"

+ 第三個if跟第四個if
1. 第三個if是找對應的branch，因為build constraints會放在跟原來constraints一樣的branch裡 (e.g. `constraints-3-0`)
2. 第四個if是萬一沒有 build_constraints_url，則創建一個空的檔案 or 如果檔案存在就清空他
3. 第四個if的意思「如果下載失敗，則走then的邏輯」
    1. `if ! curl -sSf -o "${HOME}/build-constraints.txt" "${build_constraints_url}"; then`

### `get_build_constraints_install_flags`

Phase 3會呼叫這個function來判斷是否需要添加flag `--build-constraints`

## Phase 3

### `scripts/docker/install_airflow_when_building_images.sh`

標準安裝(Standard Install)時使用這個`.sh`

1. `install_from_external_spec`
    - 官方build prod/release docker image時，會從PyPI取得對應的版本並build
    - 用戶想基於官方版本來build自定義的Airflow image時用
    - external package install ("apache-airflow") 走 pip install / uv pip install，需要使用 build constraints
2. `install_airflow_when_building_images`
    - 本地 or CI執行docker build時會被呼叫，因為Dockerfile有底下這段code
    - `COPY --from=scripts install_airflow_when_building_images.sh /scripts/docker/`
    - `RUN bash /scripts/docker/install_airflow_when_building_images.sh`
    - source install 不呼叫 common::get_build_constraints_location，是因為 uv sync 不支援也不需要 --build-constraints；它依 uv.lock 安裝

### `scripts/docker/install_from_docker_context_files.sh`

離線/自訂套件(Context Install)安裝時使用。

當 CI 或用戶透過 DOCKER_CONTEXT_FILES 提供已經build好的distributions時，Dockerfile就會呼叫這個script，把這些wheel/sdist安裝進image。

Airflow CI會用這條路徑來驗證 source build 出來的 distributions。

TL;DR - 發PR時，CI驗證會用

### `scripts/in_container/install_airflow_and_providers.py`

測試特定Airflow版本時會呼叫這個檔案，例如 USE_AIRFLOW_VERSION=3.1.0

### `scripts/docker/entrypoint_ci.sh`

同上，只有在測試特定Airflow版本時會加build constraints

只在func `determine_airflow_to_use` 中加入 --build-constraints 的最大理由是他不是 uv sync，它是 uv run，所以必須加

### `scripts/in_container/install_development_dependencies.py`

建立 Breeze CI Image 時會用到

重點是他用到 uv pip install，因此需要 --build-constraints


### `Dockerfile` & `Dockerfile.ci`

這兩個都是手動執行 `prek update-inlined-dockerfile-scripts --all-files` 後，自動把script貼過去

### Breeze相關改動

基本上就是把各個入口補齊，加上 flag `--build-constraints`

## Phase 4 - CI

將CI的generate constraints的階段加入我們前面實作的部分

## Phase 5 - 文件

將文件補齊

# 驗證

