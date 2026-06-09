# K8s 集群搭建基本操作（kubeadm）

> 参考笔记：[【kubernetes集群部署保姆级教程】](https://blog.csdn.net/yu33575/article/details/135387548)  
> 对应源码：`cmd/kubeadm/` · [kubeadm 学习目录](./kubeadm/README.md)

本文整理使用 **kubeadm** 搭建 Kubernetes 集群的完整操作流程，适用于学习环境（一主两从）。生产环境需额外考虑高可用、安全加固与版本选型。

---

## 一、环境规划

### 1.1 部署方案

| 项目 | 选择 |
|------|------|
| 拓扑 | **一主两从**（1 Master + 2 Worker） |
| 部署工具 | **kubeadm**（官方推荐快捷方式） |
| K8s 版本 | **v1.23.14** |
| 容器运行时 | **Docker 20.10.16** |
| CNI 网络 | **Calico v3.25** |
| 操作系统 | CentOS 8 Stream（内核 ≥ 4.0） |

> **版本说明**：K8s 1.23 仍支持通过 Docker + dockershim 运行。1.24 起移除 dockershim，需改用 containerd/CRI-O。本文按参考笔记的 1.23.14 + Docker 栈记录。

### 1.2 主机规划

| 角色 | 主机名 | IP 地址（示例，请按实际修改） | 说明 |
|------|--------|-------------------------------|------|
| Master | `k8s-master` | `192.168.246.156` | 控制面 |
| Worker | `k8s-node1` | `192.168.246.253` | 工作节点 |
| Worker | `k8s-node2` | `192.168.246.254` | 工作节点 |

**硬件建议（学习环境）**：每节点 ≥ 2 CPU、≥ 2Gi 内存（官方最低 2G，建议 4G+）。

---

## 二、构建流程总览

```
1. 环境准备     → 三台节点统一配置（防火墙/selinux/swap/hosts/内核参数）
2. 安装 Docker  → 三台节点，cgroup driver = systemd
3. 安装 K8s 组件 → kubeadm / kubelet / kubectl
4. 初始化 Master → kubeadm init
5. 加入 Worker   → kubeadm join
6. 部署 CNI      → Calico
7. 验证集群      → kubectl get nodes（全部 Ready）
8. 部署测试服务  → Nginx Deployment + NodePort
```

---

## 三、环境准备（Master / Node1 / Node2 均需执行）

### 3.1 关闭防火墙、SELinux，禁用 Swap

> 学习环境建议关闭；**生产环境**应通过规则/策略精细化配置，而非简单全关。

```bash
# 永久关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# 永久关闭 SELinux
sed -i 's/enforcing/disabled/g' /etc/selinux/config

# 永久禁用 swap（重启生效）
sed -ri 's/.*swap.*/#&/' /etc/fstab
swapoff -a

# 验证
systemctl status firewalld | grep active
cat /etc/selinux/config | grep ^SELINUX
cat /etc/fstab | grep swap
```

### 3.2 主机名、hosts、内核网络参数

**Master 节点：**

```bash
hostnamectl set-hostname k8s-master
```

**Node1 / Node2 分别执行：**

```bash
hostnamectl set-hostname k8s-node1   # 或 k8s-node2
```

**三台节点都添加 hosts（IP 按实际修改）：**

```bash
cat >> /etc/hosts << EOF
192.168.246.156 k8s-master
192.168.246.253 k8s-node1
192.168.246.254 k8s-node2
EOF
```

**桥接流量进入 iptables（CNI / Service 网络必需）：**

```bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```

**加载 br_netfilter 模块（如未加载）：**

```bash
modprobe br_netfilter
echo 'br_netfilter' >> /etc/modules-load.d/k8s.conf
```

> 建议完成后 **reboot** 重启，确保配置生效。

### 3.3 SSH 免密登录（可选，便于批量操作）

```bash
ssh-keygen -P '' -f ~/.ssh/id_rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node1
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node2
```

---

## 四、安装 Docker（三台节点）

### 4.1 安装 Docker 20.10.16

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum makecache

yum install -y \
  docker-ce-20.10.16-3.el8 \
  docker-ce-cli-20.10.16-3.el8 \
  docker-ce-rootless-extras-20.10.16-3.el8 \
  containerd.io \
  docker-compose-plugin

systemctl start docker && systemctl enable docker
docker --version
```

### 4.2 修改 cgroup 驱动为 systemd

K8s 1.23 + kubelet 要求 Docker 使用 **systemd** cgroup driver，不能用 cgroupfs。

```bash
docker info | grep -i cgroup

cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl daemon-reload && systemctl restart docker
docker info | grep -i cgroup   # 应显示 systemd
```

> **为什么必须一致？** kubelet 通过 CRI 读取 runtime 的 cgroup driver，并在 `pkg/kubelet/cm/` 用同一 driver 管理 Pod cgroup。详见 [kubelet cgroup 源码笔记](../04-node/kubelet/cgroup.md)。

---

## 五、安装 Kubernetes 组件（三台节点）

### 5.1 添加阿里云 K8s yum 源

```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 5.2 安装 kubeadm / kubelet / kubectl

```bash
# 查看可用版本
yum list --showduplicates kubeadm --disableexcludes=kubernetes

# 安装指定版本
yum install -y kubelet-1.23.14 kubeadm-1.23.14 kubectl-1.23.14

systemctl enable kubelet

kubectl version --client
kubeadm version
kubelet --version
```

| 组件 | 作用 |
|------|------|
| **kubeadm** | 集群引导：`kubeadm init` / `kubeadm join` |
| **kubelet** | 节点 Agent，负责 Pod 生命周期 |
| **kubectl** | 命令行管理工具 |

> 此时 kubelet 尚未加入集群，状态异常属正常，init/join 后会正常。

---

## 六、Master 节点初始化

**仅在 k8s-master 上执行：**

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.246.156 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.23.14 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.224.0.0/16
```

### 参数说明

| 参数 | 含义 |
|------|------|
| `--apiserver-advertise-address` | Master 节点 IP，APIServer 对外宣告地址 |
| `--image-repository` | 控制面组件镜像仓库（国内可用阿里云） |
| `--kubernetes-version` | K8s 版本 |
| `--service-cidr` | ClusterIP Service 网段 |
| `--pod-network-cidr` | Pod 网段（须与 Calico 配置一致） |

### 初始化成功后配置 kubectl

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
# 此时节点为 NotReady，安装 CNI 后变为 Ready
```

**务必保存 init 输出末尾的 join 命令**，类似：

```bash
kubeadm join 192.168.246.156:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 七、Worker 节点加入集群

**在 k8s-node1、k8s-node2 上分别执行 init 输出的 join 命令：**

```bash
kubeadm join 192.168.246.156:6443 \
  --token rjdydr.uu4afmz86mmgkglf \
  --discovery-token-ca-cert-hash sha256:da61eb878295e283571e268f9c81e68876cc1eadf26cb541dc7b089c350935fb
```

成功提示：`This node has joined the cluster`

**Master 上验证：**

```bash
kubectl get nodes
# NAME         STATUS     ROLES                  AGE   VERSION
# k8s-master   NotReady   control-plane,master   ...   v1.23.14
# k8s-node1    NotReady   <none>                 ...   v1.23.14
# k8s-node2    NotReady   <none>                 ...   v1.23.14
```

### Token 过期后重新生成

Bootstrap Token 默认 **24 小时**有效。

```bash
# 在 Master 上
kubeadm token list
kubeadm token create
kubeadm token list

# 获取 CA 证书 hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
# 使用时前面加 sha256:
```

---

## 八、部署 Calico 网络插件

**仅在 Master 上执行。** Pod 网段须与 init 时 `--pod-network-cidr` 一致。

```bash
curl https://docs.tigera.io/archive/v3.25/manifests/calico.yaml -O

# 修改 Pod CIDR（与 kubeadm init 的 10.224.0.0/16 对应）
# 参考笔记示例使用 10.234.0.0/16，请统一为同一网段，例如：
sed -i -e 's/#\s*- name/- name/g' \
  -e 's|#\s*value:\s*"192.168.0.0/16"|  value: "10.224.0.0/16"|g' calico.yaml

# 去掉 docker.io 前缀，加速拉镜像
sed -i 's#docker.io/##g' calico.yaml

kubectl apply -f calico.yaml
```

**验证 CNI：**

```bash
kubectl get po -n kube-system
# calico-node、calico-kube-controllers 等应为 Running

kubectl get nodes
# 所有节点 STATUS = Ready
```

---

## 九、部署测试服务（Nginx）

```bash
# 创建 Deployment
kubectl create deployment nginx --image=nginx

# 暴露为 NodePort
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看 Pod 与 Service
kubectl get pod,svc
```

输出示例：

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          1m

NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP  10.96.0.1       <none>        443/TCP        ...
service/nginx        NodePort   10.96.xxx.xxx   <none>        80:3xxxx/TCP   ...
```

浏览器访问：`http://<任意节点IP>:<NodePort>`，看到 Nginx 欢迎页即表示集群可用。

---

## 十、常用扩展操作

### 10.1 kubectl 命令自动补全

```bash
yum install -y bash-completion

# 当前用户
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# 全局
kubectl completion bash | tee /etc/bash_completion.d/kubectl > /dev/null
source /usr/share/bash-completion/bash_completion
```

### 10.2 在 Worker 节点使用 kubectl

```bash
# 从 Master 拷贝 kubeconfig
scp root@k8s-master:/etc/kubernetes/admin.conf /etc/kubernetes/

# 配置环境变量
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
source ~/.bashrc

kubectl get nodes
```

> 生产环境应使用 RBAC 限制 Worker 上的 kubectl 权限，而非直接分发 admin.conf。

### 10.3 将 Node 节点移出集群

```bash
kubectl get nodes
kubectl cordon k8s-node2
kubectl drain k8s-node2 --ignore-daemonsets
kubectl delete node k8s-node2
kubectl get nodes
```

**在 Node 本机执行清理（可选）：**

```bash
kubeadm reset
```

---

## 十一、常见问题排查

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `kubeadm init` 镜像拉取失败 | 网络 / 镜像源 | 使用 `--image-repository registry.aliyuncs.com/google_containers` |
| 节点长期 **NotReady** | CNI 未装或 Pod CIDR 不一致 | 检查 Calico Pod、`calico.yaml` 中 IP 池 |
| join 失败 **token 无效** | Token 过期（24h） | `kubeadm token create` + 重新获取 hash |
| Pod **Pending** | 调度 / 镜像 / 资源 | `kubectl describe pod` 看 Events |
| kubelet 启动失败 | swap 未关 / cgroup 驱动不对 | 检查 swap、Docker `systemd` driver |
| 多节点网络不通 | 防火墙 / br_netfilter / ip_forward | 检查 §3.1、§3.2 配置 |
| Nginx NodePort 无法访问 | 防火墙 / 未 Ready | 确认节点 Ready、安全组/防火墙放行 NodePort |

**常用诊断命令：**

```bash
kubectl get nodes -o wide
kubectl get po -A
kubectl describe node <node-name>
journalctl -xeu kubelet
systemctl status kubelet
kubeadm config images list --kubernetes-version v1.23.14
```

---

## 十二、与源码学习的关联

| 操作 | 对应组件 / 源码 |
|------|----------------|
| `kubeadm init` | `cmd/kubeadm/app/cmd/init/` — 生成静态 Pod 清单（apiserver、etcd、scheduler、controller-manager） |
| `kubeadm join` | `cmd/kubeadm/app/cmd/join/` — TLS Bootstrap、kubelet 注册 |
| `kubectl apply` | `cmd/kubectl/` + APIServer |
| Calico CNI | 集群外组件；kubelet 通过 CNI 接口调用 |
| 控制面静态 Pod | `/etc/kubernetes/manifests/` |

---

## 附录：一键脚本思路（可选）

参考笔记提供 Master / Node 一键脚本，核心即本文 **§三～§五** 的合并。可按节点角色拆分：

```
master-setup.sh   → §3 环境 + §4 Docker + §5 K8s + §6 init + §8 Calico
node-setup.sh     → §3 环境 + §4 Docker + §5 K8s + §7 join
```

---

## 参考链接

- [参考 CSDN 原文](https://blog.csdn.net/yu33575/article/details/135387548)
- [kubeadm 官方文档](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Calico 版本对照](https://docs.tigera.io/archive/v3.25/getting-started/kubernetes/requirements)
- [阿里云 Docker CE 镜像](https://developer.aliyun.com/mirror/docker-ce)
