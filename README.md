# Kubernetes 源码学习笔记

> 基于本地源码路径：`../kubernetes-master/`（kubernetes/kubernetes 官方仓库）

## 学习路线总览

```
Phase 0  前置知识          → 00-prerequisites/
Phase 1  架构全景          → 01-architecture-overview/
Phase 2  基础库            → 02-foundation/
Phase 3  控制面组件        → 03-control-plane/
Phase 4  节点组件          → 04-node/
Phase 5  CLI 与集群引导    → 05-cli-bootstrap/
Phase 6  核心机制          → 06-core-mechanisms/
Phase 7  扩展与进阶        → 07-extension/
```

## 已收录笔记

| 笔记 | 路径 |
|------|------|
| K8s 基础功能与组件 | [01-architecture-overview/k8s-basics-features-and-components.md](./01-architecture-overview/k8s-basics-features-and-components.md) |
| K8s 原理与架构图（13 张） | [diagrams/k8s-basics-architecture.md](./diagrams/k8s-basics-architecture.md) |
| K8s 核心功能与组件总览图 | [diagrams/k8s-basics-overview.png](./diagrams/k8s-basics-overview.png) |
| K8s 基础面试问答 | [01-architecture-overview/k8s-basics-interview-qa.md](./01-architecture-overview/k8s-basics-interview-qa.md) |
| K8s 集群搭建操作手册 | [05-cli-bootstrap/k8s-cluster-setup-guide.md](./05-cli-bootstrap/k8s-cluster-setup-guide.md) |
| kubelet cgroup 源码笔记 | [04-node/kubelet/cgroup.md](./04-node/kubelet/cgroup.md) |
| 飞书稳定性面试问答（90 题） | [interview/feishu-stability-interview-qa.md](./interview/feishu-stability-interview-qa.md) |
| **字节面试综合准备指南** | [interview/interview-prep-guide.md](./interview/interview-prep-guide.md) |
| **运维脚本面试题（75 题）** | [interview/ops-scripting-interview-qa.md](./interview/ops-scripting-interview-qa.md) |

## 推荐学习顺序

| 阶段 | 目录 | 预计时长 | 核心目标 |
|------|------|----------|----------|
| 0 | [00-prerequisites](./00-prerequisites/) | 1-2 周 | Go 语言、K8s 概念、开发环境 |
| 1 | [01-architecture-overview](./01-architecture-overview/) | 3-5 天 | 理解整体架构与源码目录映射 |
| 2 | [02-foundation](./02-foundation/) | 2-3 周 | apimachinery、api、client-go、apiserver 框架 |
| 3 | [03-control-plane](./03-control-plane/) | 3-4 周 | APIServer、Controller Manager、Scheduler |
| 4 | [04-node](./04-node/) | 2-3 周 | Kubelet、Kube-proxy、CRI |
| 5 | [05-cli-bootstrap](./05-cli-bootstrap/) | 1-2 周 | kubectl、kubeadm |
| 6 | [06-core-mechanisms](./06-core-mechanisms/) | 持续深入 | 调度、控制器、准入、认证、存储、网络 |
| 7 | [07-extension](./07-extension/) | 按需 | CRD、Operator、插件机制 |

## 源码目录速查

| 笔记目录 | 对应源码路径 |
|----------|-------------|
| 02-foundation/apimachinery | `staging/src/k8s.io/apimachinery/` |
| 02-foundation/api-types | `staging/src/k8s.io/api/` + `api/` |
| 02-foundation/client-go | `staging/src/k8s.io/client-go/` |
| 02-foundation/apiserver-framework | `staging/src/k8s.io/apiserver/` |
| 03-control-plane/kube-apiserver | `cmd/kube-apiserver/` + `pkg/kubeapiserver/` |
| 03-control-plane/kube-controller-manager | `cmd/kube-controller-manager/` + `pkg/controller/` |
| 03-control-plane/kube-scheduler | `cmd/kube-scheduler/` + `pkg/scheduler/` |
| 04-node/kubelet | `cmd/kubelet/` + `pkg/kubelet/` |
| 04-node/kube-proxy | `cmd/kube-proxy/` + `pkg/proxy/` |
| 05-cli-bootstrap/kubectl | `cmd/kubectl/` + `pkg/kubectl/` + `staging/src/k8s.io/kubectl/` |
| 05-cli-bootstrap/kubeadm | `cmd/kubeadm/` |

## 笔记规范

每篇笔记建议包含以下结构：

1. **学习目标** — 读完本章应掌握什么
2. **源码入口** — 从哪个 `main()` 或关键函数开始读
3. **核心流程** — 调用链 / 时序图
4. **关键数据结构** — 重要 struct 与 interface
5. **个人理解** — 用自己的话总结
6. **疑问与待验证** — 记录不确定的点

## 进度追踪

详见 [progress.md](./progress.md)

## 相关资源

- [Kubernetes 官方开发者文档](https://git.k8s.io/community/contributors/devel)
- [Kubernetes 源码剖析（书籍）](https://github.com/kubernetes/community/tree/master/contributors/devel)
- [Sample Controller 教程](https://github.com/kubernetes/sample-controller)
