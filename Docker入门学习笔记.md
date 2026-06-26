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
| `docker build -t 镜像名:版本 .` | 根据 Dockerfile 构建镜像 |
| `docker run --rm 镜像` | 容器退出后自动删除 |
| `docker run 镜像 env` | 打印容器的所有环境变量 |
| `docker volume create 卷名` | 创建 Volume |
| `docker volume ls` | 列出所有 Volume |
| `docker volume rm 卷名` | 删除 Volume |
| `docker compose up -d` | 一键启动 compose 所有服务 |
| `docker compose ps` | 看 compose 管理的容器 |
| `docker compose stop` | 停止所有服务 |
| `docker compose down` | 停止 + 删除容器（Volume 不删） |
| `docker compose down -v` | 停止 + 删除容器 + 删除 Volume |
| `docker network ls` | 列出所有网络 |

---

## 十一、Dockerfile — 构建自己的镜像

### 上帝视角

**Dockerfile = 镜像的"菜谱"。** 里面一行一个指令，告诉 Docker 怎么做出你的镜像。

```
Dockerfile（菜谱） → docker build（做菜） → Image（做好的蛋糕）
```

### 通用模板

```dockerfile
FROM _______________          ← 填：程序需要什么运行环境
COPY _______________          ← 填：代码文件在哪，放到镜像的哪里
RUN _______________           ← 填：需要编译吗？装依赖吗？
CMD _______________           ← 填：怎么启动这个程序
```

### 和 Spring Boot 项目的对应关系

把 IDEA 里跑项目的步骤翻译成 Dockerfile：

| IDEA 里做的事 | Dockerfile |
|-------------|-----------|
| 装 JDK 17 | `FROM eclipse-temurin:17-jdk-alpine` |
| 代码在本地目录 | `COPY target/my-app.jar /app.jar` |
| cd 到项目目录 | `WORKDIR /app`（非必须，让路径整齐） |
| mvn package（编译打包） | `RUN` 或本地做完直接 COPY |
| java -jar 启动 | `CMD ["java", "-jar", "/app.jar"]` |

### 示例：Hello.java → 镜像 → 容器

**第一步：写 Java 代码**

```bash
cd ~/docker-demo
cat > Hello.java << 'EOF'
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello from Docker!");
    }
}
EOF
```

**第二步：写 Dockerfile**

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
COPY Hello.java /app/
WORKDIR /app
RUN javac Hello.java
CMD ["java", "Hello"]
```

| 指令 | 人话翻译 |
|------|---------|
| `FROM eclipse-temurin:17-jdk-alpine` | "给我一个装了 JDK 17 的空 Linux" |
| `COPY Hello.java /app/` | "把 Hello.java 复制进镜像的 /app/ 目录" |
| `WORKDIR /app` | "后面所有操作默认在 /app 目录下执行" |
| `RUN javac Hello.java` | "在镜像里编译" |
| `CMD ["java", "Hello"]` | "容器启动时，运行 java Hello" |

**Q：为什么需要 WORKDIR /app？**

如果没有 WORKDIR，默认在根目录 `/`。`RUN javac Hello.java` 在 `/` 下执行，找不到 `/app/` 里的 `Hello.java`。WORKDIR 切换到 `/app`，后续命令都能找到文件。

**第三步：构建镜像**

```bash
sudo docker build -t hello-docker:1.0 .
```

`-t` = 起名（tag），`hello-docker` 是镜像名，`1.0` 是版本号，`.` = Dockerfile 在当前目录。

**第四步：运行容器**

```bash
sudo docker run hello-docker:1.0                    # Docker 自动分配随机容器名
sudo docker run --name my-hello hello-docker:1.0    # 指定容器名为 my-hello
```

---

### 核心指令

#### 1. FROM — 指定基础镜像

镜像最初是空的。`FROM` 选择一个预装了 JDK 的操作系统。

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
```

**没有 FROM，镜像里连 `java` 命令都不存在，程序完全没法跑。**

> **JDK 的作用：** `.java` 文件是给人看的，电脑只认字节码。JDK 里的 `javac` 把 `.java` 翻译成 `.class`，`java` 负责执行 `.class`。
>
> **jar 包：** `mvn package` 把编译后的 `.class`、配置文件、依赖库打包成一个 `target/my-app.jar`。
> 类比：手稿（源码）→ 出版社排版装订（mvn package）→ 成书（jar 包）。

