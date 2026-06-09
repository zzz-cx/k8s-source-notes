# kubeadm

## 源码路径

- 入口：`cmd/kubeadm/kubeadm.go`
- 核心：`cmd/kubeadm/app/`

## 学习目标

- 理解 `kubeadm init` 集群初始化流程
- 理解 `kubeadm join` 节点加入流程
- 了解静态 Pod 清单的生成机制

## 主要子命令

| 命令 | 路径 | 说明 |
|------|------|------|
| init | `app/cmd/init/` | 初始化控制面 |
| join | `app/cmd/join/` | 加入节点 |
| upgrade | `app/cmd/upgrade/` | 集群升级 |
| token | `app/cmd/token/` | Bootstrap Token 管理 |

## 建议阅读路径

1. `cmd/kubeadm/app/cmd/init/init.go` — init 流程
2. `cmd/kubeadm/app/phases/` — 各阶段实现
3. `cmd/kubeadm/app/util/staticpod/` — 静态 Pod 管理

## 笔记

参见 [k8s-cluster-setup-guide.md](../k8s-cluster-setup-guide.md) — 集群搭建完整流程与 `kubeadm init/join` 实操。
