# 容器技术

## Docker

### 核心概念
- **镜像 (Image)** - 只读模板
- **容器 (Container)** - 运行时实例
- **仓库 (Registry)** - 镜像存储

### 常用命令
```bash
# 镜像操作
docker pull <image>
docker build -t <name> .
docker
 images

# 容器操作
docker run -d --name <name> <image>
docker ps
docker stop <container>
docker rm <container>

# 日志
docker logs <container>
docker exec -it <container> /bin/bash
```

## Kubernetes

### 核心概念
- **Pod** - 最小部署单元
- **Deployment** - 声明式部署
- **Service** - 服务发现
- **Ingress** - 入口路由
- **ConfigMap/Secret** - 配置管理

### 架构组件
- **Master Node**
  - API Server
  - Scheduler
  - Controller Manager
  - etcd
- **Worker Node**
  - Kubelet
  - Kube-proxy
  - Container Runtime

## 学习笔记

待更新...
