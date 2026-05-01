# 项目五: Redis 5.0 集群部署

## 项目概述
部署 Redis 5.0 集群（Cluster 模式），实现分布式缓存的高可用和水平扩展。集群采用 3 主 3 从架构，通过哈希槽（Hash Slot）实现数据分片。

## 架构设计
```
Redis Cluster (6节点: 3主3从)
  ├── Master01: 192.168.10.220:7001 → Slot 0-5460
  │     └── Slave: 192.168.10.221:7002
  ├── Master02: 192.168.10.221:7001 → Slot 5461-10922
  │     └── Slave: 192.168.10.222:7002
  └── Master03: 192.168.10.222:7001 → Slot 10923-16383
        └── Slave: 192.168.10.220:7002
```

**核心组件：**
- **Redis 5.0.5**：内存数据库/缓存
- **Cluster 模式**：原生集群，去中心化架构
- **16384 个 Hash Slot**：数据分片基础
- **Gossip 协议**：节点间通信

## 部署步骤

### 关键要点总结

1. **安装 Redis**
   - 下载源码编译安装：`make && make install`
   - 创建集群目录结构：每个节点独立的 conf、data、logs 目录

2. **配置 Redis 节点**
   - 关键配置：port、bind 0.0.0.0、daemonize yes、dir（数据目录）、logfile
   - 集群专用配置：`cluster-enabled yes`、`cluster-config-file nodes.conf`、`cluster-node-timeout 15000`

3. **创建集群**
   ```bash
   redis-cli --cluster create \
     192.168.10.220:7001 192.168.10.220:7002 \
     192.168.10.221:7001 192.168.10.221:7002 \
     192.168.10.222:7001 192.168.10.222:7002 \
     --cluster-replicas 1 -a password
   ```
   - `--cluster-replicas 1`：每个主节点配 1 个从节点

4. **验证集群**
   - `cluster info`：查看集群状态（cluster_state:ok）
   - `cluster nodes`：查看节点信息和槽位分配
   - `cluster info` 中关注：cluster_slots_assigned=16384、cluster_size=3

## 核心知识点

### 1. Redis Cluster 数据分片
- **16384 个 Hash Slot**：`CRC16(key) % 16384` 决定 key 属于哪个 Slot
- 每个 Master 负责一部分 Slot，Slot 数量均分（约 5461 个/节点）
- 客户端请求发送到任意节点，如果 Slot 不在本节点，返回 MOVED 重定向

### 2. 集群高可用机制
- **故障检测**：节点间通过 Gossip 协议互相 PING/PONG
- **主观下线（PFAIL）**：半数以上 Master 认为目标节点不可达
- **客观下线（FAIL）**：触发从节点自动提升为 Master
- **自动故障转移**：从节点通过选举成为新 Master

### 3. Redis Cluster 限制
- 不支持多数据库（只有 db0）
- 不支持 SELECT 命令
- 事务只能在同一个 Slot 内的 key 上执行
- 批量操作（MGET/MSET）要求所有 key 在同一个 Slot（使用 Hash Tag `{tag}`）

### 4. 集群扩容和缩容
- 添加节点：`redis-cli --cluster add-node`
- 迁移 Slot：`redis-cli --cluster reshard`
- 删除节点：先迁走 Slot，再 `redis-cli --cluster del-node`

## 面试高频题

### Q1: Redis Cluster 的数据分片原理是什么？
**答：** Redis Cluster 使用 16384 个 Hash Slot 进行数据分片。key 通过 `CRC16(key) % 16384` 计算所属 Slot。每个 Master 节点负责一部分 Slot。客户端发送命令时，如果 key 不在当前节点，返回 MOVED 重定向。这种设计支持在线扩缩容。

### Q2: Redis Cluster 如何实现高可用？
**答：** ① 每个 Master 配一个或多个 Slave（从节点）；② 节点间通过 Gossip 协议互相心跳检测；③ 当半数以上 Master 认为某节点下线（PFAIL → FAIL），自动触发故障转移；④ 从节点通过 Raft 选举成为新 Master。

### Q3: 什么是 Hash Tag？解决什么问题？
**答：** Hash Tag 用 `{}` 包裹 key 中的一部分，如 `{user}:1` 和 `{user}:2`。CRC16 只计算 `{}` 内的部分，保证相关 key 分配到同一个 Slot。解决 Cluster 模式下跨 Slot 无法使用 MGET、事务（MULTI/EXEC）的问题。

### Q4: Redis Cluster 和 Redis Sentinel 有什么区别？
**答：** Sentinel 是主从模式的高可用方案，1 主 N 从，不支持数据分片。Cluster 是分布式集群方案，支持数据分片和水平扩展。Sentinel 适合数据量小（<16GB）的场景；Cluster 适合大数据量、高并发场景。

## 学习心得
- Redis Cluster 的 --cluster create 命令大大简化了集群创建过程
- 16384 个 Slot 是固定的，扩容时需要手动 reshard 迁移 Slot
- Hash Tag 是 Cluster 模式下处理多 key 操作的关键技巧
- 生产环境要注意 cluster-node-timeout 的设置，太短容易误判，太长故障恢复慢
- 每个节点至少 2 个端口：服务端口 + 集群总线端口（服务端口 + 10000）
