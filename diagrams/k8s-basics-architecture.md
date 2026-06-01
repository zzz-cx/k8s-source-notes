# Kubernetes 基础功能与组件 — 原理与架构图

> 对应笔记：[k8s-basics-features-and-components.md](../01-architecture-overview/k8s-basics-features-and-components.md)

---

## 1. 整体架构图

Kubernetes 采用**控制面（Control Plane）+ 工作节点（Worker Node）** 的分层架构，所有组件通过 kube-apiserver 统一通信。

```mermaid
flowchart TB
    subgraph User["用户层"]
        kubectl["kubectl / CI/CD / Operator"]
    end

    subgraph CP["控制面 Control Plane"]
        apiserver["kube-apiserver<br/>统一 API 入口"]
        etcd[("etcd<br/>集群状态存储<br/>Raft 强一致")]
        scheduler["kube-scheduler<br/>Pod 调度决策"]
        cm["kube-controller-manager<br/>控制循环"]
        admission["Admission Plugins<br/>准入控制"]
    end

    subgraph Node1["Worker Node"]
        kubelet1["kubelet<br/>Pod 生命周期"]
        proxy1["kube-proxy<br/>Service 转发"]
        runtime1["containerd / CRI-O<br/>容器运行时"]
        cni1["CNI 插件<br/>Pod 网络"]
    end

    subgraph Node2["Worker Node"]
        kubelet2["kubelet"]
        proxy2["kube-proxy"]
        runtime2["containerd"]
        cni2["CNI 插件"]
    end

    kubectl -->|"REST API"| apiserver
    apiserver <-->|"读写状态"| etcd
    apiserver --> admission
    scheduler -->|"Watch / Bind"| apiserver
    cm -->|"Watch / Update"| apiserver

    apiserver -->|"PodSpec 下发"| kubelet1
    apiserver -->|"PodSpec 下发"| kubelet2
    kubelet1 -->|"CRI gRPC"| runtime1
    kubelet2 -->|"CRI gRPC"| runtime2
    proxy1 -.->|"iptables/IPVS"| cni1
    proxy2 -.->|"iptables/IPVS"| cni2
```

---

## 2. 声明式 API 与控制循环原理

Kubernetes 的核心设计：**用户声明期望状态，Controller 持续收敛实际状态**。

```mermaid
flowchart LR
    subgraph Declare["声明期望状态"]
        yaml["YAML / kubectl apply"]
        apiserver["kube-apiserver"]
        etcd[("etcd")]
        yaml --> apiserver --> etcd
    end

    subgraph Loop["控制循环 Reconcile Loop"]
        watch["Informer Watch<br/>监听资源变化"]
        compare{"实际状态<br/>vs<br/>期望状态"}
        action["执行修正动作<br/>创建/删除/更新 Pod"]
        watch --> compare
        compare -->|"不一致"| action
        compare -->|"一致"| watch
        action --> apiserver
    end

    etcd --> watch
```

**典型示例 — Deployment 维持副本数：**

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as kube-apiserver
    participant ETCD as etcd
    participant DC as Deployment Controller
    participant RS as ReplicaSet Controller
    participant S as kube-scheduler
    participant K as kubelet

    User->>API: apply Deployment replicas=3
    API->>ETCD: 持久化 Deployment
    DC->>API: Watch 到 Deployment
    DC->>API: 创建/更新 ReplicaSet
    RS->>API: 创建 3 个 Pod（Pending）
    S->>API: 调度 Pod → Bind Node
    K->>API: Watch 到 PodSpec
    K->>K: CRI 创建容器
    K->>API: 上报 Pod Running

    Note over RS,K: 若 Pod 崩溃被删除
    RS->>API: 发现副本不足
    RS->>API: 自动创建新 Pod（自愈）
