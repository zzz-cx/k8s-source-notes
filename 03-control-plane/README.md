# Phase 3：控制面组件

## 子目录

| 目录 | 源码路径 | 说明 |
|------|----------|------|
| [kube-apiserver](./kube-apiserver/) | `cmd/kube-apiserver/` + `pkg/kubeapiserver/` | API 网关，唯一写 etcd 的组件 |
| [kube-controller-manager](./kube-controller-manager/) | `cmd/kube-controller-manager/` + `pkg/controller/` | 各类控制器 |
| [kube-scheduler](./kube-scheduler/) | `cmd/kube-scheduler/` + `pkg/scheduler/` | Pod 调度 |
| [cloud-controller-manager](./cloud-controller-manager/) | `cmd/cloud-controller-manager/` | 云厂商集成控制器 |

## 学习顺序

```
kube-apiserver → kube-controller-manager → kube-scheduler
```

## 笔记

（在此记录跨组件的综合理解）
