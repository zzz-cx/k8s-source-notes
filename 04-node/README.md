# Phase 4：节点组件

## 子目录

| 目录 | 源码路径 | 说明 |
|------|----------|------|
| [kubelet](./kubelet/) | `cmd/kubelet/` + `pkg/kubelet/` | 节点代理，Pod 生命周期管理 |
| [kube-proxy](./kube-proxy/) | `cmd/kube-proxy/` + `pkg/proxy/` | Service 网络代理 |
| [cri](./cri/) | `staging/src/k8s.io/cri-api/` + `pkg/kubelet/kuberuntime/` | 容器运行时接口 |

## 学习顺序

```
kubelet → cri → kube-proxy
```

## 笔记

（在此记录）
