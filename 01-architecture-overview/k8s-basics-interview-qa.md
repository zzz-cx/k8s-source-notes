# 单元：K8s 基础功能与组件 — 面试问答

> 关联笔记：[k8s-basics-features-and-components.md](./k8s-basics-features-and-components.md) · [架构图](../diagrams/k8s-basics-architecture.md)

本单元整理 Kubernetes 基础面试高频题，结合已有学习笔记给出**标准答法**、**追问方向**和**自测记录**。

---

## Q1：K8s 中的 Pod 是什么？

### 标准答案

**Pod 是 Kubernetes 中最小的调度与部署单元**，可以包含**一个或多个**紧密关联的容器。同一 Pod 内的容器共享：

- **网络命名空间**（同一 IP、同一端口空间，通过 localhost 通信）
- **存储卷**（Volume 可共享挂载）
- **生命周期**（一起调度、一起终止）

Pod 是对「运行中的应用实例」的抽象，而不是对单个容器的抽象。

### 一句话版（面试开场）

> Pod 是 K8s 最小调度单位，由一个或多个共享网络与存储的容器组成。

### 追问：为什么 Pod 是最小管理/调度单元，而不是 Container？

| 维度 | 说明 |
|------|------|
| **协同调度** | Sidecar、代理、日志采集等辅助容器必须与主应用**同生共死、同节点部署** |
| **共享网络** | 多容器通过 `127.0.0.1` 通信，比跨 Pod 通信更高效、更简单 |
| **共享存储** | 如主容器写日志、Sidecar 采集，需共享 Volume |
| **统一生命周期** | 调度、重启、销毁以 Pod 为粒度，保证「应用 + 其伴生服务」状态一致 |

**面试答法模板：**

> Pod 设计是为了把「必须部署在一起」的应用组件绑成一个单元。比如主业务容器 + Envoy 代理 + 日志 Sidecar，它们需要共享网络 IP、共享存储、一起调度。如果以 Container 为最小单元，就无法保证这些容器始终在同一节点、同一网络栈中运行。

### 常见 Sidecar 场景

- Service Mesh（istio-proxy）
- 日志采集（fluent-bit）
- 配置热加载（config-reloader）

> **对应源码**：`api/core/v1/types.go` → `Pod`、`PodSpec`

---

## Q2：K8s 主节点（控制面）的主要组件有哪些？

### 标准答案

| 组件 | 职责 |
|------|------|
| **kube-apiserver** | 集群唯一入口，提供 REST API，认证/授权/准入，与 etcd 交互 |
| **etcd** | 分布式 KV 存储，保存集群全部状态（Raft 强一致） |
| **kube-scheduler** | 为 Pending Pod 选择最优 Node 并 Bind |
| **kube-controller-manager** | 运行各类 Controller，持续 Reconcile 期望状态与实际状态 |

### 一句话版

> 控制面四个核心：APIServer 是入口，etcd 存状态，Scheduler 管调度，Controller Manager 管控制循环。

### 追问：鉴权（Authentication / Authorization）在哪个组件完成？

**答案：kube-apiserver**

一次请求在 APIServer 内的处理顺序：

```
Authentication（认证：你是谁）
  → Authorization（授权：你能做什么，RBAC）
    → Mutating Admission（准入：可修改对象）
      → Validating Admission（准入：校验合法性）
        → 写入 etcd
```

| 环节 | 说明 | 常见实现 |
|------|------|----------|
| **Authentication** | 验证身份 | X509 证书、Bearer Token、ServiceAccount Token |
| **Authorization** | 检查权限 | RBAC、Node Authorization、Webhook |
| **Admission** | 创建前的策略检查 | ResourceQuota、PodSecurity、自定义 Webhook |

> **记忆口诀**：**所有进出集群的 API 请求都经过 APIServer，认证授权也在 APIServer 内完成**，Scheduler 和 Controller 本身不对外做鉴权。

> **对应源码**：`staging/src/k8s.io/apiserver/pkg/authentication/`、`pkg/authorization/`、`pkg/admission/`

---

## Q3：ConfigMap 和 Secret 的作用和使用场景？

### 标准答案

