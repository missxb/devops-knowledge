# 2026-05-02 Kubernetes 基础概念学习笔记

---

## 一、Kubernetes 核心概念

### 1.1 什么是 Kubernetes？
Kubernetes（K8s）是一个开源的容器编排平台，用于自动化容器化应用的部署、扩展和管理。

### 1.2 核心架构
```
┌─────────────────────────────────────────────────┐
│                  Master Node                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │ API     │  │ Scheduler│  │ Controller      │  │
│  │ Server  │  │         │  │ Manager         │  │
│  └─────────┘  └─────────┘  └─────────────────┘  │
│  ┌─────────┐                                    │
│  │ etcd    │                                    │
│  └─────────┘                                    │
└─────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│                  Worker Node                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │ Kubelet │  │ Kube-   │  │ Container       │  │
│  │         │  │ Proxy   │  │ Runtime         │  │
│  └─────────┘  └─────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 1.3 核心组件
| 组件 | 功能 |
|------|------|
| **API Server** | 提供 REST API，整个系统的入口 |
| **Scheduler** | 负责调度 Pod 到合适的 Node |
| **Controller Manager** | 维护集群状态，如副本数 |
| **etcd** | 分布式键值存储，保存集群数据 |
| **Kubelet** | 管理节点上的容器 |
| **Kube-Proxy** | 网络代理，维护网络规则 |
| **Container Runtime** | 容器运行时（Docker/CRI-O） |

---

## 二、核心资源对象

### 2.1 Pod（最小的调度单位）
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
```

### 2.2 ReplicaSet（确保 Pod 副本数）
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.21
```

### 2.3 Deployment（应用部署）
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.21
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
```

### 2.4 Service（服务发现与负载均衡）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP  # ClusterIP/NodePort/LoadBalancer
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
# NodePort 类型
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### 2.5 Ingress（HTTP/HTTPS 路由）
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### 2.6 ConfigMap（配置管理）
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  config.json: |
    {
      "log_level": "info",
      "database_url": "postgres://db:5432/app"
    }
  APP_ENV: production
```

使用方式：
```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: myapp-config
      key: APP_ENV
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: myapp-config
```

### 2.7 Secret（敏感信息）
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  # base64 编码
  username: YWRtaW4=
  password: cGFzc3dvcmQxMjM=
```
注意：base64 编码不是加密，需要配合 EncryptionConfiguration 加密存储。

### 2.8 PersistentVolume & PersistentVolumeClaim
```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  hostPath:
    path: /mnt/data

# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```
使用 PVC：
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

---

## 三、核心命令

### 3.1 资源管理
```bash
# 创建资源
kubectl apply -f deployment.yaml

# 查看资源
kubectl get pods
kubectl get svc
kubectl get deployments
kubectl get all -n namespace

# 查看详情
kubectl describe pod myapp-pod
kubectl get pod myapp-pod -o yaml

# 查看日志
kubectl logs -f myapp-pod
kubectl logs -f myapp-pod -c container-name

# 进入容器
kubectl exec -it myapp-pod -- /bin/bash

# 删除资源
kubectl delete -f deployment.yaml
kubectl delete pod myapp-pod

# 扩缩容
kubectl scale deployment myapp --replicas=5
```

### 3.2 调试命令
```bash
# 查看节点
kubectl get nodes

# 查看资源使用
kubectl top nodes
kubectl top pods

# Port Forward
kubectl port-forward myapp-pod 8080:80

# 代理访问
kubectl proxy

# 复制文件
kubectl cp namespace/pod:/path/to/file ./local-file
```

---

## 四、调度与亲和性

### 4.1 Node Selector
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### 4.2 Node Affinity
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
```

### 4.3 Pod Affinity/Anti-Affinity
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: topology.kubernetes.io/zone
```

### 4.4 Taints & Tolerations
```bash
# 添加污点
kubectl taint nodes node1 key=value:NoSchedule

# 容忍污点
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

---

## 五、网络与安全

### 5.1 NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

### 5.2 RBAC（基于角色的访问控制）
```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 六、Helm 包管理

### 6.1 Helm 基础
```bash
# 添加仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 安装 Chart
helm install myrelease bitnami/nginx

# 查看 release
helm list

# 升级
helm upgrade myrelease bitnami/nginx

# 回滚
helm rollback myrelease 1

# 卸载
helm uninstall myrelease
```

### 6.2 自定义 Chart
```bash
# 创建 Chart
helm create mychart

# 打包 Chart
helm package mychart

# 模板渲染
helm template mychart ./mychart/values.yaml
```

---

## 七、学习建议

### 实践步骤
1. 搭建 Minikube 或 Kind 本地集群
2. 部署一个简单的 nginx 应用
3. 练习 Deployment 滚动更新
4. 尝试 Service 和 Ingress 配置
5. 学习 ConfigMap 和 Secret
6. 搭建 Kubernetes Dashboard

### 推荐学习路径
1. 先理解 Pod、Deployment、Service 三大件
2. 掌握 kubectl 常用命令
3. 学习 Helm 和包管理
4. 了解网络和存储
5. 学习安全（RBAC、NetworkPolicy）
6. 进阶：自动扩缩容、CI/CD 集成

---

## 八、相关资源

- 官方文档：https://kubernetes.io/docs/
- kubectl 备忘单：https://kubernetes.io/docs/reference/kubectl/quick-reference/
- Kubernetes the Hard Way：https://github.com/kelseyhightower/kubernetes-the-hard-way

---

## 今日总结
✅ Kubernetes 架构与核心组件
✅ 核心资源对象（Pod, Deployment, Service, etc.）
✅ 调度与亲和性
✅ 网络与安全基础
✅ Helm 包管理入门

**下一步**：明天学习 Kubernetes 进阶 + 微服务架构！
