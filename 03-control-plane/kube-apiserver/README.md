# kube-apiserver

## 源码路径

- 入口：`cmd/kube-apiserver/apiserver.go`
- 核心：`pkg/kubeapiserver/`、`pkg/registry/`

## 学习目标

- 理解 APIServer 启动流程（CreateServerChain）
- 掌握 Pod 等资源的 REST 路由注册
- 理解 etcd 存储层（`pkg/storage/etcd3/`）

## 关键流程

```
main() → CreateServerChain() → GenericAPIServer.PrepareRun() → Run()
```

## 建议阅读路径

1. `cmd/kube-apiserver/app/server.go` — 启动入口
2. `pkg/kubeapiserver/default_storage_factory_builder.go` — 存储工厂
3. `pkg/registry/core/pod/storage/` — Pod REST 实现
4. `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go` — etcd 读写

## 笔记

（在此记录）
