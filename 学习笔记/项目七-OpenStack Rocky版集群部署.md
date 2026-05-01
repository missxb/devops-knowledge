# 项目七: OpenStack Rocky 版集群部署

## 项目概述
部署 OpenStack Rocky 版私有云集群，包含控制节点、计算节点、Ceph 存储集群、MySQL Galera Cluster、RabbitMQ 集群等完整架构。实现企业级私有云平台的搭建。

## 架构设计
```
HAProxy01 + HAProxy02 (Keepalived VIP)
         │
┌────────┼────────┐
▼        ▼        ▼
Controller01  Controller02  Controller03
[Keystone]    [Keystone]    [Keystone]
[Glance]      [Glance]      [Glance]
[Nova-API]    [Nova-API]    [Nova-API]
[Neutron]     [Neutron]     [Neutron]
[Horizon]     [Horizon]     [Horizon]
[Cinder]      [Cinder]      [Cinder]
[RabbitMQ]    [RabbitMQ]    [RabbitMQ]
[MySQL]       [MySQL]       [MySQL]
         │
┌────────┼────────┐
▼        ▼        ▼
Computer01  Computer02   Ceph Cluster
[Nova]      [Nova]       (3节点)
[Neutron]   [Neutron]    [Ceph OSD]
[Docker]    [Docker]     [Ceph MON]
```

**核心组件：**
- **Keystone**：身份认证服务
- **Glance**：镜像服务
- **Nova**：计算服务
- **Neutron**：网络服务（OpenVSwitch + VXLAN）
- **Horizon**：Web 管理界面
- **Cinder**：块存储服务
- **Ceph**：分布式存储后端
- **MySQL Galera Cluster**：数据库高可用
- **RabbitMQ**：消息队列集群
- **HAProxy + Keepalived**：负载均衡

## 部署步骤

### 关键要点总结

1. **基础环境**
   - CentOS 7.8，内核 5.9.3
   - Chrony 时间同步，HAProxy01 为服务端
   - 安装 OpenStack Rocky YUM 源

2. **Ceph 存储集群**
   - 使用 ceph-deploy 工具部署
   - SSH 免密配置，cephadmin 用户
   - 创建存储池：images（Glance）、volumes（Cinder）、vms（Nova）

3. **MySQL Galera Cluster**
   - 三节点多主架构
   - 注意：生产环境建议使用单节点 MySQL（Galera 在 OpenStack 中有兼容问题）

4. **RabbitMQ 镜像队列集群**
   - 三节点镜像队列模式
   - 所有队列自动在节点间同步

5. **HAProxy + Keepalived**
   - 四层代理所有 OpenStack API 服务
   - VIP 实现控制节点高可用

6. **OpenStack 核心服务部署**
   - Keystone → Glance → Nova → Neutron → Horizon → Cinder
   - 每个服务都需要：创建数据库、注册 Keystone、安装配置、初始化数据库、启动服务

7. **Neutron 网络配置**
   - OpenVSwitch + VXLAN 实现
   - ML2 插件配置
   - Linux Bridge Agent、L3 Agent、DHCP Agent

## 核心知识点

### 1. OpenStack 核心服务关系
- **Keystone**：认证中心，所有服务都需要向 Keystone 注册
- **Nova**：计算引擎，管理虚拟机生命周期
- **Neutron**：网络服务，提供虚拟网络、子网、路由、安全组
- **Glance**：镜像服务，存储和管理 VM 镜像
- **Cinder**：块存储，为 VM 提供持久化卷
- **Horizon**：Web UI，统一管理入口

### 2. VXLAN 网络虚拟化
- VXLAN 在三层网络上构建二层 overlay 网络
- 使用 24 位 VNI（VXLAN Network Identifier），支持 16M 个隔离网络
- MAC-in-UDP 封装，VTEP 负责封装/解封装
- 解决了传统 VLAN 只有 4096 个的限制

### 3. Ceph 与 OpenStack 集成
- Glance 使用 Ceph RBD 存储镜像
- Cinder 使用 Ceph RBD 作为后端存储
- Nova 使用 Ceph RBD 存储 VM 磁盘
- 需要配置 Ceph 认证（cephx）和密钥分发

### 4. Galera Cluster 特点
- 多主架构，所有节点可读写
- 基于同步复制，保证数据一致性
- 与 OpenStack 存在兼容性问题，生产环境需谨慎

## 面试高频题

### Q1: OpenStack 各核心服务的作用是什么？
**答：** Keystone（认证）、Nova（计算）、Neutron（网络）、Glance（镜像）、Cinder（块存储）、Horizon（Web UI）。部署顺序：Keystone → Glance → Nova → Neutron → Cinder → Horizon，因为后续服务依赖前序服务的认证和 API。

### Q2: VXLAN 如何解决传统 VLAN 的限制？
**答：** VLAN ID 只有 12 位（4096 个），无法满足公有云多租户需求。VXLAN 使用 24 位 VNI（16M 个），通过 MAC-in-UDP 封装在三层网络上构建二层 overlay 网络，支持虚拟机跨数据中心迁移，IP 和 MAC 地址不变。

### Q3: 为什么 OpenStack 中 Neutron 最复杂？
**答：** Neutron 涉及多个组件：ML2 插件（网络类型管理）、Linux Bridge/OVS Agent（二层网络）、L3 Agent（路由和 NAT）、DHCP Agent（IP 分配）、Metadata Agent（元数据服务）。还需要配置 Provider Network、Tenant Network、VXLAN/VLAN 等多种网络模型。

### Q4: Ceph 在 OpenStack 中的角色是什么？
**答：** Ceph 作为统一存储后端，为 Glance（镜像存储）、Cinder（块存储）、Nova（VM 磁盘）提供 RBD 块存储服务。优势：统一管理、高性能、高可用、支持快照和克隆、COW（Copy-On-Write）快速创建 VM。

## 学习心得
- OpenStack 是一个庞大的系统，部署顺序很重要，必须按依赖关系依次部署
- Neutron 网络是最复杂的部分，理解 VXLAN 原理和 ML2 架构是关键
- Ceph 与 OpenStack 的集成是生产环境的标准方案
- Galera Cluster 在 OpenStack 中有兼容问题，MySQL 单节点 + 主从复制可能更稳定
- HAProxy + Keepalived 是 OpenStack 控制节点高可用的标准方案
