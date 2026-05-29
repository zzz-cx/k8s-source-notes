# cloud-controller-manager

## 源码路径

- 入口：`cmd/cloud-controller-manager/main.go`
- 核心：`staging/src/k8s.io/cloud-provider/`

## 学习目标

- 理解 CCM 与 KCM 的职责分离
- 了解 Node、Route、Service 等云控制器

## 主要控制器

- Node Controller — 云实例生命周期
- Route Controller — 云路由表
- Service Controller — LoadBalancer 类型 Service

## 笔记

（在此记录）
