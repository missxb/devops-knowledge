# 项目十七: GlusterFS 存储

## 项目概述
部署 GlusterFS 分布式文件系统集群，实现跨节点的分布式存储，支持多种卷类型（分布式卷、复制卷、条带卷等）。

## 架构设计
```
GlusterFS Cluster (3节点)
  ├── node01 (192.168.10.90) ─ /brick1, /brick2
  ├── node02 (192.168.10.91) ─ /brick1, /brick2
  └── node03 (192.168.10.92) ─ /brick1, /brick2
         │
    Client (glusterfs-fuse 挂载)
```

**核心组件：**
- **GlusterFS Server**：存储服务端（glusterfs-server、glusterfs-cli）
- **GlusterFS Client**：客户端挂载（glusterfs-fuse）
- **Brick**：底层存储单元（XFS 格式的独立分区）
- **Volume**：逻辑存储卷（由多个 Brick 组成）

## 部署步骤

### 关键要点总结

1. **基础环境配置**
   - 主机名和 hosts 配置
   - 关闭防火墙和 SELinux
   - 时间同步

2. **安装 GlusterFS**
   - 服务端：`yum install glusterfs-server glusterfs glusterfs-cli glusterfs-fuse`
   - 客户端：`yum install glusterfs glusterfs-fuse`

3. **准备存储磁盘**
   - 独立磁盘格式化为 XFS：`mkfs.xfs /dev/sdc1`
   - 挂载到 /brick1、/brick2
   - 配置 /etc/fstab 开机自动挂载

4. **组建集群**
   - 启动服务：`systemctl start glusterd`
   - 添加节点：`gluster peer probe node02`
   - 查看状态：`gluster peer status`

5. **创建卷**
   - 分布式卷：`gluster volume create testvol node01:/brick1/br1 node02:/brick1/br1`
   - 启动卷：`gluster volume start testvol`

## 核心知识点

### 1. GlusterFS 卷类型
- **分布式卷（Distribute）**：文件随机分布到各 Brick，适合大量小文件，不提供冗余
- **复制卷（Replicate）**：文件在多个 Brick 间同步复制，提供数据冗余
- **条带卷（Stripe）**：文件分成块分布到各 Brick，适合大文件
- **分布式复制卷**：分布式 + 复制的组合，生产环境最常用

### 2. Brick 和 Volume 的关系
- Brick 是底层存储单元（一个目录或挂载点）
- Volume 由多个 Brick 组成
- 一个节点可以有多个 Brick
- Brick 使用 XFS 文件系统（推荐）

### 3. GlusterFS 特点
- 无中心架构，对等节点
- 支持 POSIX 兼容的文件系统接口
- 支持 NFS、CIFS、FUSE 多种访问方式
- 弹性哈希算法（EHA）实现数据分布
- 支持在线扩展和收缩

### 4. 硬盘故障处理
- 监控 Brick 健康状态
- 故障 Brick 替换流程
- 数据自愈（复制卷模式下）

## 面试高频题

### Q1: GlusterFS 和 Ceph 的区别？
**答：** GlusterFS：文件存储，无中心架构，POSIX 兼容，适合文件共享场景。Ceph：统一存储（块/文件/对象），有 MON 中心节点，CRUSH 算法，适合 K8s 和云环境。GlusterFS 更简单，Ceph 功能更全面。

### Q2: GlusterFS 的分布式卷和复制卷的区别？
**答：** 分布式卷：文件随机分布到各 Brick，无冗余，一个 Brick 故障则该 Brick 上的数据丢失。复制卷：文件在多个 Brick 间同步复制，提供数据冗余，一个 Brick 故障不影响数据访问。生产环境建议使用分布式复制卷。

### Q3: GlusterFS 如何实现弹性扩展？
**答：** 添加新节点：`gluster peer probe new-node`。扩展卷：`gluster volume add-brick volname new-node:/brick`。GlusterFS 使用弹性哈希算法，扩展时只需迁移部分数据，无需全量重分布。

## 学习心得
- GlusterFS 部署简单，适合中小规模文件共享场景
- Brick 使用 XFS 格式是官方推荐，性能和稳定性最好
- 分布式复制卷是生产环境的首选，兼顾性能和可靠性
- GlusterFS 在 K8s 中可以通过 Heketi 或 GlusterFS CSI 集成
- 与 Ceph 相比，GlusterFS 更轻量但功能较少
