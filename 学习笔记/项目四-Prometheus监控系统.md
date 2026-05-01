# 项目四: Prometheus 监控系统

## 项目概述
部署 Prometheus 监控体系，包括 Prometheus Server、AlertManager 告警、Grafana 可视化，以及各类 Exporter 的部署。实现对服务器、中间件的全方位监控和告警通知。

## 架构设计
```
                ┌──────────────┐
                │   Grafana    │ (可视化展示)
                └──────┬───────┘
                       │
                ┌──────┴───────┐
                │  Prometheus  │ (数据采集存储)
                └──┬───┬───┬───┘
                   │   │   │
         ┌─────────┘   │   └─────────┐
         ▼             ▼             ▼
   node_exporter  redis_exporter  mysql_exporter
         │             │             │
     服务器指标    Redis指标     MySQL指标
                       │
                ┌──────┴───────┐
                │ AlertManager │ (告警分发)
                └──┬───┬───┬───┘
                   │   │   │
                   ▼   ▼   ▼
               邮件  微信  钉钉
```

**核心组件：**
- **Prometheus**：时序数据库，指标采集和存储
- **AlertManager**：告警管理和通知分发
- **Grafana**：数据可视化平台
- **各类 Exporter**：指标暴露器（node_exporter、redis_exporter、mysql_exporter）

## 部署步骤

### 关键要点总结

1. **Prometheus 安装部署**
   - 下载二进制包，配置 systemd 服务
   - 配置 prometheus.yml：global、scrape_configs、rule_files
   - 启动参数：`--storage.tsdb.retention=180d`（数据保留 180 天）

2. **Exporter 部署**
   - **node_exporter**：服务器指标采集（CPU、内存、磁盘、网络）
   - **redis_exporter**：Redis 指标采集（Grafana Dashboard: 11835）
   - **mysql_exporter**：MySQL 指标采集
   - 所有 Exporter 通过 systemd 管理，开机自启

3. **AlertManager 告警配置**
   - **邮箱告警**：配置 SMTP 服务器信息
   - **微信告警**：配置企业微信 API（corp_id、secret、agent_id）
   - **钉钉告警**：通过 Webhook 集成
   - **告警模板**：自定义告警消息格式，包含告警类型、级别、详情、时间等
   - **告警规则**：基于 PromQL 表达式定义触发条件

4. **Grafana 安装部署**
   - 安装 Grafana，配置 Prometheus 数据源
   - 导入 Dashboard 模板（如 Node Exporter Full、Redis Dashboard）

5. **Prometheus 优化**
   - 清理指定 key 的数据（需启用 `--web.enable-admin-api`）
   - 标签改写（relabel_config）：source_label、target_label、action

## 核心知识点

### 1. Prometheus 架构原理
- **Pull 模型**：Prometheus 主动从 Exporter 拉取指标
- **时序数据库**：按时间序列存储指标数据
- **PromQL**：强大的查询语言，支持聚合、过滤、计算
- **服务发现**：支持静态配置、Consul、Kubernetes 等多种发现方式

### 2. 告警规则语法
```yaml
groups:
  - name: 组名
    rules:
      - alert: 告警名
        expr: PromQL表达式      # 触发条件
        for: 持续时间            # 持续多久才告警
        labels:                  # 自定义标签
          severity: warning
        annotations:             # 告警描述
          summary: "告警摘要"
          description: "告警详情"
```

### 3. relabel_config 标签改写
- `replace`：正则匹配替换标签值
- `keep`：正则不匹配则删除 target
- `drop`：正则匹配则删除 target
- `labelmap`：正则匹配标签名，创建新标签
- `labeldrop`：正则匹配则删除标签
- `labelkeep`：正则不匹配则删除标签

### 4. 告警变量
- `{{ $labels.xxx }}`：获取标签值
- `{{ $value }}`：获取 PromQL 计算值
- 时间格式化：`{{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}`（北京时间）

## 面试高频题

### Q1: Prometheus 的 Pull 和 Push 模型有什么区别？
**答：** Pull 模型（默认）：Prometheus 主动拉取 Exporter 的 metrics 端点。优点是 Exporter 无需知道 Prometheus 地址，便于解耦。Push 模型：通过 Pushgateway 中转，适合短期任务。Pull 更适合长期运行的服务监控。

### Q2: 如何配置 Prometheus 告警的静默和抑制？
**答：** AlertManager 支持：① **静默（Silence）**：临时屏蔽特定告警，按标签匹配设置有效期；② **抑制（Inhibit）**：当高优先级告警触发时，自动抑制低优先级告警；③ **分组（Group）**：将相关告警合并发送，减少通知风暴。

### Q3: Prometheus 数据保留策略如何配置？
**答：** 启动参数 `--storage.tsdb.retention=180d` 设置数据保留 180 天。也可以用 `--storage.tsdb.retention.size=50GB` 按大小限制。清理单个指标数据需要启用 `--web.enable-admin-api`，通过 `POST /api/v1/admin/tsdb/delete_series` API 操作。

### Q4: relabel_config 中 source_label 和 target_label 的关系是什么？
**答：** `source_label` 是源标签（已有标签），`target_label` 是目标标签。配合 `action` 和 `regex` 使用：`replace` 动作用正则匹配 source_label 的值来替换 target_label 的值；`keep`/`drop` 动作用正则匹配 source_label 的值来决定是否保留/删除 target。

## 学习心得
- Prometheus 是云原生监控的事实标准，必须深入理解其架构和 PromQL
- Exporter 生态非常丰富，几乎所有主流中间件都有对应的 Exporter
- 告警规则设计需要平衡灵敏度和误报率，`for` 参数很重要
- relabel_config 是 Prometheus 最灵活也最难理解的特性，需要多实践
- Grafana Dashboard 可以从社区导入，不必从零开始
