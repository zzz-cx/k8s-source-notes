# kubelet

## 源码路径

- 入口：`cmd/kubelet/kubelet.go`
- 核心：`pkg/kubelet/`

## 学习目标

- 理解 Kubelet 启动与配置
- 掌握 Pod Sync 循环（syncLoop）
- 理解 PLEG（Pod Lifecycle Event Generator）
- 理解 Probe（Liveness/Readiness/Startup）

## 关键子系统

| 模块 | 路径 | 说明 |
|------|------|------|
| Pod Manager | `pkg/kubelet/pod/` | Pod 集合管理 |
| Pod Workers | `pkg/kubelet/pod_workers.go` | 按 Pod 串行处理 |
| PLEG | `pkg/kubelet/pleg/` | 容器事件感知 |
| Probe Manager | `pkg/kubelet/prober/` | 健康检查 |
| Volume Manager | `pkg/kubelet/volumemanager/` | 卷挂载 |
| CRI Runtime | `pkg/kubelet/kuberuntime/` | 容器运行时调用 |

## 建议阅读路径

1. `cmd/kubelet/app/server.go` — Run 入口
2. `pkg/kubelet/kubelet.go` — syncLoop
3. `pkg/kubelet/kuberuntime/kuberuntime_manager.go` — 容器操作

## 笔记

（在此记录）
