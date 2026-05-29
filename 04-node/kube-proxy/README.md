# kube-proxy

## 源码路径

- 入口：`cmd/kube-proxy/proxy.go`
- 核心：`pkg/proxy/`

## 学习目标

- 理解 kube-proxy 的三种模式（iptables、ipvs、nftables）
- 掌握 Service → Endpoints 的规则生成
- 理解 NodePort、ClusterIP 的实现

## 建议阅读路径

1. `cmd/kube-proxy/app/server.go` — 启动与模式选择
2. `pkg/proxy/iptables/proxier.go` — iptables 模式
3. `pkg/proxy/ipvs/proxier.go` — ipvs 模式

## 笔记

（在此记录）