```

---

## 3. 资源调度原理

```mermaid
flowchart TD
    start["Pod 创建<br/>spec.nodeName 为空<br/>status = Pending"]
    queue["Scheduler 调度队列"]
    prefilter["PreFilter<br/>预处理"]
    filter["Filter 过滤<br/>资源/污点/亲和性/拓扑"]
    postfilter["PostFilter<br/>抢占或失败"]
    prescore["PreScore"]
    score["Score 评分<br/>资源均衡/亲和性权重"]
    select["选择最高分 Node"]
    bind["Bind 绑定<br/>写入 spec.nodeName"]
    kubelet["kubelet 接管<br/>创建容器"]

    start --> queue --> prefilter --> filter
    filter -->|"无可用节点"| postfilter
    filter -->|"有候选节点"| prescore --> score --> select --> bind --> kubelet

    subgraph Factors["调度考量因素"]
        f1["Requests / Limits"]
        f2["Node Labels"]
        f3["Taint / Toleration"]
        f4["Node/Pod Affinity"]
        f5["Topology Spread"]
        f6["GPU / NUMA（扩展）"]
    end

    Factors -.-> filter
    Factors -.-> score
```

---

## 4. 服务发现与负载均衡

```mermaid
flowchart TB
    subgraph Stable["稳定访问层"]
        client["客户端 / 集群内 Pod"]
        svc["Service<br/>ClusterIP: 10.96.x.x<br/>Label Selector 关联 Pod"]
    end

    subgraph Node["每个 Node 上"]
        proxy["kube-proxy"]
        rules["iptables / IPVS 规则"]
        proxy --> rules
    end

    subgraph Backend["后端 Pod（IP 动态变化）"]
        pod1["Pod A<br/>10.244.1.5"]
        pod2["Pod B<br/>10.244.2.8"]
        pod3["Pod C<br/>10.244.3.2"]
    end

    subgraph CNI["CNI 网络插件"]
        network["Calico / Cilium / Flannel<br/>每个 Pod 独立 IP<br/>Pod 间直接通信"]
    end

    client -->|"访问 ClusterIP"| svc
    svc --> proxy
    rules -->|"负载均衡转发"| pod1
    rules --> pod2
    rules --> pod3
    pod1 & pod2 & pod3 --- network
```

**Service 类型对比：**

```mermaid
flowchart LR
    subgraph Types["Service 类型"]
        clusterIP["ClusterIP<br/>集群内虚拟 IP"]
        nodePort["NodePort<br/>Node IP + 端口"]
        lb["LoadBalancer<br/>云 LB 外部入口"]
        headless["Headless<br/>无 ClusterIP<br/>DNS 直连 Pod"]
    end
```

---

## 5. 自动扩缩容原理

```mermaid
flowchart TB
    subgraph HPA["HPA 水平扩缩容 Pod"]
        metrics1["Metrics Server /<br/>Prometheus 指标"]
        hpa_ctrl["HPA Controller"]
        deploy["Deployment<br/>修改 replicas"]
        metrics1 --> hpa_ctrl --> deploy
    end

    subgraph VPA["VPA 垂直调整资源"]
        vpa_ctrl["VPA Controller"]
        pod_res["Pod Requests/Limits<br/>动态调整"]
        vpa_ctrl --> pod_res
    end

    subgraph CA["Cluster Autoscaler 节点扩缩"]
        pending["Pending Pod<br/>资源不足"]
        ca["Cluster Autoscaler"]
        cloud["云 API 增删 Node"]
        pending --> ca --> cloud
    end

    load["业务流量增加"] --> metrics1
    deploy -->|"Pod 增多"| pending
```

---

## 6. 自愈机制原理

```mermaid
flowchart TD
    subgraph Detect["异常检测"]
        liveness["kubelet<br/>Liveness Probe 失败"]
        crash["容器异常退出"]
        node_down["Node 心跳丢失"]
    end

    subgraph React["自动恢复"]
        kill["kubelet 重启/杀死容器"]
        rs["ReplicaSet Controller<br/>重建 Pod"]
        nc["Node Controller<br/>标记 NotReady<br/>触发 Pod 驱逐"]
    end

    subgraph Principle["核心理念：不可变基础设施"]
        destroy["销毁异常实例"]
        recreate["重新创建新实例"]
        destroy --> recreate
    end

    liveness --> kill --> rs
    crash --> rs
    node_down --> nc --> rs
    rs --> Principle
