# api-types

## 源码路径

- `kubernetes-master/staging/src/k8s.io/api/` — 各 API Group 的类型定义
- `kubernetes-master/api/` — OpenAPI 规范

## 学习目标

- 理解 API Group 划分（core/v1、apps/v1、batch/v1 等）
- 掌握 Pod、Deployment、Service 等核心类型的字段含义
- 了解 API 版本演进与 Conversion

## 关键 API Group

| Group | 路径 | 核心资源 |
|-------|------|----------|
| core/v1 | `api/core/v1/` | Pod, Service, Node, Namespace |
| apps/v1 | `api/apps/v1/` | Deployment, ReplicaSet, StatefulSet, DaemonSet |
| batch/v1 | `api/batch/v1/` | Job, CronJob |
| networking/v1 | `api/networking/v1/` | NetworkPolicy, Ingress |
| rbac/v1 | `api/rbac/v1/` | Role, RoleBinding, ClusterRole |

## 建议阅读路径

1. `api/core/v1/types.go` — PodSpec、Container
2. `api/apps/v1/types.go` — DeploymentSpec
3. `api/meta/v1/types.go` — 与 apimachinery 的关系

## 笔记

（在此记录）