| 资源 | 作用 | 典型场景 |
|------|------|----------|
| **ConfigMap** | 存储**非敏感**配置，与镜像解耦 | 配置文件、环境变量、命令行参数 |
| **Secret** | 存储**敏感**数据（Base64 编码，非加密） | 密码、TLS 证书、Docker 镜像拉取凭证、API Token |

### 使用方式

1. **环境变量注入**：`envFrom` / `valueFrom`
2. **Volume 挂载**：以文件形式挂载到容器内
3. **命令行参数**：启动命令引用挂载路径

### 面试加分点

- ConfigMap / Secret 更新后，Pod **不会自动重启**；需配合 Reloader 或滚动更新
- Secret 默认仅 Base64，**生产环境应启用 etcd 加密**（Encryption at Rest）
- 可将 ConfigMap / Secret 挂载为 Volume，实现配置与镜像分离

> **对应笔记**：[k8s-basics-features-and-components.md §6 配置管理](./k8s-basics-features-and-components.md)

---

## Q4：Deployment 和 StatefulSet 的区别？

### 对比表（面试直接背这张）

| 维度 | Deployment | StatefulSet |
|------|------------|-------------|
| **适用场景** | 无状态应用（Web、API） | 有状态应用（MySQL、Redis、Kafka） |
| **Pod 名称** | 随机（`xxx-7d4f8b-abcde`） | 固定有序（`web-0`、`web-1`、`web-2`） |
| **网络标识** | 无固定身份 | 稳定 DNS：`pod-name.service-name` |
| **存储** | 所有 Pod 共享相同 PVC 模板（通常无持久卷） | 每 Pod 独立 PVC（`volumeClaimTemplates`） |
| **部署/扩缩** | 并行，无序 | **有序**：扩容 0→1→2，缩容 2→1→0 |
| **更新策略** | RollingUpdate / Recreate | RollingUpdate / OnDelete |

### 一句话版

> Deployment 管无状态，Pod 可替代；StatefulSet 管有状态，Pod 有固定身份、独立存储、有序启停。

### 选型口诀

- 删了 Pod 换一个新的，业务无影响 → **Deployment**
- Pod 必须保留名字、顺序、磁盘数据 → **StatefulSet**

> **对应源码**：`pkg/controller/deployment/`、`pkg/controller/statefulset/`

---

## Q5：Pod 一直处于 Pending 状态，可能是什么原因？

### 标准答案（分类排查）

#### A. 调度阶段失败（最常见）

| 原因 | 说明 | 排查命令 |
|------|------|----------|
| **资源不足** | 集群无 Node 满足 CPU/Memory/GPU Requests | `kubectl describe pod` → Events |
| **节点 Selector 不匹配** | `nodeSelector` / `nodeAffinity` 无可用节点 | 看 `spec.nodeSelector` |
| **污点无容忍** | Node 有 Taint，Pod 没有对应 Toleration | `kubectl describe node` |
| **亲和/反亲和** | Pod Affinity / Anti-Affinity 约束无法满足 | 看 `spec.affinity` |
| **拓扑分布约束** | TopologySpreadConstraints 无法满足 | 看 Events |
| **PVC 未绑定** | 存储卷 Pending，Pod 无法调度 | `kubectl get pvc` |
| **Scheduler 未运行** | 控制面组件异常 | 检查 scheduler 日志 |

#### B. 已调度但未启动（严格说状态可能已是 ContainerCreating）

| 原因 | 说明 |
|------|------|
| **镜像拉取失败** | 镜像不存在、无权限、网络问题 → `ImagePullBackOff` |
| **资源配额** | Namespace ResourceQuota 超限 |
| **准入拒绝** | PodSecurity、ResourceQuota Admission 拒绝 |

### 排查流程

```bash
kubectl describe pod <pod-name>    # 看 Events 和 Conditions
kubectl get events --sort-by='.lastTimestamp'
kubectl get nodes                  # 节点是否 Ready
kubectl get pvc                    # 存储是否 Bound
```

### 重点补强：污点（Taint）与容忍（Toleration）

**Taint** 是打在 Node 上的「排斥标记」，**Toleration** 是 Pod 上的「容忍声明」。

```
Node Taint: key=gpu, value=true, effect=NoSchedule
  → 默认 Pod 无法调度到该节点
  → 只有带了对应 Toleration 的 Pod 才能调度上去
```