#### 2. COPY — 复制文件进镜像

```dockerfile
COPY Hello.java /app/
```

把你电脑上的文件拷进镜像。左边是本机路径，右边是镜像里的目标路径。

#### 3. WORKDIR — 设置工作目录

```dockerfile
WORKDIR /app
```

后面所有命令都在这个目录下执行。不是必须的（可以写全路径），但让 Dockerfile 更整洁。

#### 4. RUN — 构建时执行命令

```dockerfile
RUN javac Hello.java          # 编译
RUN apt-get install -y curl   # 装工具
```

**只在 `docker build` 时执行一次。** 编译产物留在镜像里，运行时直接用。

#### 5. CMD — 容器启动时的默认命令

```dockerfile
CMD ["java", "-jar", "/app.jar"]
```

**容器一启动就执行。** Spring Boot 跑起来后不会停，容器就一直活着。程序结束，容器退出。

#### 6. ENV — 设置环境变量

```dockerfile
ENV DB_PASSWORD=dev123
```

在镜像里预设环境变量，代码通过 `application.yml` 的 `${DB_PASSWORD}` 占位符读取。

**ENV 完整生命周期：**

```
Dockerfile ENV → 容器环境变量 → application.yml ${...} → Spring Boot 读到
                           ↑
                   docker run -e 可以覆盖
```

**实例：**

```dockerfile
# Dockerfile
ENV DB_URL=jdbc:mysql://my-mysql:3306/mydb
ENV DB_USERNAME=root
ENV DB_PASSWORD=dev123
ENV REDIS_HOST=my-redis
ENV REDIS_PORT=6379
ENV SPRING_PROFILES_ACTIVE=dev
ENV JAVA_OPTS="-Xmx512m -Xms256m"
```

```yaml
# application.yml — 不写具体值，只写占位符
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
```

**开发/生产环境切换：**

| 环境 | 在哪 | 数据库 | 密码 |
|------|------|--------|------|
| 开发环境 | 你的电脑 | 本地 MySQL，只有测试数据 | `ENV DB_PASSWORD=123456` |
| 生产环境 | 云服务器 | 真实用户数据 | `docker run -e DB_PASSWORD=真密码` |

同一个镜像，`-e` 传不同密码。开发和生产各用各的，代码里不存密码。

**哪些要配 ENV：**

| 值 | 写死 | 用 ENV |
|----|------|--------|
| 数据库密码 | ❌ 进 Git 就泄露 | ✅ |
| Redis 地址 | ❌ 本地和线上不一样 | ✅ |
| 日志级别 | ❌ 开发 debug，线上 warn | ✅ |
| 上传文件路径 | ❌ Windows 和 Linux 不同 | ✅ |
| 项目名、版本号 | ✅ 永远不变 | ❌ 不需要 |
| 端口号 8080 | ✅ | ❌ 不需要 |

**规则：会因为环境不同而变的 → ENV。永远不变的 → 写死。**

#### 7. EXPOSE — 声明端口

```dockerfile
EXPOSE 8080
```

**只是声明"里面用 8080"，不真正打开端口。** 真正开端口靠 `docker run -p`。团队协作时必须写，让别人知道镜像用哪个端口。

#### 8. CMD vs ENTRYPOINT

| | CMD | ENTRYPOINT |
|------|-----|-----------|
| 能被 `docker run` 后面的命令覆盖吗 | ✅ 会被覆盖 | ❌ 不被覆盖 |
| 什么时候用 | 90% 场景 | 想把容器当命令行工具用 |

```bash
# CMD 可被覆盖：
docker run my-app echo hello   → 执行 echo hello（CMD 被覆盖，java 不跑了）

# ENTRYPOINT 不被覆盖：
docker run my-app other.jar    → 执行 java -jar other.jar（ENTRYPOINT 仍是 java -jar）
```

**工作中 90% 用 CMD 就够了。** 面试能讲"CMD 可被覆盖，ENTRYPOINT 不被覆盖"即可。

---

## 十二、Volume — 数据持久化

### 上帝视角

**容器删了，里面的数据也删了。Volume 让数据存到容器外面（宿主机），删容器不丢数据。**

