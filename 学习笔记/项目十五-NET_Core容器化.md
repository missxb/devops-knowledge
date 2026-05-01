# 项目十五: .NET Core 容器化

## 项目概述
学习 .NET Core 应用的 Docker 容器化部署，包括 SDK 安装、项目创建、Dockerfile 编写、镜像构建和容器运行。

## 核心知识点

### 1. .NET Core 容器化流程
```
宿主机安装 SDK → 创建项目 → 编写 Dockerfile → 构建镜像 → 运行容器
```

### 2. Dockerfile 编写
```dockerfile
FROM microsoft/dotnet:latest    # 基础镜像
WORKDIR /app                     # 工作目录
COPY . /app                      # 复制源码
RUN dotnet restore               # 还原依赖
EXPOSE 80                        # 暴露端口
ENV ASPNETCORE_URLS http://*:80  # 配置监听地址
ENTRYPOINT ["dotnet","run"]      # 启动命令
```

### 3. .NET 5 容器化
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:5.0-buster-slim
WORKDIR /app
COPY . /app
RUN dotnet restore
EXPOSE 80 443
ENV ASPNETCORE_URLS http://*:80
ENTRYPOINT ["dotnet","run"]
```

### 4. 容器运行方式
- **直接运行**：`docker run -itd -p 80:80 镜像名`
- **挂载源码**：`docker run -it -p 80:80 -v $HOME/demo/app:/app microsoft/dotnet:latest`
- 挂载方式适合开发调试，不需要每次重新构建镜像

## 面试高频题

### Q1: Dockerfile 中 ENTRYPOINT 和 CMD 的区别？
**答：** ENTRYPOINT 定义容器启动时的固定命令，CMD 提供默认参数。ENTRYPOINT 不会被 docker run 的命令行覆盖（除非用 --entrypoint），CMD 会被覆盖。最佳实践：ENTRYPOINT 定义主命令，CMD 提供默认参数。

### Q2: 为什么 .NET Core 容器化需要设置 ASPNETCORE_URLS？
**答：** 默认情况下 ASP.NET Core 监听 localhost:5000，容器内 localhost 只能本机访问。设置 `ASPNETCORE_URLS=http://*:80` 让应用监听所有网络接口，容器外部才能通过端口映射访问。

### Q3: 多阶段构建（Multi-stage Build）有什么优势？
**答：** 第一阶段使用 SDK 镜像编译应用，第二阶段使用 runtime 镜像只复制编译产物。优势：① 最终镜像更小（runtime 比 SDK 小很多）；② 源码不包含在最终镜像中，更安全；③ 减少攻击面。

## 学习心得
- .NET Core 容器化相对简单，关键是理解 Dockerfile 最佳实践
- 开发环境可以用 Volume 挂载源码，生产环境用多阶段构建
- ASPNETCORE_URLS 环境变量是容器化部署的关键配置
- 容器化是微服务架构的基础，任何语言的应用都应该掌握容器化
