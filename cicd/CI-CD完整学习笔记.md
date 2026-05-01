# CI/CD 持续集成与持续交付 - 完整学习笔记

## 一、核心概念

### 1.1 CI - 持续集成 (Continuous Integration)
**定义：** 开发人员频繁（每天多次）将代码合并到共享仓库，每次合并都通过自动化构建和测试来验证。

**核心实践：**
- 频繁提交代码（至少每天一次）
- 自动化构建和测试
- 快速发现和修复问题
- 每次提交都触发流水线
- 测试不通过就不能合并

**好处：**
- 尽早发现集成问题，避免"合并地狱"
- 减少手动测试工作量
- 提高代码质量和团队信心

### 1.2 CD - 持续交付/持续部署

| 概念 | 持续交付 (Continuous Delivery) | 持续部署 (Continuous Deployment) |
|------|------|------|
| 定义 | 代码随时可以部署到生产 | 每次变更自动部署到生产 |
| 人工审批 | ✅ 需要手动点击发布 | ❌ 全自动 |
| 风险 | 较低（有人工把关） | 需要完善的自动化测试 |
| 适用场景 | 传统企业、合规要求高 | 互联网公司、快速迭代 |

---

## 二、工具链全景

### 2.1 版本控制
| 工具 | 特点 |
|------|------|
| **Git** | 分布式版本控制，事实标准 |
| **GitHub** | 最大的开源社区，CI用 GitHub Actions |
| **GitLab** | 自托管+SaaS，内置 CI/CD |
| **Gitee** | 国内平台，适合国内团队 |

### 2.2 CI/CD 平台对比

| 平台 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| **Jenkins** | 插件丰富(1800+)、灵活强大 | 配置复杂、UI老旧 | 企业级复杂流水线 |
| **GitLab CI** | 与GitLab深度集成、配置简洁 | 需要GitLab环境 | GitLab用户首选 |
| **GitHub Actions** | GitHub原生、社区Action丰富 | 仅限GitHub | 开源项目、GitHub用户 |
| **ArgoCD** | GitOps实践、声明式部署 | 仅支持K8s | K8s环境CD |
| **Tekton** | 云原生、K8s原生 | 学习曲线陡 | K8s环境CI |
| **Drone** | 轻量、容器原生 | 功能相对简单 | 小团队、简单流水线 |

### 2.3 构建工具
| 语言 | 工具 |
|------|------|
| Java | Maven / Gradle |
| Node.js | npm / yarn / pnpm |
| Go | go build |
| Python | pip / poetry / setuptools |
| C/C++ | Make / CMake |
| 容器 | Docker / Buildah / Kaniko |

### 2.4 测试工具
| 类型 | 工具 |
|------|------|
| 单元测试 | JUnit / Pytest / Go test |
| 集成测试 | TestContainers |
| E2E测试 | Selenium / Cypress / Playwright |
| API测试 | Postman / Newman / RestAssured |
| 性能测试 | JMeter / k6 / Locust |
| 安全扫描 | SonarQube / Trivy / Snyk |

---

## 三、典型 CI/CD 流水线

### 3.1 完整流程
```
代码提交 → 触发构建 → 单元测试 → 代码扫描 → 集成测试 → 打包镜像 
    → 推送镜像仓库 → 部署测试环境 → 验收测试 → 部署生产环境 → 监控告警
```

### 3.2 Jenkins Pipeline 示例
```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = 'harbor.example.com'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('代码检出') {
            steps {
                git branch: 'main', url: 'https://github.com/org/repo.git'
            }
        }
        
        stage('单元测试') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('代码扫描') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('构建镜像') {
            steps {
                sh """
                    docker build -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }
        
        stage('推送镜像') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', ...)]) {
                    sh """
                        docker login ${REGISTRY} -u ${USERNAME} -p ${PASSWORD}
                        docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
        
        stage('部署测试环境') {
            steps {
                sh "kubectl set image deployment/myapp myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -n test"
            }
        }
        
        stage('验收测试') {
            steps {
                sh 'pytest tests/acceptance/'
            }
        }
        
        stage('部署生产') {
            input {
                message "确认部署到生产环境？"
                ok "部署"
            }
            steps {
                sh "kubectl set image deployment/myapp myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -n production"
            }
        }
    }
    
    post {
        failure {
            mail to: 'team@example.com', subject: "构建失败: ${currentBuild.fullDisplayName}"
        }
    }
}
```

### 3.3 GitHub Actions 示例
```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
      - run: npm run lint

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to K8s
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### 3.4 GitLab CI 示例
```yaml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  image: maven:3.8-openjdk-17
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy_staging:
  stage: deploy
  script:
    - kubectl set image deployment/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  environment:
    name: staging
  only:
    - main

