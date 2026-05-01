# 项目十: Kubernetes 云架构平台

## 项目概述
Kubernetes 综合实战项目，涵盖 K8s 核心概念、Pod 管理、控制器、Service、Ingress、存储、配置管理、认证授权、网络模型、资源调度、HPA 自动伸缩、Helm 包管理等全方位知识体系。

## 核心知识点

### 1. Kubernetes 核心资源对象
- **Master**：kube-apiserver（API 入口）、kube-controller-manager（控制器中心）、kube-scheduler（调度器）、etcd（数据存储）
- **Node**：kubelet（容器管理）、kube-proxy（网络代理）、Docker Engine（容器运行时）
- **Pod**：最小调度单元，包含 Pause 容器（Infra 容器）+ 用户容器，共享 Network Namespace 和 Volume
- **Service**：微服务抽象，提供负载均衡和服务发现

### 2. Pod 生命周期管理
- **阶段**：Pending → Running → Succeeded/Failed/Unknown
- **容器状态**：Waiting → Running → Terminated
- **探针**：
  - livenessProbe（存活性）：失败则重启容器
  - readinessProbe（就绪性）：失败则从 Endpoint 移除
  - startupProbe（启动探测）：防止慢启动容器被误杀

### 3. 控制器类型
- **Deployment**：无状态应用管理，支持滚动更新、回滚、金丝雀发布
- **StatefulSet**：有状态应用管理，稳定网络标识和持久存储
- **DaemonSet**：每节点运行一个 Pod（如日志采集、监控 Agent）
- **Job/CronJob**：一次性/定时任务
- **HPA**：基于 CPU/内存指标自动水平伸缩

### 4. Service 四种类型
- **ClusterIP**：集群内部访问（默认）
- **NodePort**：通过节点端口暴露给外部
- **LoadBalancer**：云厂商负载均衡器
- **ExternalName**：CNAME 映射到外部域名
- **Headless Service**：无 ClusterIP，直接返回 Pod IP（配合 StatefulSet）

### 5. kube-proxy 代理模式
- **iptables 模式**：基于 netfilter，链表查找 O(n)，大规模性能差
- **IPVS 模式**：基于哈希表 O(1)，支持多种负载均衡算法（rr、lc、dh、sh），推荐生产使用

### 6. Ingress 七层代理
- Ingress Controller：Nginx、Traefik、HAProxy 等七层代理
- Ingress 资源：定义域名 → Service 的路由规则
- 功能：域名路由、TLS 终止、重写、灰度发布、会话粘性、CertManager 自动 HTTPS

### 7. 存储体系
- **Volume**：emptyDir（临时）、hostPath（主机目录）、NFS
- **PV/PVC**：持久卷和持久卷声明，解耦存储提供者和使用者
- **StorageClass**：动态 PV 创建，按需自动分配存储
- **Ceph 集成**：RBD 块存储、CephFS 文件存储
- **GlusterFS 集成**：通过 Heketi REST API 管理

### 8. 配置管理
- **ConfigMap**：配置数据键值对，支持环境变量、Volume 挂载、envFrom 注入
- **Secret**：敏感信息（密码、Token、证书），Base64 编码存储
- **Downward API**：将 Pod 信息注入容器（环境变量或 Volume）

### 9. 认证授权
- **认证**：Token、客户端证书、ServiceAccount
- **RBAC**：Role/ClusterRole（权限定义）+ RoleBinding/ClusterRoleBinding（绑定关系）
- **kubeconfig**：集群、用户、命名空间、认证信息的组织文件

### 10. 网络模型
- **VXLAN**：大二层网络虚拟化，MAC-in-UDP 封装，24 位 VNI
- **Flannel**：Overlay 网络，VXLAN 模式
- **Calico**：BGP 路由模式，支持 Network Policy

### 11. 资源调度
- **NodeSelector**：节点标签选择
- **NodeAffinity**：增强版节点亲和性
- **PodAffinity/PodAntiAffinity**：Pod 间亲和/反亲和
- **Taint/Toleration**：污点和容忍度
- **Priority/Preemption**：优先级和抢占调度

### 12. 资源管理
- **ResourceQuota**：命名空间级别资源配额
- **LimitRange**：Pod 默认资源限制
- **QoS**：Guaranteed > Burstable > BestEffort（OOM 优先级）

### 13. Helm 包管理
- Chart：应用包定义
- Repository：Chart 仓库
- Release：Chart 的运行实例
- 支持模板函数、管道、控制流程（if/with/range）

## 面试高频题

### Q1: Pod 为什么需要 Infra 容器？
**答：** Infra 容器（Pause 容器）是 Pod 中第一个启动的容器，为 Pod 分配 Network Namespace。用户容器通过 Join Network Namespace 共享网络，通过挂载 Infra 容器的 Volume 共享存储。好处：① 容器间可以直接用 localhost 通信；② 一个 Pod 只有一个 IP；③ Pod 生命周期与 Infra 容器一致，与用户容器无关。

### Q2: Deployment 滚动更新的参数如何配置？
**答：** `strategy.rollingUpdate.maxSurge`：更新过程中最多多出的 Pod 数（如 1）；`maxUnavailable`：更新过程中最多不可用的 Pod 数（如 0）。两者不可同时为 0。`minReadySeconds`：Pod 就绪后等待多久才继续更新。示例：maxSurge=1, maxUnavailable=0 表示先启新 Pod，再删旧 Pod，保证零停机。

### Q3: iptables 和 IPVS 模式的区别？
**答：** iptables：链表结构 O(n)，规则数随 Service 线性增长，大规模性能差。IPVS：哈希表 O(1)，支持更多负载均衡算法（rr、lc、sh 等），支持会话保持，大规模场景推荐。IPVS 仍使用 iptables 处理 NodePort 和 SNAT。

### Q4: StatefulSet 与 Deployment 的区别？
**答：** ① 网络标识：StatefulSet 的 Pod 有稳定 DNS 名（pod-name.service-name.namespace.svc.cluster.local）；② 存储：StatefulSet 通过 volumeClaimTemplates 为每个 Pod 创建独立 PVC；③ 部署顺序：StatefulSet 按序号顺序创建/删除；④ 适用场景：数据库、消息队列等有状态应用。

## 学习心得
- 这是一个非常全面的 K8s 综合项目，几乎涵盖了 K8s 的所有核心概念
- Pod 的设计模式（Infra 容器、Sidecar）是理解 K8s 架构的基础
- Service 的 iptables/IPVS 实现原理是网络排错的关键
- Ingress 是生产环境暴露服务的标准方案，需要掌握 Nginx Ingress 的高级配置
- RBAC 安全模型是多租户 K8s 集群的必备知识
- 资源管理（Quota/Limit/QoS）是集群稳定运行的保障
