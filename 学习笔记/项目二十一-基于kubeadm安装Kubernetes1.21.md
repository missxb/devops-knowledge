# 项目二十一: 基于 kubeadm 安装 Kubernetes 1.21

## 项目概述
使用 kubeadm 工具部署 Kubernetes 1.21 集群，包括环境准备、集群初始化、网络插件、常用 Addons 部署（Metrics Server、Dashboard、Ingress-Nginx、Rancher、Longhorn 存储）。

## 架构设计
```
master01 (172.17.94.63)
  ├── kubeadm init
  ├── kube-apiserver
  ├── kube-controller-manager
  ├── kube-scheduler
  └── etcd
         │
    ┌────┴────┐
    ▼         ▼
worker01   worker02
(.94.62)   (.94.61)
[kubelet]  [kubelet]
[kube-proxy][kube-proxy]
[Docker]   [Docker]
```

**核心组件：**
- **kubeadm**：K8s 集群部署工具
- **kubelet**：节点代理
- **kubectl**：命令行工具
- **Docker CE 19.03**：容器运行时
- **Calico**：网络插件
- **Longhorn**：分布式块存储

## 部署步骤

### 关键要点总结

1. **环境准备（所有节点）**
   - 配置 /etc/hosts（使用标准 DNS 命名，不能用 localhost）
   - 禁用防火墙和 SELinux
   - 内核参数优化（ip_forward、bridge-nf-call-iptables 等）
   - 加载 IPVS 模块（ip_vs、ip_vs_rr、ip_vs_wrr、ip_vs_sh）
   - 时间同步（chrony）
   - 关闭 swap 分区
   - 内核升级（5.13+）

2. **安装 Docker**
   - 安装 docker-ce 19.03
   - 配置 cgroupdriver=systemd（与 kubelet 一致）
   - 配置镜像加速器

3. **安装 kubeadm/kubelet/kubectl**
   - 配置 K8s YUM 源
   - 安装 kubeadm、kubelet、kubectl
   - 设置 kubelet 开机自启

4. **初始化集群（Master 节点）**
   ```bash
   kubeadm init \
     --kubernetes-version=v1.21.0 \
     --pod-network-cidr=10.244.0.0/16 \
     --service-cidr=10.96.0.0/12 \
     --apiserver-advertise-address=<master-ip>
   ```

5. **Worker 节点加入**
   ```bash
   kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

6. **安装网络插件（Calico）**
   ```bash
   kubectl apply -f calico.yaml
   ```

7. **安装 Addons**
   - **Metrics Server**：资源监控
   - **Dashboard**：Web UI
   - **Ingress-Nginx**：七层入口
   - **Rancher**：集群管理平台
   - **Longhorn**：分布式块存储

## 核心知识点

### 1. kubeadm 工作原理
- **kubeadm init**：初始化 Master 节点
  - 生成证书和 kubeconfig
  - 启动 etcd、apiserver、controller-manager、scheduler
  - 配置 RBAC 和 Bootstrap Token
- **kubeadm join**：Worker 节点加入集群
  - 使用 Bootstrap Token 认证
  - 自动申请和颁发 kubelet 证书

### 2. cgroupdriver 一致性
- Docker 和 kubelet 必须使用相同的 cgroupdriver
- 推荐都使用 systemd（Docker 默认 cgroupfs）
- 不一致会导致 kubelet 启动失败

### 3. Longhorn 分布式存储
- Rancher Labs 开源的轻量级分布式块存储
- 每个卷创建专用存储控制器
- 跨节点同步复制副本
- 使用 Kubernetes 进行编排
- 前提：所有节点安装 open-iscsi

### 4. kubeadm 证书管理
- 证书默认存储在 /etc/kubernetes/pki/
- 有效期：CA 证书 10 年，其他证书 1 年
- 证书续签：`kubeadm certs renew all`
- 查看证书：`kubeadm certs check-expiration`

### 5. kubeadm 与二进制安装对比
| 维度 | kubeadm | 二进制 |
|------|---------|--------|
| 部署速度 | 快（分钟级） | 慢（小时级） |
| 灵活性 | 一般 | 高 |
| 学习价值 | 了解流程 | 深入理解 |
| 生产适用 | 推荐 | 需要更多维护 |
| 升级 | 简单 | 复杂 |

## 面试高频题

### Q1: kubeadm init 的主要步骤是什么？
**答：** ① 预检检查（preflight）：验证系统环境；② 生成证书和 kubeconfig；③ 启动 etcd；④ 启动 kube-apiserver；⑤ 启动 kube-controller-manager 和 kube-scheduler；⑥ 生成 bootstrap token；⑦ 配置 RBAC；⑧ 部署 CoreDNS 和 kube-proxy。

### Q2: 为什么 kubeadm 要求 hostname 不能是 localhost？
**答：** Kubernetes 中所有 API 对象（包括 Node）的名称必须符合 RFC 1123 DNS 命名规范。localhost 不是有效的 DNS 名称，会导致 etcd 存储冲突、证书验证失败、节点注册异常等一系列问题。使用 `hostnamectl set-hostname` 设置标准主机名。

### Q3: Longhorn 与 Ceph 的区别？
**答：** Longhorn：轻量级，专注块存储，K8s 原生集成，部署简单，适合中小规模。Ceph：重量级，支持块/文件/对象三种存储，功能全面，适合大规模生产环境。Longhorn 适合 K8s 环境，Ceph 适合传统和 K8s 混合环境。

### Q4: kubeadm 证书过期如何处理？
**答：** ① 查看过期时间：`kubeadm certs check-expiration`；② 续签所有证书：`kubeadm certs renew all`；③ 重启控制面组件：`systemctl restart kubelet`；④ 更新 kubeconfig：`kubeadm kubeconfig user --client-name=admin`。注意：CA 证书不能通过 renew 续签。

## 学习心得
- kubeadm 是生产环境推荐的 K8s 部署工具，比二进制安装简单很多
- cgroupdriver 一致性是最容易踩的坑，Docker 和 kubelet 必须匹配
- 内核版本对 K8s 集群稳定性影响很大，建议升级到 5.x
- Longhorn 是 K8s 中最简单的分布式存储方案，适合快速上手
- 证书管理是 K8s 运维的重要环节，需要定期检查和续签
- Rancher 是 K8s 集群管理的强大工具，可以大大简化多集群管理