```

---

## 7. 配置管理原理

```mermaid
flowchart LR
    subgraph Config["配置资源"]
        cm["ConfigMap<br/>非敏感配置"]
        secret["Secret<br/>密码/Token/证书"]
    end

    subgraph Inject["注入方式"]
        env["环境变量"]
        vol["Volume 挂载"]
        cmd["启动命令参数"]
    end

    subgraph Pod["Pod"]
        container["应用容器"]
    end

    cm --> env & vol
    secret --> env & vol
    env & vol & cmd --> container

    note["配置与镜像解耦<br/>修改 ConfigMap 可触发滚动更新"]
```

---

## 8. 存储编排原理

```mermaid
flowchart TB
    subgraph User["用户声明"]
        pvc["PVC<br/>我需要 10Gi 读写存储"]
        sc["StorageClass<br/>存储类型模板"]
    end

    subgraph Control["控制面"]
        prov["Provisioner<br/>动态供给控制器"]
        pv["PV<br/>实际存储卷"]
        pvc --> prov
        sc --> prov
        prov -->|"动态创建"| pv
        pvc -->|"绑定 Bind"| pv
    end

    subgraph Backend["底层存储"]
        ceph["Ceph"]
        nfs["NFS"]
        cloud_disk["云磁盘 EBS/SSD"]
    end

    subgraph Pod["Pod 使用"]
        mount["Volume 挂载到容器"]
    end

    pv --> ceph & nfs & cloud_disk
    pv --> mount

    principle["存储资源抽象化<br/>应用与底层存储解耦"]
```

---

## 9. 滚动更新与回滚原理

```mermaid
flowchart TD
    subgraph Deploy["Deployment 管理"]
        dep["Deployment<br/>replicas=3"]
        rs_old["ReplicaSet v1<br/>3 Pods 旧版本"]
        rs_new["ReplicaSet v2<br/>逐步扩容"]
        dep --> rs_old
        dep --> rs_new
    end

    subgraph Rolling["Rolling Update 过程"]
        step1["1. 创建新 RS，新 Pod=1，旧 Pod=3"]
        step2["2. 新 Pod=2，旧 Pod=2"]
        step3["3. 新 Pod=3，旧 Pod=0"]
        step1 --> step2 --> step3
    end

    subgraph Rollback["回滚"]
        rev["Revision 历史记录"]
        rollback["kubectl rollout undo<br/>切回旧 ReplicaSet"]
        rev --> rollback
    end

    rs_new --> Rolling
    Rolling -->|"升级失败"| Rollback
```

**滚动更新时序：**

```mermaid
sequenceDiagram
    participant User as 用户
    participant DC as Deployment Controller
    participant API as kube-apiserver
    participant Old as RS v1 Pods
    participant New as RS v2 Pods

    User->>API: 更新 Deployment 镜像
    DC->>API: 创建 RS v2
    DC->>API: RS v2 replicas +1
    New->>New: 新 Pod Ready
    DC->>API: RS v1 replicas -1
    Note over DC,New: 重复直到 RS v1=0, RS v2=3
```

---

## 10. kube-apiserver 请求处理链路

所有组件必须通过 APIServer 通信，一次写请求经过完整的认证授权与准入链路。

```mermaid
flowchart TD
    req["HTTP 请求<br/>kubectl / kubelet / controller"]
    authn["Authentication 认证<br/>X509 / Token / SA"]
    authz["Authorization 授权<br/>RBAC / Node / Webhook"]
    mutate["Mutating Admission<br/>修改对象默认值"]
    validate["Validating Admission<br/>校验对象合法性"]
    registry["Registry / RESTStorage"]
    etcd[("etcd 持久化")]

    req --> authn --> authz --> mutate --> validate --> registry --> etcd

    authn -->|"失败"| reject1["401 Unauthorized"]
    authz -->|"失败"| reject2["403 Forbidden"]
    validate -->|"失败"| reject3["422 拒绝创建"]
