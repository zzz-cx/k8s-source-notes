# apimachinery

## 源码路径

`kubernetes-master/staging/src/k8s.io/apimachinery/`

## 学习目标

- 理解 K8s 对象模型（runtime.Object、TypeMeta、ObjectMeta）
- 掌握 Scheme 的类型注册机制
- 理解 List-Watch 与 Reflector 原理

## 关键包与文件

| 包 | 关键文件 | 说明 |
|----|----------|------|
| `pkg/runtime` | `object.go`, `scheme.go` | 对象接口与 Scheme |
| `pkg/apis/meta/v1` | `types.go` | ObjectMeta、ListMeta |
| `pkg/watch` | `watch.go` | Watch 事件类型 |
| `pkg/util/wait` | `wait.go` | 轮询与退避 |
| `pkg/labels` | `selector.go` | Label Selector |

## 建议阅读路径

1. `pkg/runtime/interfaces.go` — Object、GroupVersioner 接口
2. `pkg/runtime/scheme.go` — AddKnownTypes、Convert
3. `pkg/apis/meta/v1/types.go` — 通用元数据
4. `pkg/util/runtime` — panic 处理

## 笔记

（在此记录）
