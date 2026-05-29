# CRI（Container Runtime Interface）

## 源码路径

- CRI API：`staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/`
- Kubelet 调用层：`pkg/kubelet/kuberuntime/`

## 学习目标

- 理解 CRI gRPC 接口（RuntimeService、ImageService）
- 掌握 Kubelet 如何通过 CRI 创建/停止容器
- 了解 containerd/CRI-O 与 Kubelet 的交互

## 关键接口

- `RunPodSandbox` — 创建 Pod 沙箱
- `CreateContainer` — 创建容器
- `StartContainer` / `StopContainer` — 启停容器
- `PullImage` — 拉取镜像

## 建议阅读路径

1. `staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto` — 接口定义
2. `pkg/kubelet/kuberuntime/kuberuntime_sandbox.go` — Sandbox 管理
3. `pkg/kubelet/kuberuntime/kuberuntime_container.go` — 容器管理

## 笔记

（在此记录）
