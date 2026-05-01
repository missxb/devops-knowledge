# 项目八: 通过阿里云 ECS 部署高可用 K8s 集群

## 项目概述
在阿里云 ECS 环境中部署 Kubernetes 1.19.5 高可用集群。利用阿里云 SLB 实现 API Server 的负载均衡，部署完整的 Addons 组件，实现生产级 K8s 集群。

## 架构设计
```
              阿里云 SLB (内网四层代理)
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
     Master01   Master02   Master03
     [etcd]     [etcd]     [etcd]
     [apiserver][apiserver][apiserver]
     [cm/sched] [cm/sched] [cm/sched]
         │                    │
         ▼                    ▼
      Worker01             Worker02
      [kubelet]            [kubelet]
      [kube-proxy]         [kube-proxy]
```

**核心组件：**
- **阿里云 SLB**：四层负载均衡（替代自建 HAProxy+Keepalived）
- **etcd 集群**：3 节点
- **Kubernetes 1.19.5**：二进制部署
- **manager 节点**：专用管理节点，负责证书生成和配置分发

## 部署步骤

### 关键要点总结

1. **环境准备**
   - 阿里云 ECS 服务器配置
   - 内核参数优化、IPVS 模块加载
   - SSH 免密登录配置（manager → 所有节点）
   - cfssl 组件安装（证书生成）

2. **Docker 安装**
   - 安装 docker-ce 19.03
   - cgroupdriver=systemd（与 kubelet 一致）
   - 配置镜像加速器

3. **etcd 集群部署**
   - manager 节点生成证书，推送到 master 节点
   - 三节点 etcd 集群配置和启动

4. **Kubernetes 组件部署**
   - kube-apiserver：配置 Service CIDR、etcd 地址
   - HAProxy：配置四层代理 apiserver 8443 端口
   - 阿里云 SLB：内网 IP 代理 HAProxy 的 8443 端口
   - kube-controller-manager、kube-scheduler
   - kubectl 安装和 admin.kubeconfig 配置
   - kubelet bootstrap 配置
   - kube-proxy 配置（IPVS 模式）

5. **Addons 安装**
   - Calico 网络插件
   - CoreDNS
   - Metrics Server
   - Dashboard
   - Ingress-Nginx

## 核心知识点

### 1. 阿里云 SLB 替代 Keepalived
- 传统方案：HAProxy + Keepalived（VIP 漂移）
- 云上方案：阿里云 SLB 四层代理（更稳定、更简单）
- SLB 提供健康检查、会话保持、跨可用区高可用

### 2. manager 节点设计模式
- 专用管理节点不运行 K8s 组件
- 负责证书生成、配置分发、集群管理
- 降低 master 节点的管理复杂度

### 3. Bootstrap Token 认证
- kubelet 首次启动使用 bootstrap token
- 自动向 apiserver 发起 CSR
- controller-manager 自动审批颁发证书
- 实现节点自动加入集群

### 4. 云上 K8s 集群的特殊考虑
- 安全组配置（端口开放）
- SLB 健康检查配置
- 内网通信优化
- 弹性伸缩能力

## 面试高频题

### Q1: 云上部署 K8s 与自建 IDC 有什么区别？
**答：** ① 负载均衡：云上用 SLB 替代 HAProxy+Keepalived，更稳定；② 网络：云上 VPC 网络与 K8s 网络需要规划 CIDR 避免冲突；③ 存储：可以使用云盘 CSI 驱动；④ 弹性：云上可以利用 Auto Scaling 实现节点弹性。

### Q2: 为什么建议单独设置 manager 节点？
**答：** ① 职责分离：manager 负责集群管理（证书生成、配置分发），master 负责运行 K8s 控制组件；② 安全性：manager 可以配置更严格的访问控制；③ 可维护性：不影响业务的情况下进行管理操作。

### Q3: Bootstrap Token 的安全性如何保障？
**答：** ① Token 有 TTL 有效期（默认 24 小时）；② Token 绑定特定的组（system:bootstrappers）；③ RBAC 限制 bootstrap 用户只能申请证书；④ 证书颁发后 Token 即可失效。

## 学习心得
- 阿里云 SLB 是云上 K8s 高可用的首选方案，比自建 Keepalived 更稳定
- manager 节点的职责分离模式值得在生产中推广
- 二进制安装的经验在云上部署同样适用
- 安全组规则配置是云上部署最容易忽略的环节
