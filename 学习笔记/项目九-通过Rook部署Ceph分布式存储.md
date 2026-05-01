# 项目九: 通过 Rook 部署 Ceph 分布式存储

## 项目概述
在 Kubernetes 集群中通过 Rook 部署 Ceph 分布式存储系统。实现 RBD 块存储、CephFS 文件存储、RGW 对象存储三种存储类型的统一管理，为 K8s 工作负载提供持久化存储。

## 架构设计
```
Kubernetes Cluster
    │
    ├── Rook Operator (管理 Ceph 生命周期)
    │
    └── Ceph Cluster (运行在 K8s 中)
        ├── Ceph MON (监控器) × 3
        ├── Ceph OSD (存储守护进程) × N
        ├── Ceph MGR (管理器)
        ├── Ceph MDS (元数据服务 - CephFS)
        └── Ceph RGW (对象网关)
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
      RBD    CephFS    RGW
    (块存储) (文件存储) (对象存储)
```

**核心组件：**
- **Rook**：云原生存储编排器，管理 Ceph 的生命周期
- **Ceph**：分布式存储系统
- **Ceph MON**：集群状态监控，提供认证和日志
- **Ceph OSD**：数据存储守护进程，管理物理磁盘
- **Ceph MDS**：CephFS 元数据服务
- **Ceph MGR**：集群管理仪表盘
- **Ceph RGW**：S3/Swift 兼容的对象存储网关

## 部署步骤

### 关键要点总结

1. **Rook 安装部署**
   - 使用 Helm 或 YAML 清单部署 Rook Operator
   - 创建 CephCluster CRD，指定 OSD 使用裸盘
   - 安装 snapshot 控制器（K8s 1.19+）

2. **RBD 块存储**
   - 创建 StorageClass：连接 Ceph 存储池
   - 动态 PV 申请：PVC 指定 StorageClass
   - StatefulSet 动态 PVC：volumeClaimTemplates
   - PVC 扩容：需要 K8s 1.16+，allowVolumeExpansion=true
   - PVC 快照：需要 K8s 1.17+，安装 snapshot 控制器

3. **CephFS 文件存储**
   - 创建共享文件系统（MDS + 数据池 + 元数据池）
   - 创建 StorageClass
   - 支持多 Pod 共享挂载
   - 外部访问：通过 CephFS 内核挂载或 ceph-fuse

4. **RGW 对象存储**
   - 部署 RGW 集群（CephObjectStore）
   - 支持 S3 和 Swift API
   - 高可用部署：多 RGW Pod + Service
   - 创建 Bucket 存储桶和用户

5. **Ceph Dashboard**
   - 通过 Rook 部署 Ceph Dashboard
   - 可视化监控集群状态、OSD、PG、存储池

## 核心知识点

### 1. Ceph 核心概念
- **RADOS**：可靠自治分布式对象存储，Ceph 的核心
- **PG（Placement Group）**：对象的逻辑组织单元
- **CRUSH 算法**：数据分布算法，计算数据存储位置
- **寻址流程**：File → Object → PG → OSD

### 2. Ceph 三种存储接口
- **RBD（块存储）**：类似虚拟磁盘，适合单 Pod 挂载
- **CephFS（文件存储）**：POSIX 兼容文件系统，适合多 Pod 共享
- **RGW（对象存储）**：S3/Swift 兼容，适合非结构化数据

### 3. CephX 认证机制
- MON 负责身份认证，分发 session key
- session key 通过客户端密钥加密
- MON 和 OSD 共享 secret，OSD 信任 MON 颁发的 ticket
- ticket 有有效期限制

### 4. CRUSH Map 定制
- 控制数据在 OSD 上的分布策略
- 支持故障域隔离：机架、机房、区域
- 可配置副本放置规则
- 命令行调整 crush map 实现自定义容灾策略

### 5. OSD 日常管理
- 添加 OSD：扩展存储容量
- 删除 OSD：缩容或替换
- OSD 替换：故障磁盘更换
- OSD 故障处理：自动检测和恢复

## 面试高频题

### Q1: Ceph 的 CRUSH 算法有什么优势？
**答：** CRUSH（Controlled Replication Under Scalable Hashing）是伪随机数据分布算法。优势：① 去中心化，客户端直接计算数据位置，无需查元数据服务器；② 支持故障域隔离（机架、机房感知）；③ 扩展时只需迁移少量数据；④ 数据分布均匀。

### Q2: Rook 与传统 Ceph 部署有什么区别？
**答：** 传统部署：在裸机上用 ceph-deploy/cephadm 部署，独立于 K8s。Rook 部署：将 Ceph 运行在 K8s 中，通过 Operator 模式管理。Rook 优势：与 K8s 深度集成、自动管理生命周期、通过 CRD 声明式管理、支持动态 PV。传统部署适合非 K8s 环境。

### Q3: Ceph 中 PG、OSD、Pool 的关系是什么？
**答：** Pool 是存储池，包含多个 PG。PG（Placement Group）是对象的逻辑分组，一个 Pool 的 PG 数量在创建时确定。PG 通过 CRUSH 映射到多个 OSD 上（副本数决定映射几个 OSD）。一个 OSD 承载多个 PG。三者关系：Pool → PG → OSD。

### Q4: RBD 块存储和 CephFS 文件存储的使用场景区别？
**答：** RBD：单 Pod 独占挂载，类似块设备，适合数据库等需要独占存储的场景。CephFS：多 Pod 共享挂载，POSIX 兼容，适合日志收集、共享配置等场景。RBD 性能更好，CephFS 支持多读多写。

## 学习心得
- Rook 是 K8s 中部署 Ceph 的最佳方式，Operator 模式大大简化了运维
- 理解 Ceph 寻址流程（File→Object→PG→OSD）是掌握 Ceph 的关键
- CRUSH Map 定制是高级运维技能，需要理解故障域和副本策略
- 三种存储接口（RBD/CephFS/RGW）覆盖了大部分存储需求
- PVC 扩容和快照功能需要特定 K8s 版本支持，部署前要确认版本兼容性
