# Git 学习笔记

> 基于实操问答整理，覆盖日常开发 90% 操作

---

## 一、核心概念

### 三个区域

```
工作区          暂存区            仓库
(你的文件)  →  git add  →  (准备提交)  →  git commit  →  永久记录
```

| 区域 | 是什么 | 对应命令 |
|------|--------|---------|
| 工作区 | 你直接编辑的文件 | 编辑器修改 |
| 暂存区 | 勾选"这次提交包含哪些文件" | `git add` |
| 仓库 | 永久快照，有唯一 ID | `git commit` |

### Git 不帮你改文件

Git 只记录你改了什么。修改内容用编辑器（VSCode/IDEA/记事本），Git 不管。

---

## 二、基本工作流

```bash
# 初始化仓库
git init

# 看当前状态（最常用命令，随时敲）
git status

# 勾选要提交的文件
git add <文件名>

# 提交
git commit -m "一句话描述做了什么"

# 看历史
git log --oneline
```

### 什么时候 add，什么时候 commit

- **add**：改完一个有意义的小单元就 add。一天可能 add 十几次
- **commit**：能用一句话说清楚这次做了什么，就 commit。一天 3-5 次
- 每次 commit 前先 `git status`，确认暂存区只有这次想提交的东西

### 不该提交的文件

```bash
# 该提交的              # 不该提交的
.java / .py             .class / .jar
pom.xml / build.gradle  target/ / dist/
README.md               node_modules/
配置文件                  .idea/（IDE 个人配置）
```

不该提交的写在 `.gitignore` 文件里，Git 自动忽略。

---

## 三、分支

### 分支是什么

在提交历史上分叉，走一条新路。两条路互不影响。

```
          dev → ○ → ○
         /
master ○ → ○ → ○
```

### 为什么用分支

- 写新功能写到一半，线上出 bug → 开新分支修 bug，功能代码不受影响
- 多人协作，各自在自己的分支上开发，最后合并
- 主分支永远保持可发布状态

### 分支命令

```bash
git branch                 # 看所有分支，* 是当前分支
git checkout -b <分支名>    # 创建新分支并切过去
git checkout <分支名>       # 切换到已有分支
git merge <分支名>          # 把该分支合并到当前分支
git branch -d <分支名>      # 删除分支
```

### 合并后分支还在吗

还在。合并是把改动抄过来，分支本身不自动删除。删分支要手动 `git branch -d`。

---

## 四、冲突

### 什么时候冲突

两个分支改了同一个文件的同一行，合并时 Git 不知道选哪个。

### 冲突标记

```
<<<<<<< HEAD
当前分支的内容
=======
合进来的分支的内容
>>>>>>> 分支名
```

### 怎么解决

直接编辑文件，删掉 `<<<<<<<`、`=======`、`>>>>>>>`，留下想要的内容，保存，add，commit。

---

## 五、stash（暂存）

### 场景

代码改了一半，没法 commit，但要紧急切分支修 bug。

### 原理

把**当前分支**未 commit 的改动暂存到抽屉 → 工作区恢复干净 → 切分支 → 修完 → 切回来 → 取出继续改。

**stash 只暂存当前分支的改动，不影响其他分支。**

### 命令

```bash
git stash          # 暂存改动，工作区恢复干净
git stash pop      # 取出最近一次 stash 并删除
git stash apply    # 取出但不删除（想重复用时用这个）
git stash list     # 看所有 stash 记录
```

| `pop` | `apply` |
|-------|---------|
| 取出改动 + 删掉 stash | 取出改动，stash 还留着 |
| 正常情况用这个 | 想把同一份改动应用到多个分支 |

---

## 六、GitHub 远程仓库

### 为什么用

- 代码不丢（电脑坏了还在云端）
- 别人能看、能协作（面试官看你的 GitHub）
- 团队协作的基础

### 首次连接

```bash
git remote add origin <仓库地址>    # 告诉本地：远程仓库地址叫 origin
git push -u origin main            # 推送并绑定，以后直接 git push
```

GitHub 默认分支叫 `main`，旧项目可能叫 `master`。`git branch -M main` 可以把本地的 `master` 改名。

### 日常推送

```bash
git add <文件>
git commit -m "xxx"
git push                # 因为 push -u 绑定过，不用打全
```

### 拉取别人改动

```bash
git pull
```

### 代理问题

国内连 GitHub 可能超时。配代理：

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

---

## 七、命令速查

| 命令 | 作用 |
|------|------|
| `git init` | 初始化仓库 |
| `git status` | 查看状态（随时敲） |
| `git add <文件>` | 加入暂存区 |
| `git commit -m "xxx"` | 提交 |
| `git log --oneline` | 简洁版历史 |
| `git checkout -b <分支>` | 创建并切换分支 |
| `git checkout <分支>` | 切换已有分支 |
| `git merge <分支>` | 合并到当前分支 |
| `git branch` | 列出所有分支 |
| `git branch -d <分支>` | 删除分支 |
| `git stash` | 暂存未提交的改动 |
| `git stash pop` | 取出暂存 |
| `git restore <文件>` | 丢弃文件修改 |
| `git reset HEAD <文件>` | 从暂存区撤掉 |
| `git remote add origin <url>` | 关联远程仓库 |
| `git push` | 推送到远程 |
| `git pull` | 拉取远程改动 |
