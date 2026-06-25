# Docker 入门学习笔记

> 基于 Docker Desktop + WSL Ubuntu 实操，覆盖核心概念与面试高频题

---

## 一、Docker 是什么

### 上帝视角

**Docker = 把应用和环境打包成集装箱，任何机器上一键启动。** 不用在每台机器上重装 JDK、MySQL、Redis。

### 工作中为什么用

| 没有 Docker | 有 Docker |
|-------------|----------|
| 每台机器重装环境，版本不一致，半天跑不起来 | `docker compose up -d`，3 分钟搞定 |
| "我电脑上能跑啊，服务器怎么不行" | 镜像里锁死版本，本地和线上完全一致 |
| 项目 A 要 MySQL 5.7，项目 B 要 MySQL 8.0，冲突 | 两个容器各跑各的，互不干扰 |

### 工作中 Docker 跑在哪

| 环境 | 干什么 |
|------|--------|
| 开发电脑（Docker Desktop） | 本地开发、调试 |
| 云服务器 | 生产环境运行 |
| CI/CD 流水线 | 自动构建镜像、自动部署 |

### Docker vs K8s vs containerd

| | Docker | K8s | containerd |
|------|--------|-----|-----------|
| 干什么 | 打包应用、启动容器 | 管理几千个容器（编排） | 底层运行时，真正启动容器 |
| 类比 | 集装箱 | 港口调度系统 | 发动机 |

Docker Desktop 内置了 containerd，你敲 `docker run`，底层是 containerd 在干活。

---

## 二、镜像 vs 容器（面试必问）

### 卡带类比

```
镜像 Image    = 游戏卡带（只读模板，存着代码+环境）
容器 Container = 卡带插进机器，正在玩的游戏（活的实例）
```

| | 卡带（镜像） | 正在玩的游戏（容器） |
|------|-----------|-------------------|
| 怎么来的 | 下载的 / 自己做的 | `docker run` 启动 |
| 能改吗 | 只读 | 能，运行时可写 |
| 能复制吗 | 一张卡带插 10 台机器 | 每台机器上各跑一个 |
| 命令 | `docker pull` / `docker build` | `docker run` / `docker start` |

**一个镜像 → 可启动多个容器。一个容器只基于一个镜像。**

---

## 三、一个项目怎么拆成容器

### 拆解原则：一个容器只干一件事

```
✅ 正确：3 个镜像 → 3 个容器 → 同一网络互联
   镜像 1：mysql:8.0     → 容器 my-mysql
   镜像 2：redis:7.0     → 容器 my-redis
   镜像 3：自定义镜像     → 容器 my-app（JDK + jar）

❌ 错误：全塞进一个容器
   MySQL + Redis + JDK + jar → 一锅粥
```

### 容器间如何通信

同一个网络里，通过**容器名**互相访问（Docker 自动解析成 IP）：

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://my-mysql:3306/mydb    ← 容器名当域名用
  data:
    redis:
      host: my-redis                         ← 容器名当域名用
```

---

## 四、创建容器

### docker run — 唯一方式

```bash
docker run [参数] 镜像名:版本
```

**`docker run` 同时做了三件事：下载镜像 + 创建容器 + 启动容器。**

### 参数详解

| 参数 | 含义 | 必须吗 |
|------|------|--------|
| `-d` | 后台运行（不占终端） | 不写就前台跑，关终端容器停 |
| `--name 名字` | 给容器起名 | 不写 Docker 自动起随机名 |
| `-e 变量=值` | 设置环境变量 | 看镜像要求 |
| `-p 本机:容器` | 端口映射 | 不写外面连不上 |
| `-v 本机:容器` | 目录挂载 | 数据持久化（后面学） |

### 示例：启动 MySQL

```bash
sudo docker run -d --name mysql-test \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -p 3307:3306 \
  mysql:8.0
```

### Q：`-e` 的变量名是随意起的吗？

**不是。** 每个官方镜像有规定的环境变量：

| 镜像 | 变量名 | 作用 |
|------|--------|------|
| `mysql:8.0` | `MYSQL_ROOT_PASSWORD` | root 密码 |
| `mysql:8.0` | `MYSQL_DATABASE` | 自动创建数据库 |
| `redis:7.0` | 无 | Redis 不需要密码 |
| `postgres:16` | `POSTGRES_PASSWORD` | 超级用户密码 |

`--name` 是你起的名字（给 Docker 看），`-e` 里的变量名是镜像启动脚本读的（不能改）。

### Q：镜像名怎么来的？

```
docker run mysql:8.0
            │    │
            │    └── 版本号（tag）
            └── 镜像名（Docker Hub 上的仓库名）
