# kube-controller-manager

## 源码路径

- 入口：`cmd/kube-controller-manager/controller-manager.go`
- 核心：`pkg/controller/`

## 学习目标

- 理解 Controller Manager 如何启动多个 Controller
- 掌握 Deployment Controller 的 Reconcile 逻辑
- 理解 Informer + WorkQueue 模式在控制器中的实践

## 主要控制器

| 控制器 | 路径 | 职责 |
|--------|------|------|
| Deployment | `pkg/controller/deployment/` | 管理 Deployment 与 ReplicaSet |
| ReplicaSet | `pkg/controller/replicaset/` | 维持 Pod 副本数 |
| Node | `pkg/controller/nodelifecycle/` | 节点健康与驱逐 |
| Service | `pkg/controller/service/` | Service 与 Endpoints |
| EndpointSlice | `pkg/controller/endpointslice/` | EndpointSlice 管理 |
| Job | `pkg/controller/job/` | Job 完成检测 |
| Namespace | `pkg/controller/namespace/` | Namespace 生命周期 |

## 建议阅读路径

1. `cmd/kube-controller-manager/app/controllermanager.go` — 控制器注册
2. `pkg/controller/deployment/deployment_controller.go` — 经典控制器
3. `pkg/controller/replicaset/replica_set.go` — Pod 副本管理

## 笔记

（在此记录）
