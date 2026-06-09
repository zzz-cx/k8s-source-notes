# Phase 0：前置知识

## 学习目标

- 掌握 Go 语言核心特性（interface、goroutine、channel、context）
- 理解 Kubernetes 核心概念（Pod、Deployment、Service、Namespace 等）
- 搭建本地 K8s 开发/调试环境

## 学习清单

### Go 语言

- 基础语法与类型系统
- interface 与多态
- goroutine 与 channel
- context 包的使用
- 错误处理模式
- 单元测试（testing 包）

### Kubernetes 概念

- 核心功能与组件（见 [01 架构全景笔记](../01-architecture-overview/k8s-basics-features-and-components.md)）
- 控制面 vs 数据面
- 声明式 API 与期望状态
- Informer / List-Watch 机制（概念层面）
- RBAC 与 ServiceAccount
- 常用 kubectl 命令

### 开发环境

- 本地源码编译：`make`、`make quick-release`
- 代码跳转工具（gopls / IDE）
- 本地集群（kind / minikube）用于验证

## 参考资料

- [Go 官方教程](https://go.dev/tour/)
- [Kubernetes 概念文档](https://kubernetes.io/docs/concepts/)