```

全称：`docker.io/library/mysql:8.0`，平时省略。去 hub.docker.com 搜镜像名查版本和变量。

### 常见官方镜像

| 技术 | 镜像名 | 常用版本 |
|------|--------|---------|
| MySQL | `mysql` | 8.0, 8.4 |
| Redis | `redis` | 7.0, 7.2 |
| Nginx | `nginx` | latest, 1.25 |
| PostgreSQL | `postgres` | 16 |
| OpenJDK | `openjdk` | 17-slim, 21-slim |
| Python | `python` | 3.12, 3.11 |

---

## 五、容器生命周期

```bash
docker ps                # 看正在运行的容器
docker ps -a             # 看所有容器（含停止的）
docker stop 容器名        # 停止
docker start 容器名       # 启动（已存在的容器）
docker restart 容器名     # 重启
docker rm 容器名          # 删除容器
```

### docker run vs docker start

```
docker run   = 从镜像创建新容器 + 启动（第一次用）
docker start = 启动已创建的容器（后续用）

同一个名字只能 run 一次。第二次 run 报"名字已存在"，用 start。
```

---

## 六、进入容器

### docker exec

```bash
# 执行一条命令，拿结果就退出
docker exec 容器名 ls /var/lib/mysql

# 进入容器，开交互 shell（进去随便敲）
docker exec -it 容器名 bash

# 直接连 MySQL
docker exec -it mysql-test mysql -uroot -p123456
```

| | 不加 -it | 加 -it |
|------|---------|--------|
| 干什么 | 执行一条命令，拿结果 | 进入容器，像操作小 Linux |
| 类比 | `ssh 服务器 "ls"` | `ssh 服务器`，随便敲 |
| 什么时候 | 快速看一眼 | 进去排查问题 |

---

## 七、看日志

```bash
docker logs 容器名               # 全部日志
docker logs --tail 10 容器名     # 最后 10 行
```

等于 `tail -f`，但对容器。

---

## 八、镜像管理

```bash
docker pull 镜像名:版本     # 下载镜像（只下载，不启动）
docker images              # 看本地有哪些镜像
docker rmi 镜像名           # 删除镜像
```

### docker pull vs docker run

```
docker pull = 只下载安装包
docker run  = 下载 + 安装 + 启动（三合一）
```

生产环境先 pull 确保下载成功，再 run。

### Q：镜像下载到哪了？

Docker 自己管理，存在 WSL 虚拟磁盘里。永远不需要手动找路径，用 `docker images` 管理。

---

## 九、镜像分层（面试必问）

### 是什么

**镜像不是一个大文件，而是一层一层叠起来的。** 每一层 = Dockerfile 的一行指令，只记录相对上一层的**变化**。

```
redis:7.0 的分层（从下往上看）：
  第1层：Ubuntu 基础系统        85.2MB
  第2层：创建 redis 用户组     41kB
  第3层：更新系统               4.23MB
  第4层：设置环境变量           0B
  第5层：下载安装 Redis        30MB
  第6层：复制启动脚本           20.5kB
  第7层：启动命令 CMD           0B
  ─────────────────────────
  最终镜像：164MB
```

```bash
docker image history redis:7.0    # 查看镜像分层
```

### 为什么分层

**层可以复用和缓存。** 两个镜像共用 Ubuntu 基础层，那一层只存一份。

### Q：为什么改代码不会重新下载整个镜像？

自己项目的 Dockerfile：

```dockerfile
FROM openjdk:17-slim              ← 第1层：JDK 基础
RUN apt-get update                 ← 第2层：更新
COPY target/my-app.jar /app.jar   ← 第3层：你的代码
CMD ["java", "-jar", "/app.jar"]  ← 第4层：启动命令
```

修改 Java 代码后重新 `docker build`：

```
第1层 FROM openjdk  → ✅ 缓存命中，跳过
第2层 RUN apt-get   → ✅ 缓存命中，跳过
第3层 COPY jar      → 🔄 重新构建（jar 变了！）
第4层 CMD          → 🔄 重新构建（上层变了也得重建）
────────────────────
只重建最后 2 层，3 秒完成。JDK 几百 MB 不用重新下载。
```

`docker pull` 同理：本地已有某一层就跳过，不重复下载。

---

## 十、命令速查

| 命令 | 作用 |
|------|------|
| `docker pull 镜像:版本` | 下载镜像 |
| `docker run -d --name -e -p 镜像:版本` | 创建并启动容器 |
| `docker ps` | 看运行中容器 |
| `docker ps -a` | 看所有容器 |
| `docker stop 容器` | 停止 |
| `docker start 容器` | 启动已有容器 |
| `docker restart 容器` | 重启 |
| `docker rm 容器` | 删除容器 |
| `docker exec -it 容器 bash` | 进入容器 |
| `docker exec 容器 命令` | 在容器里执行命令 |
| `docker logs 容器` | 看日志 |
| `docker images` | 看本地镜像 |
| `docker image history 镜像` | 看镜像分层 |
