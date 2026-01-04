---
title: "如何測試Airflow Provider cncf.kubernetes"
date: "2026-01-03T00:00:00-00:00"
draft: true
tags: ["Airflow"]
categories: ["Open Source"]
---

Airflow定期會做release，每當release前，PMC們會請求各個PR提交人的協助，驗證RC（Release Candidate）版本。

他們會有一個文件說明如何測試，但我在該文件中沒看到關於K8s provider該如何測，因此在這邊記錄我的做法。

整體來說，建立local k8s cluster與之前大致相同：
```console
breeze k8s create-cluster
breeze k8s configure-cluster
breeze prod-image build --python 3.10
breeze k8s build-k8s-image
breeze k8s upload-k8s-image
breeze k8s deploy-airflow
```

唯一不同的地方在於 `breeze prod-image build --python 3.10`，這邊我的做法是將wheel檔下載到根目錄的 `docker-context-files` 資料夾裡，再執行以下指令去build：
```console
breeze prod-image build --python 3.10 \
    --install-distributions-from-context \
    --install-airflow-version 3.1.2 \
    --use-constraints-for-context-distributions \
    --airflow-constraints-mode constraints-no-providers \
    --airflow-constraints-location https://raw.githubusercontent.com/apache/airflow/constraints-3.1.2-fix/constraints-no-providers-3.10.txt
```

這邊要注意 airflow 的版本是我自己選的，我選 `3.1.2` 是因為當下這個時間點，constraints只有3.1.2。而且使用 `--airflow-constraints-mode` 跟 `--airflow-constraints-location` 這兩個 flag 也是因為 breeze prod-image build 的某些 flag 有 bug，之後我再看有沒有機會去修正它。