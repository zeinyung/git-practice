# Linux 深入学习笔记

> WSL Ubuntu 实操，覆盖排错三件套（curl/grep/awk）+ shell 脚本

---

## 一、curl — 命令行 HTTP 客户端

### 上帝视角

curl = 命令行里的 Postman。测试 API、检查服务是否活着、下载文件。

### curl vs ss — 各看各的

| | `ss -tlnp` | `curl` |
|------|-----------|--------|
| 干什么 | 看**自己**这台机器上哪些端口开着、谁在用 | 从**外面**访问某个端口，看它有没有正常响应 |
| 类比 | 查公寓楼里住了谁 | 敲门，看有没有人应 |

### 已学命令

| 命令 | 作用 | 什么时候用 |
|------|------|-----------|
| `curl http://www.baidu.com` | 发 GET 请求，返回网页内容 | 测试网站是否可达 |
| `curl -v http://www.baidu.com 2>&1 \| head -40` | 显示请求头+响应头完整对话（`>` 发出的 `<` 收到的），`2>&1 \| head -40` 限制只看前 40 行 | 排查 HTTP 层问题，防止刷屏 |
| `curl -I http://www.baidu.com` | 只发 HEAD 请求，不看内容 | 检查服务是否活着，看一眼 200 OK 就行 |
| `curl -s -o /dev/null -w "%{http_code}\n" URL` | 只输出状态码 | 脚本里判断服务健康 |

### curl 选项速查

| 选项 | 含义 | 什么时候用 |
|------|------|-----------|
| `-v` | verbose，显示完整请求/响应对话 | 排查 HTTP 问题 |
| `-I` | 只发 HEAD 请求 | 检查服务存活 |
| `-X POST` | 指定 HTTP 方法 | 调 API（GET/POST/PUT/DELETE） |
| `-H` | 加请求头 | 设 Content-Type、token |
| `-d` | 发送数据（请求体） | POST/PUT 请求 |
| `-o` | 输出到文件 | 下载文件、静默输出 |
| `-s` | 静默模式（不显示进度条） | 脚本里用 |
| `-w` | 自定义输出格式 | 只看状态码 |
| `-O` | 下载文件（保留远程文件名） | 下载安装包 |

---

## 二、grep — 文件内容搜索

### 一句话区分 grep 和 awk

- **grep**：筛选行 — "我只要含 ERROR 的那些行"
- **awk**：提取列 — "我只要每行的第 2 列"

### 上帝视角

在文件里搜关键词。不用打开文件，一行命令出结果。

### 工作场景

| 场景 | 命令 |
|------|------|
| 线上排错，找所有 ERROR 日志 | `grep -r "ERROR" /var/log/myapp/` |
| 查数据库密码写在哪个配置文件 | `grep -r "password" /etc/myapp/` |
| 从几百个进程里找 Java 的 | `ps -ef \| grep java` |
| 排除 grep 自己那行 | `ps -ef \| grep java \| grep -v grep` |
| 看 ERROR 的上下文（前后各 3 行） | `grep -C 3 "ERROR" app.log` |
| 显示行号方便定位 | `grep -n "ERROR" app.log` |

### 核心选项

| 选项 | 含义 | 什么时候用 |
|------|------|-----------|
| `-r` | 递归搜目录（含所有子目录） | 日志按天归档在子目录里，不加 -r 会漏 |
| `-n` | 显示行号 | 找到后要去编辑器里定位 |
| `-C 3` | 显示匹配行前后各 3 行 | 看上下文，理解错误怎么发生的 |
| `-v` | 反向过滤，排除含关键词的行 | 去掉 grep 自己那行、排除噪音 |
| `-i` | 忽略大小写 | 不确定是 Error 还是 error |

### Q：为什么要 -r（递归）？

```
logs/
├── app.log              ← grep "ERROR" logs 能搜到这个
└── archive/
    ├── old.log          ← 不加 -r 搜不到这个（子目录被跳过）
    └── old2.log         ← 不加 -r 搜不到这个
```

