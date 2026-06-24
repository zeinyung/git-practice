# Python 速成笔记（从 Java 迁移）

> 面向 Java 开发者的 Python 基础速览，覆盖 W1-W6 Agent 开发需要的语法

---

## 一、核心哲学

| | Java | Python |
|------|------|--------|
| 类型 | 静态类型：`int x = 10;` | 动态类型：`x = 10` |
| 编译 | 先编译再运行 | 解释执行，边跑边检查 |
| 变量 | "盒子"，定了类型不能改 | "标签"，贴在哪个值上就是哪个类型 |

```python
x = 10
x = "hello"     # 同一个变量，从数字变成字符串。Java 绝不允许
```

---

## 二、if / for — 没有括号，靠缩进

### if

```python
if age >= 18:
    print("成年")
else:
    print("未成年")
```

- 条件**不需要括号**，`:` 替代
- 代码块**不需要 `{}`**，缩进就是代码块
- 缩进是**语法**，不对直接 `IndentationError`

### for

```python
for i in range(5):          # 0,1,2,3,4（等于 Java for(int i=0; i<5; i++)）
    print(i)

for name in names:          # 等于 Java 增强 for (String name : names)
    print(name)
```

---

## 三、列表 & 字典

### 列表 = `ArrayList`

```python
names = ["张三", "李四", "王五"]
names.append("赵六")         # 追加
len(names)                   # 长度
```

### 字典 = `HashMap`

```python
user = {"name": "张三", "age": 25}
user["name"]                 # "张三" — key 不存在直接报错 KeyError
user.get("phone", "没填")     # "没填" — key 不存在返回默认值（**工作中永远用这个**）
```

### f-string = 字符串拼接

```python
f"你好，{name}"              # = Java "你好，" + name
f"明年你 {age + 1} 岁"       # 花括号里可以写任意表达式
```

---

## 四、函数

```python
def greet(name):
    """文档注释，相当于 Java 的 /** */"""
    return f"你好，{name}"
```

| | Java | Python |
|------|------|--------|
| 声明 | `public String greet(String name) {}` | `def greet(name):` |
| 返回类型 | 必须声明 | 省略，自动推断 |
| 参数类型 | 必须声明 | 省略 |
| private | ✅ 有 | ❌ 没有，约定 `_func()` 开头表示"别碰" |

---

## 五、类

```python
class User:
    # 构造函数
    def __init__(self, name, age):
        self.name = name           # 成员变量不用提前声明
        self.age = age             # self = Java 的 this

    # 方法：第一个参数必须是 self
    def greet(self):
        return f"我是{self.name}，今年{self.age}岁"

    # __str__ = Java 的 toString()
    def __str__(self):
        return f"User(name={self.name}, age={self.age})"


# 使用
u = User("张三", 25)               # 没有 new 关键字
print(u.greet())
```

| | Java | Python |
|------|------|--------|
| 构造 | `public User(String name)` | `def __init__(self, name)` |
| this | `this.name` | `self.name` |
| new | `new User(...)` | `User(...)` |
| toString | `toString()` | `__str__(self)` |
| 成员声明 | 类顶部声明 `private String name;` | **不需要**，`self.name = name` 那一刻自动创建 |

### self 必须显式写在参数列表第一个位置

```python
u.greet()        # 实际调用是 greet(u)，self 自动填了
```

---

## 六、模块导入

```python
import os                      # 导入整个模块（= import java.util.*）
from math import sqrt          # 导入模块里的一个函数（= import static java.lang.Math.sqrt）
import numpy as np             # 起别名
```

---

## 七、装饰器 ⭐

### 一句话解释

**装饰器 = `函数 = 处理器(函数)` 的简写。装饰器相当于将装饰的原函数作为参数上传到处理器**

```python
@add_log                     # say_hello = add_log(say_hello)
def say_hello(name):         # 这行之后 say_hello 已被加强
    ...
```
例：@log_before_after
```python
@log_before_after
def say_hello(name):
    return f"你好，{name}"
```
原函数 = say_hello，就是 : 下面那三行代码。它原本只会返回 "你好，张三"，别的什么都不会。

处理器 = log_before_after。它做的事：接收 say_hello，在外面裹一层"开始/结束"的打印，裹完了返回。

新东西 = 裹完之后吐出来的那个函数。它做的事：先打印 【开始】，再调用原来的 say_hello 拿结果，再打印 【结束】，最后返回结果。

### 从不用装饰器到用的完整演变

```python
# 第一步：普通函数
def say_hello(name):
    return f"你好，{name}"

# 第二步：一个函数接收另一个函数，返回加强版
def add_log(original_func):
    def new_func(name):
        print("【开始】")
        result = original_func(name)
        print("【结束】")
        return result
    return new_func

# 第三步：手动替换
say_hello = add_log(say_hello)   # ← 这行等 于 @add_log

# 第四步：简写
@add_log
def say_hello(name): ...
```

### 工作中会见到

```python
@tool                          # LangChain：注册为 Agent 工具
def search(query: str): ...

@app.get("/agent/chat")        # FastAPI：绑定 HTTP 路由
def chat(): ...

@graph.add_node("search")       # LangGraph：注册为图节点
def search_node(state): ...
```

### Java 类比

装饰器的效果和 Spring 注解一模一样 — **声明一下，框架自动处理**。区别是 Python 在运行时真的替换了函数。

```java
@GetMapping("/api/users")       // 声明绑定路由
@Transactional                  // 声明事务包裹
```

---

## 八、`*args` 和 `**kwargs`

```python
*args      → 多余的位置参数，打包成元组
**kwargs   → 多余的关键字参数，打包成字典
```

```python
def demo(*args, **kwargs):
    print(f"位置: {args}")        # 位置: (1, 2, 3)
    print(f"关键字: {kwargs}")    # 关键字: {'name': '张三', 'age': 25}

demo(1, 2, 3, name="张三", age=25)
```

### 为什么装饰器里必须写 `*args, **kwargs`

因为装饰器不知道被装饰的函数要几个参数 — `wrapper(*args, **kwargs)` = "不管来多少、什么类型，我全接住，原样传进去"。

---

## 九、Python 的"魔法方法"（双下划线）

```python
__init__      # 构造函数
__str__       # 等于 Java toString()
__name__      # 函数自带的属性，存函数名
```

---

## 十、Java 程序员踩坑清单

1. **缩进不能混用空格和 Tab** → `IndentationError`。PyCharm 默认把 Tab 转 4 空格
2. **`==` 比内容，`is` 比是不是同一个对象**（和 Java 相反）
3. **赋值不复制**：`b = a` 之后 `b.append(1)` 也会改 `a`
4. **`dict["key"]` 不存在直接报错** → 永远用 `dict.get("key", 默认值)`
5. 没有 `private`：约定 `_variable` 前缀表示"内部用的，别动"
