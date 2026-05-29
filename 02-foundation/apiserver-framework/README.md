# apiserver-framework

## 源码路径

`kubernetes-master/staging/src/k8s.io/apiserver/`

## 学习目标

- 理解 Generic APIServer 的启动与配置
- 掌握 RESTStorage 接口与 Registry 模式
- 理解 Admission 链、Authentication、Authorization 的接入点

## 关键包

| 包 | 说明 |
|----|------|
| `pkg/server` | GenericAPIServer 定义 |
| `pkg/endpoints` | REST 路由与 Handler |
| `pkg/registry/rest` | RESTStorage 接口 |
| `pkg/admission` | 准入控制框架 |
| `pkg/authentication` | 认证 |
| `pkg/authorization` | 授权 |

## 建议阅读路径

1. `pkg/server/config.go` — Config 结构体
2. `pkg/server/genericapiserver.go` — PrepareRun、Run
3. `pkg/endpoints/installer.go` — API 路由注册
4. `pkg/registry/rest/rest.go` — StandardStorage 接口

## 笔记

（在此记录）