### 工作中什么时候用

| 场景 | 没有 Volume | 有 Volume |
|------|-----------|----------|
| MySQL 容器删了 | 数据库全丢，灾难 | 数据在宿主机上，重建容器挂上去数据还在 |
| 升级 MySQL 版本 | 旧容器删了，数据没了 | 新容器挂同一个 Volume，数据无缝接上 |
| 代码热更新 | 改一行代码要重新 build 镜像 | 本地目录挂进容器，保存即生效 |

### 实操

**创建 Volume：**

```bash
sudo docker volume create mysql-data
sudo docker volume ls
```

**启动容器时挂载 Volume：**

```bash
sudo docker run -d --name mysql-test \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -p 3307:3306 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```

`-v mysql-data:/var/lib/mysql` = 把 Volume `mysql-data` 挂到容器里的 `/var/lib/mysql`（MySQL 存数据的位置）。

**核心结论：容器删了，Volume 在，数据在。新容器挂同一个 Volume → 数据无缝接上。**

### Volume vs bind mount

| | Volume | bind mount |
|------|--------|-----------|
| 路径 | Docker 自动管理，你不用管 | 你自己指定本机目录，如 `/home/yzx/my-data` |
| 你 ls 能看到吗 | 不能 | 能，就是一个普通文件夹 |
| 什么时候用 | 数据库持久化（省心） | 开发时代码实时同步到容器 |

---

## 十三、docker-compose — 一键启动多服务

### 上帝视角

**把多个 `docker run` 命令写进一个 yml 文件，敲 `docker compose up -d` 一键全启动。**

### 基本 compose.yml

```yaml
services:                       # 你要启动的服务列表
  mysql:                        # 第 1 个服务，名字叫 mysql
    image: mysql:8.0            # 等于 docker run 的镜像名
    container_name: my-mysql    # 等于 --name my-mysql
    environment:                # 等于 -e
      MYSQL_ROOT_PASSWORD: "123456"
    ports:                      # 等于 -p
      - "3307:3306"
    volumes:                    # 等于 -v
      - mysql-data:/var/lib/mysql

  redis:                        # 第 2 个服务
    image: redis:7.0
    container_name: my-redis
    ports:
      - "6379:6379"

volumes:                        # 声明/定义 Volume
  mysql-data:
```

### Q：为什么有两个 volumes？

- **服务里的 `volumes:`** — 使用哪个 Volume，挂到容器哪个路径（格式：`Volume名:容器内路径`）
- **顶层的 `volumes:`** — 声明/定义这个 Volume。如果不存在，Docker 自动创建

不写顶层声明也能跑，Docker 会自动创建 `项目名_mysql-data`。手动写是为了控制名字。

### 管理命令

| 指令 | 作用 |
|------|------|
| `sudo docker compose up -d` | 启动 |
| `sudo docker compose ps` | 看状态 |
| `sudo docker compose stop` | 停止所有服务 |
| `sudo docker compose down` | 停止 + 删除容器（Volume 不删） |
| `sudo docker compose down -v` | 停止 + 删除容器 + 删除 Volume（数据也删了） |

## 十四、Networks + depends_on

### 上帝视角

- **networks**：手动给容器间通信的网络命名，同一网络里的容器通过名字互访
- **depends_on**：控制启动顺序，MySQL 先启，Java 应用后启

### 完整 compose.yml

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: my-mysql
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_DATABASE: myapp
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - app-net               # 加入 app-net 网络

  redis:
    image: redis:7.0
    container_name: my-redis
    ports:
      - "6379:6379"
    networks:
      - app-net               # 同一个网络

  app:
    image: hello-docker:1.0
    container_name: my-app
    depends_on:               # 控制启动顺序：mysql → redis → app
      - mysql                 # 里面放的是"服务名"，不是镜像名
      - redis
    networks:
      - app-net

networks:                     # 声明网络
  app-net:

volumes:
  mysql-data:
```

### 关键点

- `depends_on` 放在"谁最后启动"的服务里
- `depends_on` 里写的是**服务名**（`services:` 下的 key），不是镜像名（`image:` 后面的东西）
- compose 自动创建网络并让所有服务互通，手动 `networks:` 只是给网络起个可控的名字
