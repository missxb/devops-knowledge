# 项目十九: rsync/lsyncd/sersync 文件同步

## 项目概述
部署 rsync 文件同步服务，实现服务器间的文件备份和同步。rsync 是 Linux 下最常用的文件同步工具，支持增量传输、压缩传输和认证机制。

## 架构设计
```
Client (源) ──rsync──→ Server (目标: /backup)
   │
   └── rsync.password (客户端密码文件)
   
Server 端配置:
   ├── rsyncd.conf (主配置文件)
   ├── rsync.password (认证密码文件)
   └── /backup (备份目录)
```

**核心组件：**
- **rsync**：文件同步工具（端口 873）
- **rsyncd.conf**：服务端配置文件
- **rsync.password**：认证密码文件（权限 600）

## 部署步骤

### 关键要点总结

1. **服务端配置**
   - 配置 `/etc/rsyncd.conf`：
     - uid/gid：rsync 虚拟用户
     - port：873
     - max connections：200
     - auth users：rsync_backup
     - secrets file：/etc/rsync.password
     - 模块：[backup]，path=/backup

2. **创建虚拟用户**
   ```bash
   useradd rsync -M -s /sbin/nologin
   ```

3. **创建密码文件**
   ```bash
   echo "rsync_backup:123456" > /etc/rsync.password
   chmod 600 /etc/rsync.password
   ```

4. **创建备份目录**
   ```bash
   mkdir /backup
   chown rsync.rsync /backup
   ```

5. **启动服务**
   ```bash
   systemctl enable rsyncd && systemctl start rsyncd
   ```

6. **客户端配置**
   ```bash
   echo "123456" > /etc/rsync.password
   chmod 600 /etc/rsync.password
   rsync -avz /etc/passwd rsync_backup@192.168.10.91::backup --password-file=/etc/rsync.password
   ```

## 核心知识点

### 1. rsync 传输模式
- **本地同步**：`rsync -avz /src /dest`
- **SSH 模式**：`rsync -avz /src user@host:/dest`
- **daemon 模式**：`rsync -avz /src user@host::module`（本项目使用）

### 2. rsync 常用参数
- `-a`：归档模式（保留权限、时间戳等）
- `-v`：详细输出
- `-z`：压缩传输
- `--delete`：删除目标端多余文件
- `--password-file`：指定密码文件

### 3. rsync 安全要点
- 密码文件权限必须为 600
- 使用虚拟用户（nologin）运行
- hosts allow 限制访问网段
- read only 控制读写权限

### 4. lsyncd 和 sersync
- **lsyncd**：基于 inotify 的实时同步工具，监控文件变化自动触发 rsync
- **sersync**：类似 lsyncd，C++ 编写，性能更好
- 适合需要实时同步的场景

## 面试高频题

### Q1: rsync 的增量同步原理是什么？
**答：** rsync 使用"滚动校验和"算法：① 目标端将文件分成固定大小的块，计算校验和发送给源端；② 源端对源文件计算校验和，找出相同的块；③ 只传输不同的块。这种方式大大减少了网络传输量。

### Q2: rsync daemon 模式和 SSH 模式的区别？
**答：** SSH 模式：通过 SSH 协议传输，需要 SSH 登录权限，安全性好但性能稍差。Daemon 模式：rsync 自己监听 873 端口，通过 rsync 认证，不需要 SSH 权限，性能更好但需要额外配置密码文件。

### Q3: 如何实现 rsync 实时同步？
**答：** 三种方式：① lsyncd（基于 inotify 监控文件变化）；② sersync（类似 lsyncd，C++ 实现）；③ crontab 定时 rsync（非实时，但简单）。lsyncd 和 sersync 适合需要秒级同步的场景。

## 学习心得
- rsync 是运维必备工具，必须熟练掌握其配置和使用
- 密码文件权限 600 是最容易忽略的坑
- daemon 模式适合大规模文件同步，SSH 模式适合临时同步
- 实时同步需求推荐 lsyncd，配置简单且基于 inotify
- 生产环境建议结合 rsync + lsyncd + 监控告警实现完整的文件同步方案
