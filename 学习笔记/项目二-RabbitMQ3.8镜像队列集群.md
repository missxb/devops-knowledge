# 项目二: RabbitMQ 3.8 镜像队列集群

## 项目概述
部署 RabbitMQ 3.8.5 三节点镜像队列集群，实现消息队列的高可用。镜像队列模式下，消息会在所有节点间同步复制，确保单节点故障时消息不丢失。

## 架构设计
```
Producer → RabbitMQ Cluster (3节点镜像队列)
              ├── mq01 (192.168.10.220)
              ├── mq02 (192.168.10.221)
              └── mq03 (192.168.10.222)
                    ↓
            Prometheus + Grafana 监控
```

**核心组件：**
- **Erlang 23.0.2**：RabbitMQ 运行时环境
- **RabbitMQ 3.8.5**：消息队列服务
- **镜像队列**：消息在所有节点间同步复制
- **Management Plugin**：Web 管理界面（端口 15672）

## 部署步骤

### 关键要点总结

1. **基础环境配置**
   - 主机名配置：mq01、mq02、mq03
   - hosts 文件互相解析
   - 关闭防火墙和 SELinux

2. **安装 Erlang + RabbitMQ**
   - 安装依赖 socat
   - 安装 Erlang RPM 包
   - 安装 RabbitMQ 3.8.5 RPM 包
   - 启用管理插件：`rabbitmq-plugins enable rabbitmq_management`
   - 创建管理员用户并授权

3. **组建集群**
   - 从 mq01 拷贝 `.erlang.cookie` 到 mq02、mq03（保证集群认证一致）
   - 修改 cookie 文件属主为 rabbitmq 用户
   - 将 mq02、mq03 加入集群：`rabbitmqctl stop_app && rabbitmqctl join_cluster rabbit@mq01 && rabbitmqctl start_app`

4. **配置镜像队列策略**
   - `rabbitmqctl set_policy ha-all "" '{"ha-mode":"all","ha-sync-mode":"automatic"}'`
   - 所有队列自动在所有节点间镜像同步

5. **监控配置**
   - Python 脚本通过 API 监控集群状态和内存使用
   - Prometheus + Grafana 集成监控

## 核心知识点

### 1. Erlang Cookie 机制
- `.erlang.cookie` 是 Erlang 节点间认证的共享密钥
- 所有集群节点必须使用相同的 cookie 文件
- 文件权限必须为 400，属主为 rabbitmq 用户

### 2. 镜像队列原理
- 镜像队列模式下，每个队列有一个 master 和若干 mirror
- 所有读写操作都经过 master，mirror 异步/同步复制
- `ha-mode: all` 表示所有节点都镜像
- `ha-sync-mode: automatic` 新加入的节点自动同步数据

### 3. RabbitMQ 端口说明
- **5672**：AMQP 协议端口（客户端连接）
- **15672**：Management Web UI
- **25672**：集群内部通信端口

### 4. 消息可靠性
- 持久化：消息 + 队列 + 交换机都需要持久化
- 确认机制：Publisher Confirm + Consumer Ack
- 幂等性：消费端需要保证消息处理的幂等性

## 面试高频题

### Q1: RabbitMQ 镜像队列和普通队列有什么区别？
**答：** 普通队列消息只存储在一个节点上，节点故障则消息丢失。镜像队列将消息复制到所有（或指定数量的）节点上，master 节点负责读写，mirror 节点实时同步。master 故障时，最老的 mirror 自动提升为新 master，保证消息不丢失。

### Q2: 如何保证 RabbitMQ 消息不丢失？
**答：** 三个层面保障：① 生产者确认（Publisher Confirm）；② 消息持久化（exchange、queue、message 都设置 durable/persistent）；③ 消费者手动确认（manual ack）。配合镜像队列实现高可用。

### Q3: .erlang.cookie 的作用是什么？为什么集群节点必须一致？
**答：** Erlang 使用 cookie 作为节点间通信的认证令牌，类似密码。只有 cookie 相同的节点才能互相发现和加入集群。cookie 必须在所有节点上完全一致，且权限为 400（仅 owner 可读），否则 Erlang VM 拒绝使用。

### Q4: RabbitMQ 集群的脑裂问题如何处理？
**答：** 网络分区（脑裂）时，RabbitMQ 提供三种处理策略：① `ignore`（忽略，不推荐）；② `pause-minority`（少数派节点暂停）；③ `autoheal`（自动恢复，选择一个分区的节点重启）。生产环境推荐 `pause-minority`。

## 学习心得
- Erlang Cookie 是集群组建的关键，注意文件权限和一致性
- 镜像队列虽然保证了高可用，但性能有一定损耗，需要权衡
- 消息可靠性需要生产者、Broker、消费者三方协同保障
- 监控是消息队列运维的重要环节，关注队列深度、内存使用、连接数等指标
