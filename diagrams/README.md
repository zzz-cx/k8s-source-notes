# 架构图

在此目录存放学习过程中绘制的架构图、时序图、流程图。

建议格式：`.md`（Mermaid）、`.png`、`.svg`

## 已绘制

| 图 | 文件 | 说明 |
|----|------|------|
| **K8s 核心功能与组件总览** | [k8s-basics-overview.png](./k8s-basics-overview.png) | 一图速览：功能、组件、资源对象、协作、调度、网络 |
| K8s 基础原理与架构全集 | [k8s-basics-architecture.md](./k8s-basics-architecture.md) | 对应「基础功能与组件」笔记，含 13 张 Mermaid 图 |

### k8s-basics-architecture.md 目录

1. 整体架构图（控制面 + Worker Node）
2. 声明式 API 与控制循环原理
3. 资源调度原理（Filter / Score / Bind）
4. 服务发现与负载均衡
5. 自动扩缩容（HPA / VPA / Cluster Autoscaler）
6. 自愈机制原理
7. 配置管理原理（ConfigMap / Secret）
8. 存储编排原理（PV / PVC / StorageClass）
9. 滚动更新与回滚原理
10. kube-apiserver 请求处理链路
11. 核心资源对象关系图
12. Node 内部组件协作
13. Pod 创建全链路（综合时序图）

## 待绘制

- [ ] Informer 工作原理图
- [ ] RBAC 授权链路图
- [ ] CRI 调用时序图
