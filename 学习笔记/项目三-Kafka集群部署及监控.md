# 项目三: Kafka 集群部署及监控

## 项目概述
部署 ZooKeeper 集群和 Kafka 集群，实现分布式消息系统的高可用。同时配置 Prometheus + Grafana 监控体系，以及 Kafka-Eagle 管理界面，实现集群的可视化管理和监控。

## 架构设计
```
                    ┌─────────────┐
                    │  Kafka-Eagle │  (管理界面)
                    └──────┬──────┘
                           │
Producer → Kafka Cluster (3节点) → Consumer
              ├── kafka01
              ├── kafka02
              └── kafka03
                    │
           ZooKeeper Cluster (3节点)
              ├── zk01
              ├── zk02
              └── zk03
                    │
           Prometheus + Grafana (监控)
```

**核心组件：**
- **JDK 8**：Java 运行环境
- **ZooKeeper 3.7.0**：分布式协调服务，Kafka 依赖
- **Kafka**：分布式消息队列
- **Kafka-Eagle**：Kafka 集群管理 Web 界面
- **Prometheus + Grafana**：监控和可视化

## 部署步骤

### 关键要点总结

1. **ZooKeeper 集群部署**
   - 安装 JDK 8u181
   - 下载 ZooKeeper 3.7.0，配置 `zoo.cfg`
   - 关键参数：tickTime=2000、initLimit=10、syncLimit=5
   - 配置 `server.A=B:C:D`（A=myid，B=IP，C=数据同步端口，D=选举端口）
   - 在 dataDir 下创建 myid 文件（值为 0、1、2）
   - 验证：`zkServer.sh status` 查看 leader/follower 角色

2. **ZooKeeper 监控配置**
   - ZooKeeper 3.6.0+ 内置 Prometheus 支持
   - 配置 `metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider`
   - 暴露 HTTP 端口 7000 提供 metrics 数据

3. **Kafka 集群部署**
   - 通用配置：broker.id、listeners、log.dirs、zookeeper.connect
   - 分节点配置：每个 broker 的 broker.id 和 listeners 不同
   - 启动：`kafka-server-start.sh -daemon config/server.properties`

4. **Kafka-Eagle 管理界面**
   - 下载安装 Kafka-Eagle
   - 配置集群连接信息
   - 提供 Topic 管理、消费者监控、消息查询等功能

5. **Kafka 监控**
   - 方式一：kafka_exporter + Prometheus
   - 方式二：JMX Exporter
   - Grafana Dashboard 可视化

## 核心知识点

### 1. ZooKeeper 在 Kafka 中的作用
- **Broker 注册**：每个 Broker 启动时在 ZooKeeper 注册
- **Topic 注册**：Topic 的分区和副本信息存储在 ZooKeeper
- **Consumer 注册**：消费者组的偏移量管理（新版 Kafka 已迁移到内部 Topic）
- **选举 Controller**：Kafka 集群的 Controller 通过 ZooKeeper 选举

### 2. ZooKeeper 集群角色
- **Leader**：处理所有写请求和读请求
- **Follower**：处理读请求，参与写请求的投票
- **Observer**：处理读请求，不参与投票（扩展读能力）

### 3. ZooKeeper 核心参数
- `tickTime`：心跳间隔（2000ms）
- `initLimit`：Follower 连接 Leader 的初始化时间（10 * tickTime = 20s）
- `syncLimit`：Leader 与 Follower 通信超时（5 * tickTime = 10s）
- `server.A=B:C:D`：A=myid，B=IP，C=同步端口 2888，D=选举端口 3888

### 4. Kafka 核心概念
- **Broker**：Kafka 服务器节点
- **Topic**：消息的逻辑分类
- **Partition**：Topic 的物理分区，实现并行处理
- **Replica**：分区的副本，保障数据冗余
- **Consumer Group**：消费者组，实现负载均衡消费

## 面试高频题

### Q1: ZooKeeper 集群为什么建议奇数个节点？
**答：** ZooKeeper 使用 ZAB 协议进行 Leader 选举和数据同步，需要超过半数节点（quorum）同意才能正常工作。3 节点容忍 1 个故障，5 节点容忍 2 个故障。4 节点也只能容忍 1 个故障（quorum=3），与 3 节点相同但多消耗资源，性价比低。

### Q2: Kafka 如何保证消息不丢失？
**答：** ① 生产者端：设置 `acks=all`，确保所有副本确认写入；② Broker 端：设置 `min.insync.replicas >= 2`，确保至少 2 个副本同步；③ 消费者端：手动提交 offset，处理完再确认。

### Q3: Kafka 的 ISR 机制是什么？
**答：** ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。如果 Follower 复制延迟超过 `replica.lag.time.max.ms`，会被踢出 ISR。Producer 的 `acks=all` 要求 ISR 中所有副本确认，才能认为写入成功。ISR 机制在性能和可靠性之间取得平衡。

### Q4: ZooKeeper 3.6.0+ 的 Prometheus 监控如何配置？
**答：** 在 `zoo.cfg` 中添加 MetricsProvider 配置：`metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider`，设置 `metricsProvider.httpPort=7000`。然后在 Prometheus 中添加 scrape target 即可。

## 学习心得
- ZooKeeper 是很多分布式系统（Kafka、HBase、Dubbo）的基础设施，必须熟练掌握
- myid 文件和 zoo.cfg 的 server 配置是 ZK 集群最容易出错的地方
- Kafka 监控需要关注：分区数、副本同步状态、消费者 Lag、Broker 负载
- Kafka-Eagle 是轻量级的 Kafka 管理工具，适合中小规模集群
