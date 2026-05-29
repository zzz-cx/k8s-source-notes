# kubectl

## 源码路径

- 入口：`cmd/kubectl/kubectl.go`
- 核心：`pkg/kubectl/` + `staging/src/k8s.io/kubectl/`

## 学习目标

- 理解 Cobra 命令框架的组织方式
- 掌握 `kubectl get/create/apply/delete` 的实现
- 理解 Builder 模式（resource.Builder）

## 建议阅读路径

1. `cmd/kubectl/kubectl.go` — main 入口
2. `staging/src/k8s.io/kubectl/pkg/cmd/cmd.go` — 命令注册
3. `staging/src/k8s.io/cli-runtime/pkg/resource/builder.go` — 资源构建器
4. `pkg/kubectl/cmd/get/get.go` — get 命令实现

## 笔记

（在此记录）
