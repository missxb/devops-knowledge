# 安全运维

## 容器安全

### 镜像安全
- 使用官方或可信镜像
- 定期更新镜像
- 扫描镜像漏洞 (Trivy, Clair)
- 多阶段构建减少攻击面

### 运行时安全
- **非 root 用户运行**
- **只读根文件系统**
- **资源限制** (CPU, 内存)
- **网络隔离**
- **Seccomp / AppArmor** 配置

## Kubernetes 安全

### RBAC (基于角色的访问控制)
- Role / ClusterRole
- RoleBinding / ClusterRoleBinding
- ServiceAccount

### 网络策略
- Pod 间通信控制
- 命名空间隔离
- Ingress/Egress 规则

### Secrets 管理
- 不明文存储敏感信息
- 使用 K8s Secrets
- 外部密钥管理 (Vault)

## 供应链安全

### 签名验证
- 镜像签名 (Notary, Cosign)
- 代码签名
- SBOM (软件物料清单)

### 依赖管理
- 定期更新依赖
- 扫描依赖漏洞
- 固定版本号

## 学习笔记

待更新...
