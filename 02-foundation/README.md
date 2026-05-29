# Phase 2：基础库

> 这是阅读 K8s 源码的基石，建议按顺序学习。

## 子目录

| 目录 | 源码路径 | 说明 |
|------|----------|------|
| [apimachinery](./apimachinery/) | `staging/src/k8s.io/apimachinery/` | 对象模型、Scheme、序列化、List-Watch |
| [api-types](./api-types/) | `staging/src/k8s.io/api/` | Pod、Deployment 等 API 类型定义 |
| [client-go](./client-go/) | `staging/src/k8s.io/client-go/` | Client、Informer、WorkQueue |
| [apiserver-framework](./apiserver-framework/) | `staging/src/k8s.io/apiserver/` | 通用 APIServer 框架 |

## 学习顺序

```
apimachinery → api-types → client-go → apiserver-framework
```

## 核心概念

- **runtime.Object** — 所有 K8s 对象的基类
- **Scheme** — 类型注册与转换
- **GVK / GVR** — Group-Version-Kind / Group-Version-Resource
- **Informer** — 本地缓存 + 事件驱动
- **RESTStorage** — APIServer 存储抽象

## 笔记

（在此记录跨模块的综合理解）
