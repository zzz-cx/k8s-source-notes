# Phase 5：CLI 与集群引导

## 子目录

| 目录 | 源码路径 | 说明 |
|------|----------|------|
| [kubectl](./kubectl/) | `cmd/kubectl/` + `pkg/kubectl/` | 命令行工具 |
| [kubeadm](./kubeadm/) | `cmd/kubeadm/` | 集群初始化与加入 |

## 已有笔记

| 笔记 | 内容 |
|------|------|
| [k8s-cluster-setup-guide.md](./k8s-cluster-setup-guide.md) | kubeadm 一主两从集群搭建完整操作手册 |

## 建议学习顺序

1. 按 [k8s-cluster-setup-guide.md](./k8s-cluster-setup-guide.md) 动手搭建集群
2. 对照 [kubeadm 源码目录](./kubeadm/README.md) 理解 `init` / `join` 流程
3. 使用 [kubectl 笔记](./kubectl/README.md) 练习日常运维命令