deploy_production:
  stage: deploy
  script:
    - kubectl set image deployment/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  environment:
    name: production
  when: manual
  only:
    - main
```

---

## 四、GitOps 实践

### 4.1 什么是 GitOps
**核心思想：** Git 仓库是唯一的真相来源（Single Source of Truth），所有环境配置和部署清单都存储在 Git 中。

### 4.2 GitOps vs 传统 CI/CD

| 对比项 | 传统 CI/CD | GitOps |
|--------|-----------|--------|
| 部署方式 | CI 工具 push 到 K8s | K8s agent pull Git 变更 |
| 状态管理 | 以 CI 工具为准 | 以 Git 仓库为准 |
| 回滚 | 重新部署旧版本 | git revert |
| 审计 | 查 CI 日志 | git log |
| 工具 | Jenkins/GitLab CI | ArgoCD/FluxCD |

### 4.3 ArgoCD 工作原理
```
开发者 → Git Push → Git 仓库 (K8s manifests)
                          ↓
                    ArgoCD 监听变更
                          ↓
                    对比 Git vs K8s 实际状态
                          ↓
                    自动同步到 K8s 集群
```

---

## 五、企业级 CI/CD 最佳实践

### 5.1 流水线设计原则
1. **快速反馈**：单元测试 < 5分钟，整个流水线 < 15分钟
2. **并行执行**：独立阶段并行运行，缩短总时间
3. **缓存优化**：依赖缓存、Docker 层缓存
4. **失败快速**：任何阶段失败立即停止
5. **环境一致**：开发/测试/生产环境尽量一致

### 5.2 安全左移 (Shift Left)
- **代码提交时**：静态代码分析 (SonarQube)
- **构建时**：依赖漏洞扫描 (Snyk/Trivy)
- **测试时**：DAST 安全扫描 (OWASP ZAP)
- **部署前**：镜像签名验证 (Cosign/Notary)

### 5.3 蓝绿/金丝雀部署
| 策略 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **蓝绿部署** | 两套环境切换流量 | 回滚秒级 | 资源翻倍 |
| **金丝雀发布** | 小比例流量验证 | 风险可控 | 监控要求高 |
| **滚动更新** | 逐步替换Pod | 资源占用少 | 回滚慢 |
| **A/B测试** | 按用户特征分流 | 精准测试 | 配置复杂 |

---

## 六、面试高频题

### Q1: CI 和 CD 的区别是什么？
**答：** CI（持续集成）关注代码合并后的自动化构建和测试，目标是尽早发现集成问题。CD 有两个含义：持续交付（代码随时可部署，但需手动审批）和持续部署（全自动部署到生产）。CI 是 CD 的前提。

### Q2: Jenkins Pipeline 中 Declarative 和 Scripted 的区别？
**答：** Declarative Pipeline 使用预定义结构（pipeline { stages { } }），语法简洁，适合大多数场景。Scripted Pipeline 使用 Groovy 脚本，更灵活但更复杂。推荐使用 Declarative，需要复杂逻辑时在 script 块中使用 Groovy。

### Q3: GitOps 的核心原则是什么？
**答：** 四个核心原则：① 声明式（所有配置用声明式定义）；② 版本化（Git 是唯一真相来源）；③ 自动化（变更自动应用）；④ 持续调谐（控制器持续监控实际状态与期望状态的差异）。

### Q4: 如何保证 CI/CD 流水线的安全性？
**答：** ① 密钥管理：使用 Vault 或 CI 平台的 Secret 管理，不硬编码；② 最小权限：CI 服务账号只给必要权限；③ 镜像签名：验证镜像来源；④ 依赖扫描：检测已知漏洞；⑤ 审计日志：记录所有部署操作。

### Q5: 流水线太慢怎么优化？
**答：** ① 并行化：独立测试/构建并行执行；② 缓存：依赖缓存、Docker 层缓存；③ 增量构建：只构建变更部分；④ 分层测试：PR 只跑单元测试，合并后再跑完整测试；⑤ 硬件升级：更快的构建机器。

---

## 七、学习心得
- CI/CD 是 DevOps 的核心实践，是所有自动化的基础
- Jenkins 虽然老但仍是企业主流，必须掌握 Pipeline 语法
- GitHub Actions 适合开源项目和小团队，上手快
- GitOps 是 K8s 环境下 CD 的最佳实践，ArgoCD 是首选工具
- 安全左移越来越重要，SAST/DAST 应该集成到流水线中
- 蓝绿/金丝雀部署是高级技能，需要配合完善的监控
