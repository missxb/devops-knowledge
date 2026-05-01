# 项目十一: MongoDB 4.4.2 副本集 + 认证部署

## 项目概述
部署 MongoDB 4.4.2 三节点副本集集群，配置认证模式，实现数据库的高可用和安全访问。

## 架构设计
```
MongoDB Replica Set
  ├── mongodb01 (192.168.10.113) PRIMARY
  ├── mongodb02 (192.168.10.112) SECONDARY
  └── mongodb03 (192.168.10.111) SECONDARY
        │
        └── keyfile 认证 (mongo.key)
```

**核心组件：**
- **MongoDB 4.4.2**：文档数据库
- **副本集（Replica Set）**：数据自动复制，主从切换
- **keyfile 认证**：集群内部通信认证
- **systemd 服务管理**：开机自启，进程管理

## 部署步骤

### 关键要点总结

1. **安装 MongoDB**
   - 创建目录结构：app、data、logs
   - 下载解压 MongoDB 二进制包
   - 创建 mongo 用户，修改目录权限

2. **配置 MongoDB**
   ```yaml
   storage:
     dbPath: /data/mongodb/data
     journal:
       enabled: true
   systemLog:
     destination: file
     path: /data/mongodb/logs/mongod.log
   net:
     port: 27017
     bindIp: 0.0.0.0
   replication:
     replSetName: "testrs0"
   ```

3. **systemd 服务配置**
   - 创建 mongodb.service 文件
   - User=mongo, Group=mongo
   - ExecStart 使用 --config 指定配置文件

4. **初始化副本集**
   ```javascript
   rs.initiate({
     _id: "testrs0",
     members: [
       { _id: 0, host: "192.168.10.113:27017" },
       { _id: 1, host: "192.168.10.112:27017" },
       { _id: 2, host: "192.168.10.111:27017" }
     ]
   })
   ```

5. **配置认证**
   - 创建 root 用户：`db.createUser({user:"root", pwd:"Root#123", roles:[{role:"root", db:"admin"}]})`
   - 生成 keyfile：`openssl rand -base64 756 > mongo.key`
   - 拷贝 keyfile 到所有节点，权限 400，属主 mongo
   - 配置文件添加 security.authorization 和 keyFile

## 核心知识点

### 1. MongoDB 副本集原理
- **Primary**：处理所有写操作，通过 Oplog 同步到 Secondary
- **Secondary**：复制 Primary 的数据，可配置为读节点
- **Arbiter**：参与选举投票，不存储数据（可选）
- 至少 3 个节点（奇数个），保证选举可用

### 2. 副本集同步机制
- **Oplog**：操作日志（local.oplog.rs 集合），记录所有写操作
- Secondary 持续从 Primary 拉取 Oplog 并重放
- Oplog 是固定大小的 capped collection
- 初始同步：全量数据复制 + Oplog 追平

### 3. 副本集选举机制
- Primary 故障时，Secondary 自动发起选举
- 需要获得多数派（>50%）投票才能成为 Primary
- 奇数节点避免脑裂
- priority 参数控制选举优先级

### 4. 认证模式
- **keyfile 认证**：集群内部节点间通信认证
- **SCRAM 认证**：客户端用户名密码认证
- keyfile 必须在所有节点间一致，权限 400

## 面试高频题

### Q1: MongoDB 副本集为什么建议奇数个节点？
**答：** 副本集选举需要多数派（>50%）投票。3 节点容忍 1 个故障（需要 2 票），4 节点也只能容忍 1 个故障（需要 3 票），但多消耗一个节点资源。5 节点容忍 2 个故障。奇数节点在容错能力和资源消耗之间取得最佳平衡。

### Q2: MongoDB Oplog 的作用是什么？
**答：** Oplog（操作日志）是 MongoDB 副本集同步的核心。它是一个固定大小的 capped collection（local.oplog.rs），记录所有写操作。Secondary 通过持续拉取和重放 Oplog 来保持与 Primary 的数据同步。Oplog 也用于故障恢复时的数据追赶。

### Q3: 如何实现 MongoDB 的读写分离？
**答：** ① 在连接字符串中设置 readPreference=secondaryPreferred；② 在代码中显式指定从 Secondary 读取；③ 注意 Secondary 读取可能有延迟（复制延迟），适合对一致性要求不高的场景。

### Q4: keyfile 认证和 SCRAM 认证的区别？
**答：** keyfile 用于集群内部节点间认证（内部安全），基于共享密钥。SCRAM（Salted Challenge Response Authentication Mechanism）用于客户端认证（用户安全），基于用户名密码。生产环境两者都需要配置。

## 学习心得
- MongoDB 副本集部署相对简单，关键是理解 Oplog 同步和选举机制
- keyfile 认证是集群安全的基础，注意文件权限和一致性
- 奇数节点是副本集的硬性要求，避免脑裂问题
- 生产环境建议配置 readPreference 控制读写分离策略
- 副本集不等于分片，大数据量还需要配置 Sharding
