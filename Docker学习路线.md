# Docker 学习路线（对标课程 + 面试标准）

## 上帝视角

Docker = 把你的应用和环境打包成集装箱，任何机器上一键启动。

---

## 🛑 不看

| 课程 | 理由 |
|------|------|
| 01 课程介绍 | 8 分钟废话 |
| 02 Docker 安装 | 你 WSL 里已有，敲 `docker --version` 确认即可 |
| 06 命令别名 | 没人用，所有文档都用标准命令 |
| 13 部署前端 | 你是后端 |

---

## 📅 四步学习路线

### 第一步：感受 Docker（看 03+04+05，1 天）

| 看课 | 学什么 | 实操 | 达成标准 |
|------|--------|------|---------|
| 03 部署 MySQL | `docker run` 一条命令启动 MySQL | 在 WSL 里启动一个 MySQL 容器 | 🟢 能讲 Docker run 做了什么 |
| 04 命令解读 | `-d -p -v -e --name` 每个参数含义 | 逐参数测试效果 | 🟢 面试能逐参数解释 |
| 05 常见命令 | `docker ps / stop / rm / logs / exec` | 每个命令实操一遍 | 🟡 能管理容器生命周期 |

**产出：** MySQL 容器跑起来，用 `docker exec` 进容器连上 MySQL。

---

### 第二步：制作镜像（看 09+10+07+08，1 天）

| 看课 | 学什么 | 实操 | 达成标准 |
|------|--------|------|---------|
| 09 Dockerfile 语法 | FROM/RUN/COPY/CMD/ENTRYPOINT/WORKDIR | 写一个 Dockerfile | 🟢 面试能背出常用指令 |
| 10 自定义镜像 | `docker build` 把 Dockerfile 打成镜像 | 构建并运行自己的镜像 | 🟡 能构建和推送镜像 |
| 07 数据卷挂载 | Volume：容器删了数据还在 | MySQL 数据挂 Volume 测试 | 🟡 面试能讲 Volume vs bind mount |
| 08 本地目录挂载 | bind mount：代码实时同步 | 挂本地目录到 Nginx 容器 | 🟡 开发环境代码热更新 |

**产出：** 自己的 Dockerfile + Volume 持久化 MySQL 数据。

---

### 第三步：多服务编排（看 11+12+14，1 天）

| 看课 | 学什么 | 实操 | 达成标准 |
|------|--------|------|---------|
| 11 容器网络 | 容器间通过服务名通信 | 两个容器互 ping | 🟡 能解释 bridge 网络 |
| 12 部署 Java | Dockerfile 打包 Spring Boot | 自己的 Java 项目容器化 | 🟡 Java jar 跑在容器里 |
| 14 Compose | 一个 yml 启动 Java+MySQL+Redis | 写 compose.yml | 🔴 一键启动三服务 |

**产出：** docker-compose.yml 启动 Java + MySQL + Redis。

---

### 第四步：排错 + 优化（W3，1 天）

| 内容 | 实操 | 达成标准 |
|------|------|---------|
| 多阶段构建 | 镜像从 800MB 压到 200MB | 🟡 面试能讲多阶段构建原理 |
| 故障模拟 | 端口冲突 / OOM / 镜像拉取失败 | 🟡 每个故障能排查解决 |
| 项目容器化 | W3 Spring Boot 项目完整部署 | 🔴 公网可访问 |

---

## 🎯 最终检验标准

不看资料，完成以下操作：

```
1. 写一个 Dockerfile，多阶段构建 Java 应用（< 250MB）
2. 写一个 docker-compose.yml，启动 Java + MySQL + Redis
3. 进 MySQL 容器，创建数据库和表
4. curl http://localhost:8080/health → 200 OK
```

全部做到 = Docker 已达标，面试够用。
