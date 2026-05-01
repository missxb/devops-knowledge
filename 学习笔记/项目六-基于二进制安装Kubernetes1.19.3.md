# 项目六: 基于二进制安装 Kubernetes 1.19.3

## 项目概述
通过二进制方式手动部署 Kubernetes 1.19.3 高可用集群。包含 3 个 Master 节点、2 个 Worker 节点、2 个 HAProxy 负载均衡节点。手动安装每个组件，深入理解 K8s 各组件的作用和协作方式。

## 架构设计
```
                 ┌─────────────────┐
                 │   Keepalived    │
                 │   VIP: .31.99   │
                 └──┬──────────┬───┘
                    │          │
              HAProxy01    HAProxy02
              (.31.92)     (.31.93)
                    │          │
         ┌──────────┼──────────┼──────────┐
         ▼          ▼          ▼          ▼
     Master01   Master02   Master03
     (.31.59)   (.31.60)   (.31.61)
     [etcd]     [etcd]     [etcd]
     [apiserver][apiserver][apiserver]
     [cm]       [cm]       [cm]
     [scheduler][scheduler][scheduler]
         │                    │
         ▼                    ▼
      Worker01             Worker02
      (.31.62)             (.31.63)
      [kubelet]            [kubelet]
      [kube-proxy]         [kube-proxy]
      [Docker]             [Docker]
```

**核心组件：**
- **etcd**：分布式键值存储，K8s 数据存储层
- **kube-apiserver**：API 网关，所有操作的唯一入口
- **kube-controller-manager**：控制器管理器，资源状态协调
- **kube-scheduler**：Pod 调度器
- **kubelet**：节点代理，管理 Pod 生命周期
- **kube-proxy**：网络代理，实现 Service 负载均衡
- **HAProxy + Keepalived**：API Server 高可用入口
- **Calico**：网络插件（CNI）
- **CoreDNS**：集群 DNS 服务

## 部署步骤

### 关键要点总结

1. **环境准备**
   - 主机名和 hosts 配置
   - 关闭防火墙、SELinux、Swap
   - 内核参数优化（ip_forward、bridge-nf-call-iptables 等）
   - 配置 Limit（65535）
   - 时间同步（chrony/ntpdate）
   - 开启 IPVS 模块

2. **安装 Docker**
   - 安装 docker-ce 19.03
   - 配置 cgroupdriver=systemd（与 kubelet 一致）
   - 配置镜像加速和数据目录

3. **部署 etcd 集群**
   - 使用 CFSSL 生成 TLS 证书（CA、Server、Peer 证书）
   - 三节点 etcd 集群配置
   - 验证：`etcdctl endpoint health`

4. **部署 HAProxy + Keepalived**
   - HAProxy 四层代理 kube-apiserver 的 8443 端口
   - Keepalived VIP 漂移实现高可用

5. **部署 Kubernetes 组件**
   - kube-apiserver：配置 service-cluster-ip-range、etcd-servers、authorization-mode=Node,RBAC
   - kube-controller-manager：配置 cluster-cidr、service-cluster-ip-range
   - kube-scheduler：配置 leader-elect
   - TLS Bootstrapping：自动颁发 kubelet 证书

6. **部署 Worker 组件**
   - kubelet：配置 bootstrap-kubeconfig，自动申请证书
   - kube-proxy：配置 iptables 或 ipvs 模式

7. **安装 Addons**
   - Calico 网络插件（CIDR: 10.244.0.0/16）
   - CoreDNS（默认 2 副本）
   - Metrics Server
   - Ingress-Nginx
   - Dashboard 2.0

## 核心知识点

### 1. etcd 证书体系
- CA 证书：根证书，签名所有其他证书
- Server 证书：etcd 对外服务使用
- Peer 证书：etcd 节点间通信使用
- 使用 CFSSL 工具链简化证书管理

### 2. kube-apiserver 关键参数
- `--etcd-servers`：etcd 集群地址
- `--service-cluster-ip-range`：Service CIDR（如 10.96.0.0/12）
- `--authorization-mode=Node,RBAC`：授权模式
- `--enable-admission-plugins`：准入控制器

### 3. TLS Bootstrapping
- kubelet 首次启动使用 bootstrap token 认证
- 自动向 apiserver 申请证书
- controller-manager 自动审批和颁发证书
- 实现 kubelet 证书的自动化管理

### 4. 二进制安装 vs kubeadm
- 二进制：手动安装每个组件，深入理解架构，灵活可控
- kubeadm：自动化工具，快速部署，但不够灵活
- 生产环境推荐 kubeadm 或专用部署工具（rancher、kubespray）

## 面试高频题

### Q1: Kubernetes 二进制安装需要生成哪些证书？
**答：** ① etcd CA + Server/Peer 证书；② Kubernetes CA + apiserver 证书；③ controller-manager kubeconfig；④ scheduler kubeconfig；⑤ admin kubeconfig；⑥ kubelet bootstrap token；⑦ ServiceAccount 密钥对。证书体系是安全通信的基础。

### Q2: kube-apiserver 的 --authorization-mode=Node,RBAC 是什么意思？
**答：** Node 模式授权 kubelet 访问本节点相关的 API 资源。RBAC 基于角色的访问控制，通过 Role/ClusterRole 定义权限，RoleBinding/ClusterRoleBinding 绑定到用户或服务账户。两者组合实现细粒度的权限控制。

### Q3: 为什么 kube-proxy 推荐使用 IPVS 模式？
**答：** IPVS 使用哈希表（O(1) 时间复杂度）替代 iptables 的链表（O(n)），大规模 Service 场景下性能更好。IPVS 支持更多负载均衡算法（rr、lc、sh 等），并支持会话保持。Service 数量多时，iptables 规则刷新非常慢，IPVS 优势明显。

### Q4: TLS Bootstrapping 的工作流程是什么？
**答：** ① kubelet 使用 bootstrap token 创建 bootstrap-kubeconfig；② kubelet 向 apiserver 发起 CSR（证书签名请求）；③ controller-manager 中的 BootstrapSigner 自动审批；④ apiserver 颁发证书给 kubelet；⑤ kubelet 使用新证书后续认证。

## 学习心得
- 二进制安装 K8s 是理解集群架构的最佳方式，建议每个运维都做一遍
- 证书管理是二进制安装中最复杂的部分，CFSSL 工具链可以简化
- cgroupdriver 一致性很重要：docker 和 kubelet 必须都用 systemd
- 内核参数优化是 K8s 集群稳定的前提
- 生产环境建议使用 kubeadm 或 Rancher 等工具，但二进制安装的经验对排错很有帮助
