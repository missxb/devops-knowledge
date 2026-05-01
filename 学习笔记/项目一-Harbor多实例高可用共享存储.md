# 项目一: Harbor 多实例高可用共享存储

## 项目概述
部署 Harbor 企业级容器镜像仓库的高可用集群方案。通过 NFS 共享存储实现多实例数据共享，配合 Keepalived 或 SLB/Nginx 实现负载均衡和故障切换，保障镜像仓库的持续可用。

## 架构设计
```
外部访问 → SLB/Nginx 或 Keepalived (VIP)
              ├── Harbor01 (主)
              └── Harbor02 (备)
                    ↓
              NFS 共享存储 (镜像数据 + 数据库数据)
                    ↓
         ┌──────────┴──────────┐
      PostgreSQL (外部DB)    Redis (外部缓存)
```

**核心组件：**
- **Harbor v2.2.1**：容器镜像仓库（双实例）
- **NFS**：共享存储层，存储镜像数据
- **PostgreSQL**：外部数据库，存储 Harbor 元数据
- **Redis**：外部缓存，提升访问性能
- **Keepalived/SLB**：高可用入口，VIP 漂移或四层代理
- **自签证书**：HTTPS 访问保障

## 部署步骤

### 关键要点总结

1. **基础环境准备**
   - 安装 Docker CE 19.03.9 和 docker-compose 1.28.4
   - 配置 daemon.json：镜像加速、insecure-registries、数据目录等

2. **NFS 共享存储配置**
   - NFS 服务端：`/data/harbor` 导出给 192.168.10.0/24 网段
   - 所有 Harbor 节点 + DB/Redis 节点挂载到 `/data/harbor`
   - 配置 `/etc/fstab` 实现开机自动挂载

3. **外部数据库和缓存部署**
   - PostgreSQL：创建 harbor、notarysigner、notaryserver 三个数据库，授权 harbor 用户
   - Redis：配置 bind 0.0.0.0、关闭 protected-mode、开启 AOF 持久化

4. **自签证书生成**
   - 使用 OpenSSL 生成 CA 证书、服务端证书（4096 位 RSA）
   - 包含 SAN（Subject Alternative Name）扩展
   - 证书安装到系统信任锚点

5. **Harbor 安装配置**
   - 配置 harbor.yml：hostname、HTTPS 证书路径、data_volume 指向 NFS 挂载点
   - 使用 `./prepare` 和 `./install.sh` 初始化（支持 Notary、Trivy、Chartmuseum）
   - 先单节点启动验证，再扩展到多节点

6. **高可用配置**
   - 方案一：SLB 或 Nginx 反向代理（推荐，简单易维护）
   - 方案二：两台 Harbor 上安装 Keepalived 实现 VIP 漂移

## 核心知识点

### 1. Harbor 架构理解
- Harbor 由 Core、Registry、Database、Redis、JobService 等组件构成
- 多实例共享同一个 PostgreSQL 和 Redis 即可实现数据一致性
- 镜像数据通过 NFS 共享，保证多实例看到相同的数据

### 2. NFS 共享存储
- NFS 是网络文件系统，适合中小规模共享存储场景
- 关键参数：`rw,sync,no_root_squash`
- 生产环境建议使用分布式存储（Ceph、GlusterFS）替代 NFS

### 3. Harbor 自签证书
- CA 根证书 → 服务端证书链
- SAN 扩展确保客户端验证通过
- 需要更新系统 CA 信任库

### 4. 高可用设计
- 数据层高可用：外部 PostgreSQL（可配置主从）+ Redis（可配置哨兵/集群）
- 服务层高可用：多 Harbor 实例 + 负载均衡
- 存储层高可用：NFS 服务器自身的高可用（生产环境需考虑）

## 面试高频题

### Q1: Harbor 如何实现高可用？
**答：** Harbor 高可用需要三个层面：① 服务层部署多个 Harbor 实例；② 数据层使用外部 PostgreSQL 和 Redis；③ 存储层使用共享存储（NFS/分布式存储）。入口通过 SLB 或 Keepalived 做负载均衡。关键在于所有实例共享同一套数据库、缓存和存储。

### Q2: Harbor 使用 NFS 共享存储有什么风险？生产环境如何改进？
**答：** NFS 单点风险：NFS 服务器故障会导致所有 Harbor 不可用。生产环境建议：① 使用分布式存储（Ceph、MinIO）替代 NFS；② NFS 本身做高可用（NFS+DRBD+Keepalived）；③ 使用云存储服务（如阿里云 NAS）。

### Q3: 如何为 Harbor 配置自签 HTTPS 证书？
**答：** 步骤：① 生成 CA 根证书（openssl genrsa + openssl req -x509）；② 生成服务端 CSR 和证书（包含 SAN 扩展）；③ PEM 转 CERT 格式；④ 证书安装到系统信任锚点（update-ca-trust）；⑤ 在 harbor.yml 中配置证书路径。

### Q4: Harbor 多实例共享数据库时需要注意什么？
**答：** ① 所有实例的 harbor.yml 中数据库连接信息必须一致；② PostgreSQL 需要创建 registry、notarysigner、notaryserver 三个数据库；③ 数据库用户需要对所有库有完全权限；④ 建议数据库做主从复制保障高可用。

## 学习心得
- Harbor 是企业级容器镜像仓库的标准方案，高可用是生产必备
- 理解"共享存储+外部DB+外部缓存"是实现多实例一致性的关键
- 自签证书的完整流程是运维基本功，建议熟练掌握
- NFS 适合学习和小规模场景，生产环境要评估分布式存储方案
