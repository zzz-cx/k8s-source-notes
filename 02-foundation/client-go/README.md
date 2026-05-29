# client-go

## 源码路径

`kubernetes-master/staging/src/k8s.io/client-go/`

## 学习目标

- 掌握 REST Client 的构建与请求流程
- 理解 SharedInformer 与 DeltaFIFO
- 理解 WorkQueue 在 Controller 中的作用

## 关键包

| 包 | 说明 |
|----|------|
| `rest` | RESTClient、Request、Result |
| `tools/cache` | SharedInformer、Reflector、DeltaFIFO |
| `tools/cache/synctrack` | Informer 同步追踪 |
| `util/workqueue` | 速率限制队列 |
| `kubernetes` | 各资源的 typed client |

## 建议阅读路径

1. `rest/client.go` + `rest/request.go` — HTTP 请求封装
2. `tools/cache/shared_informer.go` — Informer 核心
3. `tools/cache/reflector.go` — List-Watch 实现
4. `util/workqueue/rate_limiting_queue.go` — 工作队列

## 经典调用链

```
Reflector.ListAndWatch()
  → DeltaFIFO
  → SharedInformer.OnAdd/OnUpdate/OnDelete
  → WorkQueue.Add()
  → Controller.processNextWorkItem()
```

## 笔记

（在此记录）
