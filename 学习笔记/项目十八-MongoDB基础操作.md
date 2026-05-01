# 项目十八: MongoDB 基础操作

## 项目概述
MongoDB 基础知识学习，涵盖 CRUD 操作、索引、复制集和数据分片等核心概念。

## 核心知识点

### 1. MongoDB CRUD 操作

**创建文档：**
```javascript
// 插入单个文档
db.accounts.insertOne({ _id: "account1", name: "alice", balance: 100 })
// 插入多个文档
db.accounts.insertMany([{ name: "bob", balance: 50 }, { name: "charlie", balance: 500 }])
```

**读取文档：**
```javascript
db.accounts.find()                           // 读取全部
db.accounts.find({name: "alice"})            // 条件查询
db.accounts.find({},{name:1, balance:1})     // 投影（只返回指定字段）
db.accounts.find().sort({balance:-1})        // 排序（-1 降序）
db.accounts.find().skip(2).limit(5)          // 分页
```

**游标操作：**
```javascript
var cursor = db.accounts.find().noCursorTimeout()
cursor.hasNext()       // 是否有下一个
cursor.forEach(printjson)  // 遍历
cursor.close()         // 关闭游标
// 执行顺序：sort → skip → limit
```

### 2. MongoDB 索引
- 单字段索引、复合索引、多键索引、文本索引、地理空间索引
- 索引提高查询效率但增加写入开销
- explain() 分析查询计划

### 3. MongoDB 副本集（Replica Set）
- **Primary**：处理所有写操作
- **Secondary**：复制 Primary 数据，可配置为读节点
- **Arbiter**：参与选举投票，不存储数据
- 同步机制：通过 Oplog（local.oplog.rs）
- 心跳检测：节点间互相探测
- 自动选举：Primary 故障时 Secondary 自动选举

### 4. MongoDB 数据分片（Sharding）
- **mongos**：路由服务，客户端入口
- **config server**：配置服务器，存储元数据和分片信息
- **shard**：数据分片，每个 shard 可以是副本集
- 分片键（Shard Key）决定数据分布策略

### 5. MongoDB 数据安全
- 认证模式：SCRAM、x.509、LDAP、Kerberos
- 授权：基于角色的访问控制（RBAC）
- 加密：TLS/SSL 传输加密、静态数据加密
- 审计：操作审计日志

## 面试高频题

### Q1: MongoDB 的 _id 字段有什么特点？
**答：** _id 是 MongoDB 文档的主键，自动创建索引。特点：① 每个文档必须有 _id；② 在集合中必须唯一；③ 默认类型是 ObjectId（12 字节：时间戳+机器+进程+计数器）；④ 可以自定义类型（字符串、整数等）。

### Q2: MongoDB 副本集的 Oplog 是什么？
**答：** Oplog（操作日志）是一个固定大小的 capped collection（local.oplog.rs），记录所有写操作。Secondary 通过持续拉取和重放 Oplog 保持同步。特点：① 固定大小，循环覆盖；② 记录幂等操作；③ 初始同步时先全量复制再追 Oplog。

### Q3: MongoDB 分片的 Shard Key 如何选择？
**答：** Shard Key 选择原则：① 高基数（取值范围广）：避免数据倾斜；② 写分布均匀：避免热点 shard；③ 查询相关：大多数查询包含 shard key，避免跨 shard 查询。常用策略：哈希分片（均匀分布）、范围分片（范围查询友好）。

### Q4: MongoDB 和 MySQL 的区别？
**答：** ① 数据模型：MongoDB 文档型（JSON/BSON），MySQL 关系型（表/行）；② Schema：MongoDB 灵活 schema，MySQL 固定 schema；③ 扩展：MongoDB 原生支持分片水平扩展，MySQL 扩展较复杂；④ 事务：MongoDB 4.0+ 支持多文档事务，MySQL 一直支持；⑤ 适用场景：MongoDB 适合半结构化数据、快速迭代，MySQL 适合强一致性事务场景。

## 学习心得
- MongoDB 的文档模型非常灵活，适合快速开发和迭代
- 游标操作的执行顺序（sort→skip→limit）是常见考点
- 副本集的 Oplog 机制是理解 MongoDB 高可用的关键
- 分片键的选择直接影响集群性能和可扩展性
- MongoDB 的安全模型（认证+授权+加密+审计）是生产部署的必备知识