生产环境日志按天归档在子目录里，不加 `-r` 会漏掉昨天以前的 ERROR。

### Q：为什么需要 `grep -v grep`？

因为 `grep java` 这条命令本身也是一个进程，`ps -ef` 里会多出一行：

```
yzx  5678  ...  grep --color=auto java    ← 这条是 grep 自己，不是真的 Java 进程
```

`grep -v grep` 把这行过滤掉，只保留真实的 Java 进程行。

---

## 三、awk — 从输出里提取列

### 上帝视角

很多命令输出是表格形式，awk 帮你从每行里只拿某几列。

```
一行输出：  yzx  1234  0.1  java
            $1   $2    $3   $4
```

### 工作场景

| 场景 | 命令 |
|------|------|
| 拿到某端口对应的 PID | `sudo ss -tlnp \| grep 8080 \| awk '{print $6}'` |
| Docker 容器列表只显示容器 ID | `docker ps \| awk '{print $1}'` |
| 日志里只提取时间和错误信息 | `grep ERROR app.log \| awk '{print $1, $2, $NF}'` |

### 必会用法

```bash
awk '{print $1}'         # 第 1 列
awk '{print $2}'         # 第 2 列
awk '{print $1, $2}'     # 第 1 列 + 第 2 列
awk '{print $NF}'        # 最后一列（NF = Number of Fields）
```

### Q：为什么 `$6` 就是 PID 那串？

```
sudo ss -tlnp 输出的一行：
LISTEN  0  4096  127.0.0.54  53  0.0.0.0:*  users:(("systemd-resolve",pid=80,fd=19))
 $1     $2  $3     $4       $5     $6                    $7
```

---

## 四、mkdir -p

### Q：`mkdir -p` 什么意思，不能直接 `mkdir logs/archive` 吗？

不能。如果父目录不存在，不加 `-p` 会报错：

```bash
mkdir logs/archive        # logs 不存在 → 报错
mkdir -p logs/archive     # logs 不存在 → 先创建 logs，再创建 archive → 成功
```

| | 无 -p | 有 -p |
|------|--------|--------|
| 父目录已存在 | ✅ | ✅ |
| 父目录不存在 | ❌ 报错 | ✅ 自动补建 |
| 目录已存在 | ❌ 报错 | ✅ 静默跳过 |

工作中永远用 `mkdir -p`。

---

## 五、shell 脚本

### 上帝视角

**把终端里手动敲的一串命令写进文件，以后只敲一个命令自动执行。** 省时间 + 不出错。

### 创建脚本五步

```bash
# 1. 创建脚本文件
cat > myscript.sh << 'EOF'

# 2. 声明解释器（必须写在第一行）
#!/bin/bash

# 3. 写要执行的命令
echo "你好"

# 4. 结束标记（EOF 可以是任何单词，如 EFD、USB、END）
EOF

# 5. 加执行权限
chmod +x myscript.sh

# 6. 运行
./myscript.sh
```

**每一步都不能省：** 不加 `#!/bin/bash` 可能语法出错，不加 `chmod +x` 会 Permission denied。

### Q：每次都要写 `#!/bin/bash` 吗？

要。告诉系统用 bash 解释这个脚本。不写也能跑，但可能用不同的 shell，语法差异出 bug。

### Q：`cat > 文件 << 'EOF'` 是什么意思？

```
cat              → 读文件内容
   > 文件        → 把内容写入文件
      << 'EOF'  → 从键盘读入多行，直到遇到 EOF 停止
```

= 创建一个文件，把多行内容写进去，EOF 是结束标记（可以是任何单词）。

### Q：`chmod +x` 什么意思？

给文件加执行权限。Linux 新建文件默认只能读不能写不能执行，不加 +x 直接运行会报 Permission denied。

### `$` 的两种用法

