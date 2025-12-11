---
title: "用Breeze測試Airflow K8s環境"
date: "2025-12-01T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["Airflow"]
categories: ["Open Source"]
---

記錄我使用 `breeze k8s` 測試時的步驟以及遇到的坑。

## Build a local k8s cluster

近期我會用以下指令來建立以及啟動一個Local K8s Cluster來做測試，我的筆電是 Macbook Air M1 w/ 16GB ram。

```console
breeze k8s create-cluster
breeze k8s configure-cluster
breeze prod-image build --python 3.10
breeze k8s build-k8s-image
breeze k8s upload-k8s-image
breeze k8s deploy-airflow
```

當我想要看 pod 的 log 來 debug 時，用 `kubectl` 列出 pods 會遇到以下 error：

```console
aaron.chen@Aarons-MacBook-Air-3 airflow % kubectl get po -n airflow
E1123 10:59:15.472245   16746 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
E1123 10:59:15.473418   16746 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
```

問題在於Kubectl沒有指到我們剛剛建立的K8s cluster，此時直接手動設置context。

```console
aaron.chen@Aarons-MacBook-Air-3 airflow % kind get clusters                                           
airflow-python-3.10-v1.30.13
aaron.chen@Aarons-MacBook-Air-3 airflow % kind export kubeconfig --name airflow-python-3.10-v1.30.13
Set kubectl context to "kind-airflow-python-3.10-v1.30.13"
aaron.chen@Aarons-MacBook-Air-3 airflow % kubectl config get-contexts
CURRENT   NAME                                CLUSTER                             AUTHINFO                            NAMESPACE
          docker-desktop                      docker-desktop                      docker-desktop                      
*         kind-airflow-python-3.10-v1.30.13   kind-airflow-python-3.10-v1.30.13   kind-airflow-python-3.10-v1.30.13   
aaron.chen@Aarons-MacBook-Air-3 airflow % kubectl get ns  
NAME                 STATUS   AGE
airflow              Active   15m
default              Active   45h
kube-node-lease      Active   45h
kube-public          Active   45h
kube-system          Active   45h
local-path-storage   Active   45h
test-namespace       Active   15m
```

## Update dag to k8s cluster

就我所知，目前要在 update k8s cluster airflow dag 有點麻煩，必須將 dag 放到 airflow 根目錄的 `/dags` 資料夾下並 rebuild image 並重啟 dag-processor 和 scheduler，整個過程在我的 local device 大約花費六分鐘。

```console
breeze k8s build-k8s-image
breeze k8s upload-k8s-image
helm uninstall airflow -n airflow
breeze k8s deploy-airflow
```