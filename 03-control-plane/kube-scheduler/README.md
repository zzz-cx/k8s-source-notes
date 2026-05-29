# kube-scheduler

## 源码路径

- 入口：`cmd/kube-scheduler/scheduler.go`
- 核心：`pkg/scheduler/`

## 学习目标

- 理解调度框架（Scheduling Framework）的扩展点
- 掌握 Predicates（过滤）与 Priorities（打分）机制
- 理解 Pod 绑定（Bind）流程

## 调度框架扩展点

```
PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → PreBind → Bind → PostBind
```

## 建议阅读路径

1. `cmd/kube-scheduler/app/server.go` — 启动入口
2. `pkg/scheduler/scheduler.go` — ScheduleOne 主循环
3. `pkg/scheduler/framework/runtime/framework.go` — 插件框架
4. `pkg/scheduler/framework/plugins/` — 内置插件

## 笔记

（在此记录）