| 语法 | 含义 | 例子 |
|------|------|------|
| `$变量名` 或 `${变量名}` | 取出变量的值，嵌到字符串里 | `echo "你好，${name}"` |
| `$(命令)` | 执行命令，把命令结果嵌到字符串里 | `echo "当前时间：$(date)"` |

```bash
name="张三"
echo $name               # $变量名 = 取出变量存的文字 → 张三
echo ${name}              # ${变量名} 更安全，不会跟后面字符粘连

echo $(date)              # $(命令) = 执行命令，结果嵌进来 → Wed Jun 24 10:02:15 CST 2026
```

### 变量 + 判断

```bash
#!/bin/bash
PID=$(ps -ef | grep hello | grep -v grep | awk '{print $2}')

if [ -n "$PID" ]; then          # -n = 变量非空吗？
    echo "找到了，PID 是：$PID"
else
    echo "没找到"
fi
```

| 判断 | 含义 |
|------|------|
| `[ -f 文件 ]` | 文件存在吗 |
| `[ -d 目录 ]` | 目录存在吗 |
| `[ -n "字符串" ]` | 字符串非空吗（not empty） |
| `[ -z "字符串" ]` | 字符串为空吗（zero） |
| `[ "a" = "b" ]` | 两个字符串相等吗 |

注意：`[` 和 `]` 两边必须留空格。`[条件]` 是错的，`[ 条件 ]` 才对。

### PID 提取管道的完整逻辑

```bash
PID=$(ps -ef | grep hello | grep -v grep | awk '{print $2}')
```

```
ps -ef                         → 列出所有进程
    | grep hello               → 只保留含 "hello" 的行
    | grep -v grep             → 剔除 "grep hello" 命令本身（它也在进程列表里）
    | awk '{print $2}'         → 只输出第 2 列（PID）
```

---

## 六、ss / netstat — 看端口

### 上帝视角

看自己这台机器上哪些端口开着、哪个进程占的。

### 工作中什么时候用

- 启动服务报 `Port 8080 already in use` → 查是谁占了
- 巡检：检查该开的服务（MySQL 3306/Redis 6379）是否在监听

```bash
sudo ss -tlnp | grep 8080
```

### 理解：机器 / 服务 / 端口

```
一台云服务器 = 一栋公寓楼（IP 地址 = 门牌号）
  3306 端口 → MySQL 住这个房间
  6379 端口 → Redis 住这个房间
  8080 端口 → Spring Boot 住这个房间
```

`curl` 和 `ss` 的区别：

| | `ss -tlnp` | `curl` |
|------|-----------|--------|
| 干什么 | 看自己机器上哪些端口开着 | 从外面访问某个端口，看它有没有正常响应 |
| 类比 | 查公寓里住了谁 | 敲门，看有没有人应 |

---

## 七、sed — 只学一行

```bash
echo "hello world" | sed 's/world/linux/'   # 把 world 替换成 linux
echo "aaa bbb aaa" | sed 's/aaa/xxx/g'      # /g = 全局替换
```

就 `s/A/B/g` 够用。多行编辑用 Python，不用 sed。

---

## 八、wget — 不学

curl 已经能下载文件（`curl -O`），功能重叠。不花时间。

---

## 九、命令速查（深入学习部分）

| 命令 | 作用 |
|------|------|
| `curl -v URL` | 完整 HTTP 对话 |
| `curl -I URL` | 只拿响应头，检查服务存活 |
| `curl -s -o /dev/null -w "%{http_code}\n" URL` | 只输出状态码 |
| `curl -X POST URL -H "头" -d '数据'` | 发 POST 请求 |
| `grep -r "关键词" 目录` | 递归搜索 |
| `grep -C 3 "关键词" 文件` | 显示匹配行 + 上下文 |
| `grep -v "关键词"` | 反向过滤 |
| `awk '{print $N}'` | 提取第 N 列 |
| `awk '{print $NF}'` | 提取最后一列 |
| `mkdir -p 路径` | 递归创建目录 |
| `sudo ss -tlnp` | 看端口占用 |
| `sed 's/A/B/g'` | 全局替换文本 |
