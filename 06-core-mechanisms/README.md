# Phase 6：核心机制

> 这些主题贯穿多个组件，建议在完成 Phase 2-5 后按需深入。

## 子目录

| 目录 | 主要源码路径 | 说明 |
|------|-------------|------|
| [scheduling](./scheduling/) | `pkg/scheduler/` | 调度算法与插件 |
| [controllers](./controllers/) | `pkg/controller/` | 控制器模式与实践 |
| [admission](./admission/) | `plugin/pkg/admission/` + `pkg/admission/` | 准入控制 |
| [auth-rbac](./auth-rbac/) | `plugin/pkg/auth/` + `pkg/auth/` | 认证与授权 |
| [volume](./volume/) | `pkg/volume/` + `pkg/controller/volume/` | 存储卷 |
| [networking](./networking/) | `pkg/proxy/` + CNI 相关 | 网络 |

## 笔记

（在此记录跨主题的综合理解）
