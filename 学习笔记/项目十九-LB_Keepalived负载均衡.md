# 项目十九: LB + Keepalived 负载均衡

## 项目概述
部署 HAProxy + Keepalived 高可用负载均衡方案。HAProxy 提供四/七层负载均衡，Keepalived 通过 VRRP 协议实现 VIP 漂移，保障负载均衡器本身的高可用。

## 架构设计
```
           Client
              │
         VIP: 192.168.10.200
              │
    ┌─────────┴─────────┐
    │                   │
HAProxy01 (MASTER)   HAProxy02 (BACKUP)
Keepalived           Keepalived
    │                   │
    └─────────┬─────────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
  Server1   Server2   Server3
```

**核心组件：**
- **HAProxy 2.0.21**：四/七层负载均衡器
- **Keepalived 2.0.17**：VRRP 协议实现 VIP 高可用
- **Lua 5.3.5**：HAProxy 依赖的 Lua 环境
- **chk_server.sh**：健康检查脚本

## 部署步骤

### 关键要点总结

1. **安装 Keepalived**
   - 安装依赖：gcc、openssl-devel、popt-devel、libnl-devel、kernel-devel
   - 编译安装 Keepalived 2.0.17
   - 配置文件：`/etc/keepalived/keepalived.conf`

2. **安装 HAProxy**
   - 编译安装 Lua 5.3.5（HAProxy 依赖）
   - 编译安装 HAProxy 2.0.21（支持 SSL、PCRE、Lua、Systemd）
   - 配置 systemd 服务文件

3. **Keepalived 配置**
   ```conf
   vrrp_instance VI_1 {
       state MASTER           # MASTER 或 BACKUP
       interface ens33        # 网卡接口
       virtual_router_id 52   # 路由器标识（MASTER/BACKUP 必须一致）
       priority 100           # 优先级（MASTER > BACKUP）
       advert_int 1           # VRRP 通告间隔
       authentication {
           auth_type PASS
           auth_pass 1111
       }
       virtual_ipaddress {
           192.168.10.200/32  # VIP 地址
       }
   }
   ```

4. **健康检查脚本**
   ```bash
   #!/bin/bash
   counter=$(netstat -na|grep "LISTEN"|grep "80"|wc -l)
   if [ "${counter}" -eq 0 ]; then
       systemctl stop keepalived
   fi
   ```

5. **邮件通知**
   - 配置 mailx 发送告警邮件
   - Keepalived 状态变化时自动通知

## 核心知识点

### 1. VRRP 协议原理
- **VRRP**（Virtual Router Redundancy Protocol）：虚拟路由冗余协议
- 多台路由器组成虚拟路由器组，共享一个 VIP
- MASTER 节点定期发送 VRRP 通告报文
- BACKUP 节点监听通告，超时未收到则接管 VIP
- 优先级高的节点成为 MASTER

### 2. Keepalived 核心组件
- **VRRP Stack**：VRRP 协议实现
- **Checkers**：健康检查模块
- **WatchDog**：看门狗进程
- **通知脚本**：状态变化时执行自定义脚本

### 3. HAProxy 工作模式
- **TCP 模式（四层）**：代理 TCP 流量，如 MySQL、Redis
- **HTTP 模式（七层）**：代理 HTTP 流量，支持 URL 路由、Header 改写
- **健康检查**：TCP/HTTP 健康检查，自动剔除故障后端

### 4. Keepalived 脑裂问题
- **脑裂**：MASTER 和 BACKUP 同时持有 VIP
- 原因：网络分区导致 VRRP 通告中断
- 解决：① 使用仲裁节点；② 检测脚本中加入对端 IP 检测；③ 使用第三方存储（如 Redis）做状态协调

### 5. Keepalived 通知脚本
- 支持 master、backup、fault 三种状态通知
- 可以在状态变化时自动执行操作（如发送邮件、启动/停止服务）
- 脚本参数：-m（模式）、-s（服务）、-a（VIP）、-n（通知类型）

## 面试高频题

### Q1: Keepalived 的 MASTER 和 BACKUP 是如何选举的？
**答：** 基于 VRRP 协议，优先级（priority）高的节点成为 MASTER。MASTER 定期发送 VRRP 通告（默认每秒），BACKUP 监听通告。如果 BACKUP 超时（3 倍通告间隔）未收到通告，则认为 MASTER 故障，BACKUP 提升为 MASTER 并接管 VIP。

### Q2: HAProxy 和 Nginx 作为负载均衡器的区别？
**答：** HAProxy：专注负载均衡，四/七层代理，性能更好，支持更丰富的健康检查算法。Nginx：Web 服务器 + 反向代理 + 负载均衡，功能更全面，配置更灵活。大规模负载均衡场景推荐 HAProxy，Web 服务场景推荐 Nginx。

### Q3: 如何解决 Keepalived 脑裂问题？
**答：** ① 检测脚本中增加对端 IP 的连通性检测；② 使用共享存储（如 Redis）做状态仲裁；③ 配置合理的 advert_int 和故障检测时间；④ 使用多播模式替代单播模式；⑤ 确保网络链路冗余。

### Q4: HAProxy 的健康检查机制有哪些？
**答：** ① TCP 检查：尝试建立 TCP 连接；② HTTP 检查：发送 HTTP 请求检查返回状态码；③ 自定义检查：通过脚本检查业务状态。配置参数：inter（检查间隔）、fall（连续失败次数）、rise（连续成功次数）。

## 学习心得
- Keepalived + HAProxy 是传统负载均衡高可用的经典方案
- VRRP 协议的优先级机制和通告机制是理解 Keepalived 的核心
- 健康检查脚本是保障业务连续性的关键，要根据业务特点定制
- 脑裂问题是 VRRP 方案的最大风险，需要多种手段防范
- 生产环境建议配合邮件/钉钉通知，第一时间感知状态变化