| Effect | 含义 |
|--------|------|
| **NoSchedule** | 不允许调度（除非有 Toleration） |
| **PreferNoSchedule** | 尽量不调度，非强制 |
| **NoExecute** | 不允许调度，且驱逐已有 Pod |

**典型场景：**

- Master 节点默认 Taint `node-role.kubernetes.io/control-plane:NoSchedule`，防止业务 Pod 调度到控制面
- GPU 节点 Taint，只允许 GPU 工作负载调度
- 节点维护时临时 Taint，配合驱逐

**面试答法：**

> Pending 如果是调度问题，常见有：资源 Requests 不满足、nodeSelector/affinity 不匹配、**节点污点而 Pod 没有对应 toleration**、PVC 未绑定。我会先看 `kubectl describe pod` 的 Events。

> **对应源码**：`pkg/scheduler/framework/plugins/tainttoleration/`、`pkg/controller/nodelifecycle/`

---

## Q6：Service 负载均衡有哪些类型？

### 四种类型

| 类型 | 说明 |
|------|------|
| **ClusterIP** | 集群内部虚拟 IP，仅集群内访问（默认） |
| **NodePort** | 在每个 Node 上开放固定端口（30000-32767），外部可通过 `NodeIP:NodePort` 访问 |
| **LoadBalancer** | 云厂商创建外部负载均衡器，自动关联 NodePort |
| **ExternalName** | DNS CNAME 映射，将 Service 映射到外部域名 |

> 补充：**Headless Service**（`clusterIP: None`）不是独立 type，而是 ClusterIP 的特殊配置，用于 StatefulSet 稳定 DNS。

### 追问：ClusterIP、NodePort、LoadBalancer 的区别？

| 维度 | ClusterIP | NodePort | LoadBalancer |
|------|-----------|----------|--------------|
| **访问范围** | 仅集群内 | 集群外 via NodeIP:Port | 集群外 via 云 LB 公网/内网 IP |
| **暴露方式** | 虚拟 IP（kube-proxy 转发） | 每节点开同一端口 | 云厂商 LB → NodePort |
| **典型场景** | 微服务内部调用 | 开发测试、无 LB 环境 | 生产对外暴露 |
| **关系** | 基础 | ClusterIP + NodePort | NodePort + 云 LB |

**层层包含关系：**

```
LoadBalancer Service
  └── 自动创建 NodePort
        └── 底层仍是 ClusterIP + Endpoints
              └── kube-proxy 做四层负载均衡
```

### 一句话对比

- **ClusterIP**：集群内访问，Service VIP → Pod
- **NodePort**：Node IP + 固定端口 → Service → Pod
- **LoadBalancer**：云 LB 公网 IP → NodePort → Service → Pod

> **对应笔记**：[k8s-basics-features-and-components.md §3 服务发现](./k8s-basics-features-and-components.md) · [架构图 §4](../diagrams/k8s-basics-architecture.md)

---

## 模拟面试速记卡

```
Pod        = 最小调度单元，多容器共享网络/存储/生命周期
控制面      = APIServer + etcd + Scheduler + Controller Manager
鉴权        = APIServer（Authn → Authz → Admission → etcd）
ConfigMap  = 非敏感配置；Secret = 敏感数据
Deployment = 无状态；StatefulSet = 有状态（固定名/存储/有序）
Pending    = 调度（资源/亲和/污点/PVC）+ 镜像拉取
Service    = ClusterIP（内）→ NodePort（Node端口）→ LoadBalancer（云LB）
```

---

## 下一步复习建议

1. **污点与容忍** — 手动给节点打 Taint，创建带 Toleration 的 Pod 验证调度
2. **Pending 排查** — 故意制造资源不足、镜像错误，练习 `describe pod` 读 Events
3. **鉴权链路** — 画 APIServer 内 Authn → Authz → Admission 三步
4. **Service 三层** — 在笔记架构图 §4 对照 ClusterIP / NodePort / LB 数据流

## 自测记录

| 日期 | 形式 | 得分项 | 待改进 |
|------|------|--------|--------|
| 2026-05-29 | 口述模拟 | Pod、控制面、ConfigMap/Secret、Deploy/STS、Service 类型 | 污点容忍、Pod 最小单元追问表述 |