```

---

## 11. 核心资源对象关系图

```mermaid
flowchart TB
    subgraph Workload["工作负载 Workloads"]
        deploy["Deployment<br/>无状态应用"]
        sts["StatefulSet<br/>有状态应用"]
        ds["DaemonSet<br/>每节点一个"]
        job["Job / CronJob<br/>批处理任务"]
        pod["Pod<br/>最小调度单位"]
    end

    subgraph Network["网络 Network"]
        svc["Service<br/>稳定访问入口"]
        ing["Ingress<br/>七层 HTTP 路由"]
    end

    subgraph Config["配置 Config"]
        cm["ConfigMap"]
        secret["Secret"]
    end

    subgraph Storage["存储 Storage"]
        pvc["PVC 存储申请"]
        pv["PV 实际存储"]
        sc["StorageClass"]
    end

    deploy -->|"管理"| pod
    sts -->|"管理"| pod
    ds -->|"管理"| pod
    job -->|"管理"| pod

    svc -->|"Label Selector"| pod
    ing -->|"路由到"| svc

    cm & secret -->|"挂载/注入"| pod
    pvc -->|"绑定"| pv
    sc -->|"动态创建"| pv
    pvc -->|"挂载"| pod

    deploy -.->|"内部通过"| rs["ReplicaSet"]
    rs --> pod
```

**资源对象分类速查：**

```mermaid
mindmap
  root((K8s 资源对象))
    工作负载
      Pod
      Deployment
      StatefulSet
      DaemonSet
      Job
      CronJob
    网络
      Service
      Ingress
      NetworkPolicy
    配置
      ConfigMap
      Secret
    存储
      PV
      PVC
      StorageClass
    集群
      Node
      Namespace
      RBAC
```

---

## 12. Node 内部组件协作

```mermaid
flowchart TB
    subgraph Node["Worker Node"]
        apiserver["kube-apiserver"]

        subgraph kubelet_sys["kubelet"]
            sync["syncLoop<br/>Pod 同步循环"]
            pleg["PLEG<br/>容器生命周期事件"]
            probe["Probe Manager<br/>Liveness/Readiness"]
            vol_mgr["Volume Manager<br/>挂载 PV/ConfigMap"]
        end

        proxy["kube-proxy<br/>Service 规则维护"]
        runtime["Container Runtime<br/>containerd via CRI"]
        cni["CNI<br/>Pod 网络配置"]

        apiserver -->|"Watch PodSpec"| sync
        sync --> runtime
        sync --> vol_mgr
        sync --> probe
        runtime --> pleg
        pleg --> sync
        proxy --> cni
    end
```

---

## 13. Pod 创建全链路（综合）

将调度、配置、存储、网络串联的完整流程：

```mermaid
sequenceDiagram
    participant U as 用户
    participant API as kube-apiserver
    participant ETCD as etcd
    participant DC as Deployment Controller
    participant S as kube-scheduler
    participant K as kubelet
    participant CRI as containerd
    participant CNI as CNI 插件
    participant P as kube-proxy

    U->>API: kubectl apply Deployment
    API->>ETCD: 存储 Deployment
    DC->>API: 创建 ReplicaSet + Pod
    API->>ETCD: Pod Pending
    S->>API: Filter + Score + Bind Node
    K->>API: Watch 到新 Pod
    K->>K: 挂载 ConfigMap/Secret/PVC
    K->>CRI: RunPodSandbox + CreateContainer
    CRI->>CNI: 配置 Pod 网络
    K->>API: 上报 Pod Running
    P->>API: Watch Service Endpoints
    P->>P: 更新 iptables/IPVS 规则
```

---

## 图例说明

| 符号 | 含义 |
|------|------|
| 实线箭头 | 直接调用 / 数据流 |
| 虚线箭头 | 间接依赖 / 辅助关系 |
| 圆柱体 | 持久化存储（etcd、PV） |
| subgraph | 逻辑分组 |

## 相关源码入口

| 图 | 主要源码路径 |
|----|-------------|
| 控制循环 | `pkg/controller/deployment/` |
| 调度 | `pkg/scheduler/scheduler.go` |
| Service 网络 | `pkg/proxy/` |
| HPA | `pkg/controller/podautoscaler/` |
| 存储 | `pkg/controller/volume/persistentvolume/` |
| APIServer 链路 | `staging/src/k8s.io/apiserver/pkg/endpoints/` |
