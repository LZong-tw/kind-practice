# Kubernetes 監控環境架設指南

## 目錄

1. [架構概述](#架構概述)
2. [步驟 1-2: 建立 Kind 叢集與劃分節點角色](#步驟-1-2-建立-kind-叢集與劃分節點角色)
3. [步驟 3: 安裝 MetalLB](#步驟-3-安裝-metallb)
4. [步驟 4: 安裝監控元件](#步驟-4-安裝監控元件)
5. [步驟 5: 安裝與配置 Grafana](#步驟-5-安裝與配置-grafana)
6. [步驟 6: 部署測試應用程式和 HPA](#步驟-6-部署測試應用程式和-hpa)
7. [監控面板說明](#監控面板說明)
8. [CPU Throttling 監控說明](#cpu-throttling-監控說明)

## 架構概述

本文檔說明如何使用 Kind 建立一個包含 1 個 Control Plane 和 3 個 Worker 節點的 Kubernetes 叢集，並基於以下需求進行配置：

- 將 3 個 Worker 節點中的 1 個標記為 Infra 節點，用於執行基礎設施服務
- 將其餘 2 個 Worker 節點標記為 Application 節點，用於執行應用程式
- 在 Infra 節點上安裝 MetalLB (L2模式) 的 Speaker 組件
- 在 Infra 節點上安裝 Prometheus 和 kube-state-metrics
- 在所有節點上安裝 node-exporter
- 在叢集外部使用 Docker 執行 Grafana，並連接到 Prometheus
- 部署測試應用程式到 Application 節點，並設定 HPA

## 步驟 1-2: 建立 Kind 叢集與劃分節點角色

### 1. 準備 Kind 配置檔

首先，我們需要一個 Kind 的設定檔來定義叢集結構。創建 `kind-config.yaml` 檔案：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: monitoring-demo
nodes:
- role: control-plane
  # 為了讓 Prometheus/Grafana 可以抓到 etcd metrics，需要暴露 etcd 的監聽埠
  # 同時，kube-apiserver 也需要啟用 --bind-address=0.0.0.0
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
    etcd:
      local:
        extraArgs:
          "listen-metrics-urls": "http://0.0.0.0:2381" # 監聽所有接口的指標請求
- role: worker
  # 這個 worker 將被標記為 infra node
- role: worker
  # 這個 worker 將被標記為 app node
- role: worker
  # 這個 worker 將被標記為 app node
```

### 2. 建立叢集並標記節點角色

使用以下命令建立叢集：

```bash
$ kind create cluster --config kind-config.yaml
```

識別節點名稱：

```bash
$ kubectl get nodes
```

標記節點角色：

```bash
$ kubectl label node monitoring-demo-worker node-role.kubernetes.io/infra=true
$ kubectl label node monitoring-demo-worker2 node-role.kubernetes.io/app=true
$ kubectl label node monitoring-demo-worker3 node-role.kubernetes.io/app=true
```

驗證標記：

```bash
$ kubectl get nodes --show-labels
```

## 步驟 3: 安裝 MetalLB

### 1. 安裝 MetalLB

根據 MetalLB 官方文件安裝：

```bash
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

### 2. 配置 MetalLB (L2 模式)

首先，確定 Kind 叢集的 Docker 網路範圍：

```bash
$ docker ps | grep -i kind
```

使用以下命令確定各 container 的 IP：

```bash
$ docker inspect <container-id> | grep IPAddress
```

選擇一個未被 Kind 節點使用的 IP 範圍，創建 `metallb-l2config.yaml` 檔案：

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250 # 請根據 docker inspect 結果調整此範圍
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

套用設定：

```bash
$ kubectl apply -f metallb-l2config.yaml
```

### 3. 限制 Speaker 只在 Infra 節點運行

編輯 MetalLB 的 speaker DaemonSet：

```bash
$ kubectl edit daemonset speaker -n metallb-system
```

在 spec.template.spec 下增加 nodeSelector：

```yaml
spec:
  template:
    spec:
      # ... 其他設定 ...
      nodeSelector:
        node-role.kubernetes.io/infra: "true" # << 新增這一行
      tolerations:
      # MetalLB speaker 可能需要容忍 control-plane 的 taint，如果有的話
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      # 也需要容忍我們自己加的 infra taint
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        effect: "NoSchedule"
      # ... 其他設定 ...
```

確認 Speaker 只在標記為 infra=true 的節點上運行：

```bash
$ kubectl get pods -n metallb-system -o wide
```

## 步驟 4: 安裝監控元件

### 1.	安裝 Helm:
```bash
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
