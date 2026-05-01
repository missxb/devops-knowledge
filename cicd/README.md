# CI/CD 持续集成与持续交付

## 概念

### CI (Continuous Integration)
- 频繁提交代码
- 自动化构建和测试
- 快速发现和修复问题

### CD (Continuous Delivery/Deployment)
- 自动化部署到测试/生产环境
- 降低发布风险
- 加快交付速度

## 工具链

### 版本控制
- Git
- GitHub / GitLab / Gitee

### CI/CD 平台
- **Jenkins** - 灵活强大的开源方案
- **GitLab CI** - 与 GitLab 深度集成
- **GitHub Actions** - GitHub 原生支持
- **ArgoCD** - GitOps 实践

### 构建工具
- Maven / Gradle (Java)
- npm / yarn (Node.js)
- Make / CMake (C/C++)

### 测试工具
- JUnit / Pytest
- Selenium / Cypress
- JMeter / k6

## 典型流程

```
代码提交 → 触发构建 → 单元测试 → 集成测试 → 打包镜像 → 部署到测试环境 → 验收测试 → 部署到生产
```

## 学习笔记

待更新...
