# Phase 1：架构全景

## 学习目标

- 理解 Kubernetes 整体架构（控制面 + 节点 + 附加组件）
- 建立源码目录与运行时组件的映射关系
- 了解一次 `kubectl apply` 请求在集群中的完整路径

## 已有笔记

| 笔记 | 内容 |
|------|------|
| [k8s-basics-features-and-components.md](./k8s-basics-features-and-components.md) | K8s 八大核心功能、七大组件、九大资源对象 |
| [k8s-basics-architecture.md](../diagrams/k8s-basics-architecture.md) | 对应原理与架构图（13 张 Mermaid） |
| [k8s-basics-overview.png](../diagrams/k8s-basics-overview.png) | 核心功能与组件总览说明图 |
| [k8s-basics-interview-qa.md](./k8s-basics-interview-qa.md) | 基础面试问答单元（6 题 + 追问） |

## 源码顶层目录

| 目录 | 作用 |
|------|------|
| `cmd/` | 各组件二进制入口（kube-apiserver、kubelet 等） |
| `pkg/` | 核心业务逻辑实现 |
| `staging/src/k8s.io/` | 可独立发布的 Go 模块（apimachinery、client-go 等） |
| `api/` | OpenAPI 规范与 API 规则 |
| `plugin/` | 准入插件、认证插件等 |
| `test/` | 集成测试、e2e 测试 |

## 核心组件入口

| 组件 | 源码入口 |
|------|----------|
| kube-apiserver | `cmd/kube-apiserver/apiserver.go` |
| kube-controller-manager | `cmd/kube-controller-manager/controller-manager.go` |
| kube-scheduler | `cmd/kube-scheduler/scheduler.go` |
| kubelet | `cmd/kubelet/kubelet.go` |
| kube-proxy | `cmd/kube-proxy/proxy.go` |
| kubectl | `cmd/kubectl/kubectl.go` |
| kubeadm | `cmd/kubeadm/kubeadm.go` |

## 请求链路（kubectl apply Pod）

```
kubectl → kube-apiserver → etcd
                ↓
         controller-manager（Reconcile）
                ↓
         kube-scheduler（Bind）
                ↓
         kubelet（创建容器）
```

## 建议阅读顺序

1. 阅读 [k8s-basics-features-and-components.md](./k8s-basics-features-and-components.md)，建立功能与组件的全局认知
2. 用 [k8s-basics-interview-qa.md](./k8s-basics-interview-qa.md) 自测面试高频题
3. 通读本目录，画出架构图
4. 浏览 `cmd/` 下各组件的 `main()` 入口
5. 阅读 `staging/src/k8s.io/apimachinery/pkg/runtime/` 了解对象模型
