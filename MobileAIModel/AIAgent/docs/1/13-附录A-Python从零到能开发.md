# 附录 A Python 从零到能开发（写给 Android 开发者）

## 一句话总结

如果你会 Kotlin/Java 写 Android，那你**已经会 90% 的编程了**——你缺的不是"编程能力"，而是三样具体的东西：**Python 的语法长什么样**（缩进代替大括号、动态类型、没有 `new`）、**Python 世界的 Gradle 叫什么**（答案是 `uv` + `pyproject.toml`）、以及**一个 Python 应用是怎么从空目录长成能被别人 `pip install` 的东西、再变成一个跑在服务器上的容器的**。这一章就把这三样按顺序补齐，全程只做一件事：**从空目录敲出一个叫 `notekit` 的笔记工具**，它同时是命令行程序和 Web API，带测试、带类型检查、能打包、能装到别人机器上、能塞进 Docker 跑起来。

---

## 本章你能学到什么

| # | 你会学到 | 学完能干什么 |
|---|---|---|
| 1 | **环境**：为什么系统自带的 `python3` 不能用、`uv` 是什么、虚拟环境到底虚拟了什么 | 在任何一台新机器上 5 分钟搭好一个干净的 Python 项目 |
| 2 | **基础语法**：动态类型、缩进、真值、切片、推导式、`==` vs `is` | 读懂任何一段 Python 代码，不再被"这行为什么不报错"卡住 |
| 3 | **函数与类**：四种参数传法、`self`、魔术方法、`@property`、`@dataclass`、鸭子类型 | 把 Kotlin 的 `data class` / `interface` / `companion object` 一一翻译过来 |
| 4 | **异常 / 文件 / JSON / 日志**：`try/except/else/finally`、`pathlib`、`with`、`logging` | 写出不会静默吞错、不会写死路径的代码 |
| 5 | **模块与包**：`import` 到底干了什么、`__init__.py`、相对导入、`if __name__ == "__main__"` | 彻底搞定 `ImportError`，这是新手最大的一道坎 |
| 6 | **装饰器 / 生成器 / 闭包**：`@` 是语法糖、`yield` 是暂停键 | 看懂 FastAPI、pytest、LangChain 里满屏的 `@` |
| 7 | **类型提示与 mypy**：为什么写了类型还是不检查 | 用 mypy 把 Kotlin 编译器帮你抓的那类错重新抓回来 |
| 8 | **并发**：GIL、线程、进程、`async/await` | 亲手测出"多线程为什么在 Python 里不加速"，并知道三者各用在哪 |
| 9 | **主要库**：pydantic / typer / rich / httpx / FastAPI / pytest / ruff | 拿到一个 Python 项目能看懂它的依赖表 |
| 10 | **项目实战**：从空目录敲出 `notekit`（模型 → 存储 → CLI → HTTP API → 测试） | 独立写出一个结构正确、有测试、能交付的 Python 应用 |
| 11 | **发布**：`uv build` 出 wheel、装进别人的环境、`uv tool install` 变成全局命令 | 把自己的工具发出去给别人用 |
| 12 | **部署**：`uvicorn --workers`、多阶段 Dockerfile、compose、健康检查、数据卷 | 把 Web 服务以生产标准跑起来，不用 root、密钥不进镜像 |
| 13 | **调试与踩坑**：读 Traceback、七种常见报错、28 条坑的总表 | 遇到报错能自己定位，而不是复制到搜索框 |

**动手部分**：**16 个 ✅ Checkpoint**，全部在本机真实跑过，**每个 Checkpoint 里贴的输出都是实际跑出来的**（包括那些故意让它报错的）。除第十二节的 Docker 部分需要装 Docker Desktop 外，全程离线、不需要任何 API Key。预计 2~3 天。

> **这一章和前十二章的关系**：前面十二章教的是"智能体的原理"，代码用的都是 Python；这一章补的是"你敲那些代码时用到的每一个 Python 知识点"。如果你在读前面章节时被某段语法卡住过，可以直接来这里按目录查。

---

## 先看一张全局对照表：你的 Android 知识怎么迁移

**别从零开始学，从"翻译"开始学。** 下面这张表里的每一行，后面都会展开。

| 你在 Android 里熟悉的 | Python 里对应的 | 关键区别（这是最容易踩的地方） |
|---|---|---|
| Java / Kotlin | Python 3.12 | **不编译**，直接解释执行；写错类型要到运行那一行才炸 |
| `{ }` 划分代码块 | **缩进**（4 个空格） | 缩进错了就是语法错误，不是风格问题 |
| `String name = "x";` | `name = "x"` | 不写类型、不写分号；同一个变量可以先存数字再存字符串 |
| `new Note(...)` | `Note(...)` | **没有 `new`** |
| `this` | `self` | 必须**显式**写成方法的第一个参数 |
| `data class Note(...)` | `@dataclass class Note:` | 或者用 pydantic 的 `BaseModel`（多了运行时校验） |
| `interface Speaker` | `Protocol` / 直接鸭子类型 | Python 里**不实现接口也能用**，只要方法名对得上 |
| `companion object` 里的常量 | 类属性（写在 class 下、方法外） | 所有实例共享 |
| `getter/setter` | `@property` | 用起来像属性，实际是方法 |
| `Long`（64 位，会溢出） | `int` | **没有上限**，永远不溢出 |
| `try/catch/finally` | `try/except/finally` | 多了一个 `else`（没出错才走） |
| `null` | `None` | 判断永远写 `x is None`，不写 `x == None` |
| `Gradle` + `build.gradle.kts` | **`uv`** + `pyproject.toml` | 概念几乎一一对应，见下一节 |
| `gradle/libs.versions.toml` + `gradle.lockfile` | `uv.lock` | 锁定精确版本 + 哈希 |
| `implementation` / `testImplementation` | `dependencies` / `dependency-groups.dev` | 一样是"运行时依赖"和"只在测试用的依赖" |
| `.aar` / `.apk` | `.whl`（wheel） | wheel 就是一个 zip，改个后缀就能解开看 |
| Maven Central | **PyPI** (pypi.org) | 发布用 `uv publish` |
| `JUnit` | **pytest** | 断言直接写 `assert`，不用 `assertEquals` |
| `ktlint` / `detekt` | **ruff** | 一个工具同时干 lint + format，快到几乎无感 |
| Kotlin 编译器的类型检查 | **mypy**（要单独装、单独跑） | **Python 运行时完全不检查类型**，不跑 mypy 就等于没写 |
| `Application` 类 / 四大组件 | 没有对应物 | Python 应用就是"一个入口函数"，别找框架 |
| `AndroidManifest.xml` | `[project.scripts]` | 声明"哪个命令对应哪个函数" |

> **最重要的一行是倒数第三行**：你在 Android 里靠编译器兜住的一大类错误（类型不匹配、拼错方法名），在 Python 里默认**没人管**。这不是 Python 差，而是这部分工作被移交给了 mypy 和测试。**你必须把它们捡起来**，否则会写出"跑一半才炸"的代码。这也是第 7 节和第 10.12 节存在的原因。

---
## 一、环境：先把地基打对（90% 的新手劝退都发生在这一步）

### 1.1 是什么：系统自带的 Python 为什么不能用

Mac 和 Linux 都自带 Python。**但自带的那个版本通常很老。** 在我这台机器上：

```bash
/usr/bin/python3 --version
```

输出：

```
Python 3.9.6
```

3.9 有多老？前面十二章里大量用到的写法它**直接语法错误**。比如 `match` 语句（相当于 Kotlin 的 `when`）：

```bash
/usr/bin/python3 -c "
x=1
match x:
    case 1: print('ok')
"
```

输出：

```
  File "<string>", line 3
    match x:
          ^
SyntaxError: invalid syntax
```

注意这是 **SyntaxError**，不是"功能不支持"——解释器连这段代码都读不懂。同类的还有 `str | None`（3.10+）、`itertools.batched`（3.12+）、`def f[T](...)` 泛型语法（3.12+）。

**所以第一条规则：永远不要用系统自带的 `python3` 开发，也永远不要往它里面 `pip install` 任何东西**（那是操作系统自己在用的 Python，装乱了会影响系统工具）。

### 1.2 是什么：虚拟环境到底"虚拟"了什么

Android 里每个项目的依赖是隔离的——A 项目用 OkHttp 4.9，B 项目用 4.12，互不影响，因为依赖是**按项目**下载到 Gradle 缓存再链进去的。

Python 早期不是这样：`pip install` 默认装到**全局**。于是 A 项目要 `pydantic 1.x`、B 项目要 `pydantic 2.x` 时就打起来了，这个现象有个专门的外号叫"依赖地狱"。

**虚拟环境（virtual environment）** 就是解决办法：在项目目录里建一个 `.venv/` 文件夹，里面放一份独立的 Python 解释器（其实是符号链接）和一份独立的 `site-packages`（第三方包的存放处）。**"激活"虚拟环境的本质，就是把 `.venv/bin` 插到 `PATH` 最前面**，让你敲的 `python` 命中项目里的那个，而不是系统那个。

一句话类比：**`.venv/` 就是这个项目专属的 Gradle 缓存 + 专属的 JDK。**

### 1.3 怎么用：`uv` —— Python 世界的 Gradle

历史上 Python 的工具链是碎的：`pyenv` 管版本、`venv` 建虚拟环境、`pip` 装包、`pip-tools`/`poetry` 锁版本、`build`/`twine` 打包发布。每个都要单独学。

**`uv`（Astral 出的，用 Rust 写的）把这些全合成了一个命令**，而且快得离谱（下面所有输出里的 "Installed 5 packages in 12ms" 都是真实耗时）。它的角色就是 Gradle：

| 你想干的事 | Gradle | uv |
|---|---|---|
| 装指定版本的语言运行时 | `gradle-wrapper.properties` 里定 JDK | `uv python install 3.12` / `.python-version` |
| 新建项目骨架 | Android Studio 模板 | `uv init --package myapp` |
| 加一个依赖 | 编辑 `build.gradle.kts` 再 sync | `uv add httpx` |
| 加一个只在测试用的依赖 | `testImplementation` | `uv add --dev pytest` |
| 按锁文件精确还原 | `--offline` + lockfile | `uv sync --frozen` |
| 看依赖树 | `gradle dependencies` | `uv tree` |
| 跑项目里的命令 | `gradle run` | `uv run <命令>` |
| 打包 | `assembleRelease` → `.apk`/`.aar` | `uv build` → `.whl` |
| 装一个全局 CLI 工具 | —— | `uv tool install <包>` |

**安装 uv**（只需一次，装完就有 `uv` 命令）：

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
# 或者用 Homebrew
brew install uv
```

> **安全提醒**：`curl | sh` 这种写法是把网上的脚本直接交给 shell 执行。这里之所以可以接受，是因为 astral.sh 是 uv 官方域名、走 HTTPS。**但请养成习惯：任何来源不明的 `curl | sh` 都不要执行**，至少先 `curl -LsSf <url> -o install.sh` 下载下来看一眼。

验证：

```bash
uv --version
```

输出：

```
uv 0.11.32 (3010295ae 2026-07-23 aarch64-apple-darwin)
```

### 1.4 相关知识：`uv run` 和"激活虚拟环境"的关系

传统流程要三步：

```bash
python3 -m venv .venv
source .venv/bin/activate     # 激活，提示符前面会多一个 (.venv)
pip install httpx
```

**用 uv 的话，你几乎不需要手动激活。** `uv run <命令>` 会自动：确认 `.venv` 存在（不存在就建）→ 确认依赖和 `uv.lock` 一致（不一致就装）→ 在这个环境里执行命令。

```bash
uv run python -c "import sys; print(sys.executable)"
```

这条会打印出 `.venv` 里那个 python 的路径，而不是系统的。**本章后面所有命令都用 `uv run` 前缀**，这样你不会遇到"我明明装了包却 import 不到"这个经典问题。

> **一个真实会遇到的警告**：如果你的 shell 里已经激活了**别的**虚拟环境（比如你在另一个项目的 `.venv` 里），`uv run` 会提示 `VIRTUAL_ENV=... does not match the project environment path .venv and will be ignored`。它只是提醒，不影响结果；想让它消失就先 `deactivate`，或者在命令前加 `unset VIRTUAL_ENV`。

### 1.5 ✅ Checkpoint 1：环境跑通

```bash
uv python install 3.12
uv run --python 3.12 python -c "import sys; print(sys.version)"
uv run --python 3.12 python -c "import sys; print(sys.executable)"
```

我这里的真实输出：

```
3.12.13 (main, Mar  3 2026, 12:39:30) [Clang 21.0.0 (clang-2100.0.123.102)]
/private/tmp/verify/py/.venv/bin/python3
```

两点要确认：**第一行是 3.12 而不是 3.9**；**第二行指向项目目录里的 `.venv`，不是 `/usr/bin`**。只要这一步成功，后面所有代码都能跑。

（你的版本号可能是 3.12.x 的其它小版本，路径也会是你自己的目录，这都正常。）

---
## 二、基础语法：把 Kotlin 翻译成 Python

**这一节请边看边敲。** 新建一个文件 `a_basics.py`，一段一段加进去，每加一段就跑 `uv run --python 3.12 python a_basics.py` 看一眼。

### 2.1 缩进代替大括号

Python 用**缩进**表示代码块，冒号 `:` 表示"下面要缩进了"。

```python
# Kotlin:  if (x > 0) { println("正") } else { println("负") }
if x > 0:
    print("正")
else:
    print("负")
```

规则只有两条，但都是硬规则：**同一个块里缩进必须完全一致**（统一用 4 个空格，别用 Tab）；**缩进错了是 `IndentationError`，程序根本跑不起来**。

好处是你再也不用为大括号放哪儿吵架了；代价是复制粘贴代码时要小心。

### 2.2 动态类型：变量是"标签"不是"盒子"

```python
x = 10
print(type(x).__name__, x)
x = "十"
print(type(x).__name__, x)
```

输出：

```
int 10
str 十
```

在 Kotlin 里 `var x: Int = 10` 之后 `x = "十"` 是编译错误。Python 里完全合法。

**心智模型的差别很重要**：Kotlin 的变量像一个**贴了类型标签的盒子**，只能装那种东西；Python 的变量像一张**便利贴**，你可以把它从一个对象上撕下来贴到另一个对象上。类型属于**对象**，不属于变量名。

**这带来的实际后果**：拼错变量名、传错类型，都不会在"编译期"被发现（Python 没有编译期），只在执行到那一行时才炸。这就是为什么第 7 节的 mypy 是必需品而不是奢侈品。

### 2.3 数字：`int` 没有上限

```python
big = 2 ** 200
print(big)
print("Kotlin 的 Long 最大值:", 2 ** 63 - 1)
```

输出：

```
1606938044258990275541962092341162602522202993782792835301376
Kotlin 的 Long 最大值: 9223372036854775807
```

Kotlin 的 `Long` 是 64 位，超了就**静默溢出**变成负数——这是金融、计数场景的经典 bug 来源。Python 的 `int` 会自动扩展成任意精度，**永远不溢出**（代价是大数运算比 C 慢，但正确性优先）。

除法和取模有两个坑：

```python
print(7 / 2, 7 // 2, 7 % 2, 7 ** 2)
print(-7 // 2, -7 % 2)   # 注意和 Java 不一样
```

输出：

```
3.5 3 1 49
-4 1
```

- `/` **永远返回浮点数**，`7 / 2` 是 `3.5` 不是 `3`。要整除得用 `//`。这和 Java 反过来，是从 Java 转过来最常见的第一个 bug。
- `-7 // 2` 是 **-4**（向下取整，不是向零截断），`-7 % 2` 是 **1**（结果符号跟除数）。Java 里分别是 `-3` 和 `-1`。**做取模索引（哈希分桶、环形缓冲）时这个差别会真的咬人。**
- `**` 是幂运算，没有 `Math.pow` 那么啰嗦。

### 2.4 字符串：f-string 是唯一你需要记的

```python
name, score = "小林", 91.456
print(f"{name} 得了 {score:.1f} 分, 名字长度 {len(name)}")
print(f"{score=}")
```

输出：

```
小林 得了 91.5 分, 名字长度 2
score=91.456
```

`f"..."` 前面那个 `f` 表示"里面的 `{}` 要求值"，和 Kotlin 的 `"$name"` 一个意思，但更强：`{}` 里可以写任意表达式（`{len(name)}`），`:.1f` 是格式化（保留 1 位小数），`{score=}` 是调试神器——**自动把变量名和值一起打出来**。

字符串是**不可变**的，和 Kotlin 一样：

```python
t = "abc"
print(t.upper(), t)          # upper 返回新串，原串没变
print("拼接大量字符串要用 join:", "-".join(["a", "b", "c"]))
```

输出：

```
ABC abc
拼接大量字符串要用 join: a-b-c
```

**在循环里用 `+=` 拼几万次字符串会很慢**（每次都新建一个），正确做法是先收集到列表再 `"".join(...)`——就是 Kotlin 里用 `StringBuilder` 的那个理由。

### 2.5 真值判断：空的东西都是假

这是 Python 和 Kotlin 差异最大、也最容易写出 bug 的地方。

```python
for v in [0, 1, "", "a", [], [0], {}, None, 0.0]:
    print(f"  bool({v!r:>6}) = {bool(v)}")
```

输出：

```
  bool(     0) = False
  bool(     1) = True
  bool(    '') = False
  bool(   'a') = True
  bool(    []) = False
  bool(   [0]) = True
  bool(    {}) = False
  bool(  None) = False
  bool(   0.0) = False
```

规律：**数字 0、空字符串、空列表、空字典、空元组、`None` 都是假，其它都是真。** 注意 `[0]` 是**真**——它是"装了一个元素的列表"，非空。

所以 Python 里判空可以直接写：

```python
if not items:          # 而不是 if len(items) == 0
    print("空的")
```

**但这也是个陷阱**：`if not count:` 在 `count == 0` 时会进去，而你可能只想判断"没传值"。**要区分"0"和"没有"，必须写 `if count is None:`。** 这个坑在第 10 节的 `NoteUpdate` 里会以真实形态出现。

### 2.6 `==` 和 `is`：值相等 vs 同一个对象

```python
a = [1, 2]
b = [1, 2]
print("a == b:", a == b, " a is b:", a is b)
c = a
print("c is a:", c is a)
```

输出：

```
a == b: True  a is b: False
c is a: True
```

`==` 比较**内容**（相当于 Kotlin 的 `==`/`equals`），`is` 比较**是不是同一个对象**（相当于 Kotlin 的 `===`）。

**记住一条铁律：判断值用 `==`，`is` 实践中只用来写 `x is None`。** 为什么这么绝对？第 12 节有一个亲手跑出来的例子，会展示 `is` 在整数上的行为有多不可预测。

### 2.7 切片：比 `substring` 好用一百倍

```python
s = "0123456789"
print(s[2:5], s[:3], s[-3:], s[::2], s[::-1])
nums = list(range(10))
print(nums[2:5], nums[::3])
```

输出：

```
234 012 789 02468 9876543210
[2, 3, 4] [0, 3, 6, 9]
```

格式是 `[起点:终点:步长]`，**含头不含尾**（`s[2:5]` 取下标 2、3、4）。

- 省略起点 = 从头，省略终点 = 到尾
- **负数从右边数**：`s[-3:]` 是最后三个
- 步长 2 = 隔一个取一个；**步长 -1 = 反转**（`s[::-1]` 是最常见的反转写法）
- 列表和字符串通用；**列表的 `x[:]` 会生成一个浅拷贝**，第 12 节会用到

### 2.8 推导式：一行搞定 map/filter

```python
squares = [n * n for n in range(6)]
evens = [n for n in range(10) if n % 2 == 0]
table = {n: n * n for n in range(4)}
uniq = {c for c in "hello"}
print(squares)
print(evens)
print(table)
print(sorted(uniq))
```

输出：

```
[0, 1, 4, 9, 16, 25]
[0, 2, 4, 6, 8]
{0: 0, 1: 1, 2: 4, 3: 9}
['e', 'h', 'l', 'o']
```

读法是**从中间往两边读**：先看 `for n in range(6)`（对谁循环），再看 `if ...`（筛掉谁），最后看最前面的表达式（每个变成什么）。对照 Kotlin：

```kotlin
// Kotlin
val evens = (0 until 10).filter { it % 2 == 0 }
val squares = (0 until 6).map { it * it }
```

```python
# Python：一个语法覆盖 map + filter
evens = [n for n in range(10) if n % 2 == 0]
squares = [n * n for n in range(6)]
```

四种括号对应四种结果：`[...]` 列表、`{k: v ...}` 字典、`{...}` 集合、`(...)` **生成器**（第 6 节讲，行为完全不同）。

**别嵌套超过两层**——写得爽读着痛苦，超过两层就改成普通 for 循环。

### 2.9 元组：不可变的列表

```python
point = (3, 4)
px, py = point
print(px, py)
try:
    point[0] = 9
except TypeError as e:
    print("改元组报错:", e)
```

输出：

```
3 4
改元组报错: 'tuple' object does not support item assignment
```

**列表 `[]` 可改，元组 `()` 不可改。** 元组的两个主要用途：一是当"轻量数据结构"（坐标、键值对）；二是**多返回值**——`return a, b` 其实就是返回一个元组，接收时 `x, y = f()` 叫**解包**。这就是 Python 不需要 Kotlin `Pair`/`Triple` 的原因。

### 2.10 最经典的坑：可变默认参数

**这一条独立出来，因为它是 Python 最著名的陷阱，而且 Kotlin 里不存在对应现象。**

```python
def bad(item, box=[]):
    box.append(item)
    return box


print(bad("a"), bad("b"), bad("c"))


def good(item, box=None):
    if box is None:
        box = []
    box.append(item)
    return box


print(good("a"), good("b"), good("c"))
```

输出：

```
['a', 'b', 'c'] ['a', 'b', 'c'] ['a', 'b', 'c']
['a'] ['b'] ['c']
```

**看第一行**：三次独立调用，结果互相污染了。原因是**默认值只在函数「定义」的那一刻求值一次**，那个 `[]` 从此成为函数对象的一个属性，所有调用共用它。

**规则：默认值只能写不可变的东西**（数字、字符串、`None`、元组）。**要可变默认值，一律写 `None` + 函数内部判断。** 第 10 节的 pydantic 里有对应的官方解法（`default_factory`）。

### 2.11 ✅ Checkpoint 2：基础语法全跑通

把上面所有片段按顺序拼成 `a_basics.py`（我这份共 84 行），然后：

```bash
uv run --python 3.12 python a_basics.py
```

完整真实输出：

```
--- 1. 动态类型 ---
int 10
str 十
--- 2. int 没有上限 ---
1606938044258990275541962092341162602522202993782792835301376
Kotlin 的 Long 最大值: 9223372036854775807
--- 3. 整除 / 取模 / 幂 ---
3.5 3 1 49
-4 1
--- 4. f-string ---
小林 得了 91.5 分, 名字长度 2
score=91.456
--- 5. 真值判断 ---
  bool(     0) = False
  bool(     1) = True
  bool(    '') = False
  bool(   'a') = True
  bool(    []) = False
  bool(   [0]) = True
  bool(    {}) = False
  bool(  None) = False
  bool(   0.0) = False
--- 6. == vs is ---
a == b: True  a is b: False
c is a: True
--- 7. 切片 ---
234 012 789 02468 9876543210
[2, 3, 4] [0, 3, 6, 9]
--- 8. 推导式 ---
[0, 1, 4, 9, 16, 25]
[0, 2, 4, 6, 8]
{0: 0, 1: 1, 2: 4, 3: 9}
['e', 'h', 'l', 'o']
--- 9. 元组不可变 / 解包 ---
3 4
改元组报错: 'tuple' object does not support item assignment
--- 10. 可变默认参数陷阱 ---
['a', 'b', 'c'] ['a', 'b', 'c'] ['a', 'b', 'c']
['a'] ['b'] ['c']
--- 11. 字符串不可变 ---
ABC abc
拼接大量字符串要用 join: a-b-c
```

**自查**：第 10 节那两行必须一行污染、一行干净。如果你的两行一模一样，说明 `good` 里的 `if box is None` 写漏了。

---
## 三、函数与类：Kotlin 的每个特性都有对应物

新建 `b_func_class.py`，同样边加边跑。

### 3.1 参数的四种传法

Python 的函数签名比 Kotlin 灵活得多，一次讲清四种：

```python
def order(dish, size="中", *extras, spicy=False, **options):
    print(f"  dish={dish} size={size} extras={extras} spicy={spicy} options={options}")


order("拉面")
order("拉面", "大")
order("拉面", "大", "加蛋", "加肉")
order("拉面", spicy=True)
order("拉面", "小", "加蛋", spicy=True, note="不要葱", to_go=True)
```

输出：

```
  dish=拉面 size=中 extras=() spicy=False options={}
  dish=拉面 size=大 extras=() spicy=False options={}
  dish=拉面 size=大 extras=('加蛋', '加肉') spicy=False options={}
  dish=拉面 size=中 extras=() spicy=True options={}
  dish=拉面 size=小 extras=('加蛋',) spicy=True options={'note': '不要葱', 'to_go': True}
```

逐个解释：

| 写法 | 叫什么 | Kotlin 对应 | 说明 |
|---|---|---|---|
| `dish` | 位置参数 | 普通参数 | 必须传 |
| `size="中"` | 默认参数 | `size: String = "中"` | 一样 |
| `*extras` | 可变位置参数 | `vararg extras` | 收进一个**元组** |
| `spicy=False`（在 `*` 后面） | 仅关键字参数 | 无对应 | **只能写 `spicy=True`**，不能按位置传 |
| `**options` | 可变关键字参数 | 无对应 | 收进一个**字典** |

`*args` 和 `**kwargs` 这两个名字你会在所有 Python 库里见到（约定俗成，名字可以换但没人换）。它们最大的用途是**转发**——第 6 节的装饰器全靠这个。

**`*` 后面的参数强制用名字传**，这是个好设计：`create_user(name, admin=True)` 比 `create_user(name, True)` 可读得多。你可以主动加一个裸 `*` 来强制这一点：`def f(a, *, b)`。

### 3.2 多返回值就是元组

```python
def divide(a, b):
    return a // b, a % b


q, r = divide(17, 5)
print(q, r, type(divide(17, 5)).__name__)
```

输出：

```
3 2 tuple
```

不需要 `Pair`，不需要定义 data class。**但超过 3 个返回值时，请改用 `@dataclass` 或 `NamedTuple`**——不然调用方要数位置，很容易搞错顺序。

### 3.3 类型提示：写了不等于生效

```python
def greet(name: str, times: int = 1) -> str:
    return (f"你好, {name}! " * times).strip()


print(greet("小林", 2))
print("类型提示不会拦住你:", greet(123, 1))   # 运行期完全不检查
print("函数的注解:", greet.__annotations__)
```

输出：

```
你好, 小林! 你好, 小林!
类型提示不会拦住你: 你好, 123!
函数的注解: {'name': <class 'str'>, 'times': <class 'int'>, 'return': <class 'str'>}
```

**请把第二行盯上五秒**：我们声明了 `name: str`，却传了整数 `123`，Python **照跑不误**。

这是从 Kotlin 转过来最需要重建的认知：**Python 的类型提示（type hints）只是注解，运行时完全不检查。** 它的价值在于：给 IDE 做补全和跳转、给 mypy 做静态检查、给读代码的人当文档、给 pydantic/FastAPI 这类库当**元数据**（它们会在运行时读 `__annotations__` 来生成校验逻辑——这是第 10 节的关键机制）。

第 7 节会用 mypy 把这类错误抓出来。

### 3.4 类：`self` 必须自己写

```python
class Note:
    """一条笔记。"""

    count = 0                      # 类属性：所有实例共享(相当于 companion object 里的变量)

    def __init__(self, title: str, body: str = "", tags: list | None = None):
        self.title = title         # 实例属性，必须显式写 self
        self.body = body
        self.tags = tags or []
        Note.count += 1

    def add_tag(self, tag: str) -> "Note":
        self.tags.append(tag)
        return self                # 返回 self 就能链式调用

    def __repr__(self) -> str:     # 相当于 Kotlin data class 自带的 toString()
        return f"Note(title={self.title!r}, tags={self.tags})"

    def __len__(self) -> int:      # 让 len(note) 能用
        return len(self.body)

    def __eq__(self, other) -> bool:
        return isinstance(other, Note) and self.title == other.title

    @property
    def summary(self) -> str:      # 像属性一样访问的方法
        return self.body[:5] + ("..." if len(self.body) > 5 else "")

    @staticmethod
    def help() -> str:
        return "Note 用来存一条笔记"

    @classmethod
    def from_line(cls, line: str) -> "Note":
        title, _, body = line.partition(":")
        return cls(title.strip(), body.strip())
```

用起来：

```python
n = Note("Python 入门", "先学语法再学库")
n.add_tag("python").add_tag("笔记")
print(n)
print("len(n) =", len(n))
print("n.summary =", n.summary)
print("Note.count =", Note.count)
print("Note.help() =", Note.help())
n2 = Note.from_line("Kotlin: 和 Python 的对比")
print("from_line ->", n2)
print("Note.count =", Note.count)
print("标题相同就相等:", Note("A") == Note("A"))
```

输出：

```
Note(title='Python 入门', tags=['python', '笔记'])
len(n) = 7
n.summary = 先学语法再...
Note.count = 1
Note.help() = Note 用来存一条笔记
from_line -> Note(title='Kotlin', tags=[])
Note.count = 2
标题相同就相等: True
```

七个要点，逐条对照 Kotlin：

1. **`__init__` 是构造函数**，`self` 是 `this` 但**必须显式写成第一个参数**。写方法时忘了 `self` 是新手最高频的错误，报错长这样：`TypeError: add_tag() takes 1 positional argument but 2 were given`。
2. **实例属性必须写 `self.x = ...`**，没有"声明区"。不写 `self.` 就只是个局部变量，方法一结束就没了。
3. **类属性**（`count = 0`）写在 class 下、方法外，所有实例共享——就是 `companion object` 里的变量。注意 `Note.count += 1` 要用类名，写 `self.count += 1` 会**创建一个同名的实例属性**把类属性遮住（这个坑很隐蔽）。
4. **双下划线开头结尾的是"魔术方法"**（dunder method），是 Python 的运算符重载机制：`__repr__` 对应 `toString()`、`__len__` 让 `len(x)` 能用、`__eq__` 让 `==` 能用。注意 `!r` 在 f-string 里表示"用 `repr()` 而不是 `str()`"，所以字符串会带引号。
5. **`@property`** 让方法可以像属性一样访问（`n.summary` 不带括号）——就是 Kotlin 的自定义 getter。
6. **`@staticmethod`** 不需要 `self`，就是个挂在类里的普通函数。
7. **`@classmethod`** 第一个参数是 `cls`（类本身），主要用来写**替代构造函数**——`Note.from_line(...)` 就是 Kotlin 里 `companion object { fun fromLine(...) }` 的标准做法。

> **没有 `private`**。Python 靠约定：单下划线 `_x` 表示"内部用的，别碰"（工具和人都会尊重它，但语言不强制）；双下划线 `__x` 会触发名字改写（name mangling），有点像 private 但主要目的是避免子类冲突，**不是安全机制**。第 10 节的 `NoteStore._load()` 就是这个约定的实际用法。

### 3.5 继承

```python
class TodoNote(Note):
    def __init__(self, title: str, body: str = "", done: bool = False):
        super().__init__(title, body)
        self.done = done

    def __repr__(self) -> str:
        mark = "x" if self.done else " "
        return f"[{mark}] {self.title}"


t = TodoNote("买菜", done=True)
print(t, "| isinstance(t, Note) =", isinstance(t, Note))
```

输出：

```
[x] 买菜 | isinstance(t, Note) = True
```

`class 子类(父类):` 就是继承，`super().__init__(...)` 调父类构造。**和 Kotlin 最大的区别：Python 的类默认就能被继承、方法默认就能被覆盖**，不需要 `open`。另外 Python 支持多继承（`class C(A, B)`），但**初学阶段请避免**——方法解析顺序（MRO）的规则很绕，实际项目里用组合更好。

`x if cond else y` 是三元表达式，相当于 Kotlin 的 `if (cond) x else y`。

### 3.6 `@dataclass`：Kotlin `data class` 的直接对应

```python
from dataclasses import dataclass, field


@dataclass
class Config:
    model: str = "deepseek-chat"
    temperature: float = 0.7
    tags: list = field(default_factory=list)   # 可变默认值必须用 default_factory


c1 = Config()
c2 = Config()
c1.tags.append("a")
print(c1)
print("两个实例的 tags 是不是同一个:", c1.tags is c2.tags)
print("dataclass 自带 ==:", Config(model="x") == Config(model="x"))
```

输出：

```
Config(model='deepseek-chat', temperature=0.7, tags=['a'])
两个实例的 tags 是不是同一个: False
dataclass 自带 ==: True
```

`@dataclass` 会**自动生成** `__init__`、`__repr__`、`__eq__`——就是 `data class` 干的事。

**注意第三个字段**：可变默认值不能写 `tags: list = []`（那就是 2.10 的坑），必须写 `field(default_factory=list)`，意思是"每次创建实例时**调用一次** `list()` 生成新的"。输出第二行 `False` 就是这个机制生效的证据。

`@dataclass` vs pydantic `BaseModel` 怎么选：**纯内部数据结构用 `@dataclass`（标准库自带、零开销）；数据来自外部（HTTP 请求、配置文件、LLM 返回）用 pydantic（会真的校验和转换）**。第 10 节会展示 pydantic 那一半。

### 3.7 鸭子类型：不实现接口也能用

```python
class FakeFile:
    def write(self, s):
        print("  FakeFile 收到:", s.strip())


def dump(f):
    f.write("hello\n")


dump(FakeFile())      # 不需要实现任何接口，有 write 方法就行
```

输出：

```
  FakeFile 收到: hello
```

"**如果它走起来像鸭子、叫起来像鸭子，那它就是鸭子**"——Python 不检查你实现了什么接口，只在调用的那一刻看有没有这个方法。

**这对写测试是巨大的便利**：想 mock 一个文件对象，不用继承任何东西，随手写个有 `write` 方法的类就行，也不需要 Mockito。代价是"接口"只存在于文档和你的脑子里，写错了要运行时才发现——第 7 节的 `Protocol` 就是把这个契约重新写下来给 mypy 检查的办法。

### 3.8 ✅ Checkpoint 3：函数与类

`b_func_class.py` 完整跑一遍（我这份 139 行）：

```bash
uv run --python 3.12 python b_func_class.py
```

真实输出：

```
--- 1. 参数四种传法 ---
  dish=拉面 size=中 extras=() spicy=False options={}
  dish=拉面 size=大 extras=() spicy=False options={}
  dish=拉面 size=大 extras=('加蛋', '加肉') spicy=False options={}
  dish=拉面 size=中 extras=() spicy=True options={}
  dish=拉面 size=小 extras=('加蛋',) spicy=True options={'note': '不要葱', 'to_go': True}
--- 2. 多返回值 ---
3 2 tuple
--- 3. 类型提示 ---
你好, 小林! 你好, 小林!
类型提示不会拦住你: 你好, 123!
函数的注解: {'name': <class 'str'>, 'times': <class 'int'>, 'return': <class 'str'>}
--- 4. 类 ---
Note(title='Python 入门', tags=['python', '笔记'])
len(n) = 7
n.summary = 先学语法再...
Note.count = 1
Note.help() = Note 用来存一条笔记
from_line -> Note(title='Kotlin', tags=[])
Note.count = 2
标题相同就相等: True
--- 5. 继承 ---
[x] 买菜 | isinstance(t, Note) = True
--- 6. dataclass ---
Config(model='deepseek-chat', temperature=0.7, tags=['a'])
两个实例的 tags 是不是同一个: False
dataclass 自带 ==: True
--- 7. 鸭子类型 ---
  FakeFile 收到: hello
```

**自查两处**：`类型提示不会拦住你: 你好, 123!` 必须出现（证明类型不被强制）；`两个实例的 tags 是不是同一个: False`（证明 `default_factory` 生效）。

---
## 四、异常、文件、JSON、日志：写出不会静默出错的代码

新建 `c_io_json.py`。

### 4.1 异常：多了一个 `else`

```python
def parse_age(text):
    try:
        age = int(text)
    except ValueError as e:
        print(f"  转不了数字({e!r}) -> 返回 None")
        return None
    except (TypeError, KeyError):
        print("  可以一次抓多种")
        return None
    else:
        print("  没出错才会走 else")
        return age
    finally:
        print("  finally 一定执行(不管 return 还是抛错)")


print("结果:", parse_age("28"))
print("结果:", parse_age("二十八"))
```

输出：

```
  没出错才会走 else
  finally 一定执行(不管 return 还是抛错)
结果: 28
  转不了数字(ValueError("invalid literal for int() with base 10: '二十八'")) -> 返回 None
  finally 一定执行(不管 return 还是抛错)
结果: None
```

四段的职责：

| 段 | 什么时候执行 | Kotlin 对应 |
|---|---|---|
| `try` | 可能出错的代码 | `try` |
| `except X as e` | 出了这类错 | `catch (e: X)` |
| `else` | **一个错都没出**时 | 无对应 |
| `finally` | 无论如何都执行 | `finally` |

`else` 是 Python 特有的，价值在于**把"不该被保护的代码"挪出 try**。如果把 `return age` 写在 try 里，而 `age` 的后续处理又抛了 `ValueError`，就会被上面的 `except` 误抓。**try 块越小越好，这是异常处理的第一原则。**

`except (TypeError, KeyError)` 用元组一次抓多种。

### 4.2 自定义异常与异常链

```python
class ToolError(Exception):
    """业务自己的异常类型，只要继承 Exception。"""


def call_tool(name):
    try:
        return {"get_weather": "晴"}[name]
    except KeyError as e:
        raise ToolError(f"没有名为「{name}」的工具") from e


try:
    call_tool("get_wether")     # 故意拼错
except ToolError as e:
    print("  捕获:", e)
    print("  原始原因:", repr(e.__cause__))
```

输出：

```
  捕获: 没有名为「get_wether」的工具
  原始原因: KeyError('get_wether')
```

定义异常只要 `class XxxError(Exception):` 加一行文档字符串就够了（**类体不能为空，用文档字符串或 `pass` 占位**）。

**`raise ... from e` 是重点**：它把底层异常挂在新异常的 `__cause__` 上。这样上层拿到的是业务语义的错误（"没有这个工具"），但排查时仍能顺着链条找到根因（`KeyError`）。打印 traceback 时会显示 "The above exception was the direct cause of..."。**前面章节里包装 LLM 调用、工具调用的异常，都该这么写。**

### 4.3 千万别写裸 `except`

```python
try:
    try:
        raise KeyboardInterrupt("用户按了 Ctrl+C")
    except Exception as e:          # 正确：Exception 抓不到 KeyboardInterrupt
        print("  不该走到这里")
except KeyboardInterrupt:
    print("  KeyboardInterrupt 逃出了 except Exception —— 这是对的")
```

输出：

```
  KeyboardInterrupt 逃出了 except Exception —— 这是对的
```

Python 的异常有两条继承线：`Exception`（业务错误）和 `BaseException`（包含 `KeyboardInterrupt`、`SystemExit` 这些"控制流"信号）。

- `except Exception:` —— **抓业务错误，不会拦住 Ctrl+C**。这是你 99% 的场合该写的。
- `except:` 或 `except BaseException:` —— **连 Ctrl+C 都吞掉**，程序变成杀不死的僵尸。**永远不要写。**

上面的实验就是证据：`except Exception` 没有拦住 `KeyboardInterrupt`，它成功逃到了外层。

**另一条铁律：`except` 里绝不能什么都不做。** `except Exception: pass` 是"静默吞错"，是最难排查的 bug 来源。至少 `log.exception(...)` 一下。

### 4.4 `pathlib`：别再手拼路径

```python
from pathlib import Path

work = Path("workspace")
work.mkdir(exist_ok=True)
f = work / "notes.txt"          # 用 / 拼路径，跨平台
f.write_text("第一行\n第二行\n", encoding="utf-8")
print("  存在吗:", f.exists(), "| 大小:", f.stat().st_size, "字节")
print("  文件名:", f.name, "| 后缀:", f.suffix, "| 父目录:", f.parent)
print("  读回来:", repr(f.read_text(encoding="utf-8")))
print("  目录里有:", sorted(p.name for p in work.iterdir()))
```

输出：

```
  存在吗: True | 大小: 20 字节
  文件名: notes.txt | 后缀: .txt | 父目录: workspace
  读回来: '第一行\n第二行\n'
  目录里有: ['data.json', 'notes.txt']
```

`Path` 用 **`/` 运算符拼路径**（重载了除号），跨平台自动处理分隔符——相当于 Kotlin 里 `File(parent, child)` 但顺手得多。

三个要点：**`mkdir(exist_ok=True)` 目录已存在时不报错**（还有 `parents=True` 递归创建）；**`write_text`/`read_text` 一行完成读写**，省掉 `open` + `with`；**永远显式写 `encoding="utf-8"`**——不写就用系统默认编码，在中文 Windows 上是 GBK，读中文文件必乱码。这是跨平台协作最常见的一个坑。

### 4.5 `with`：自动关资源

```python
with open(f, "a", encoding="utf-8") as fp:
    fp.write("第三行\n")
with open(f, encoding="utf-8") as fp:
    for i, line in enumerate(fp, start=1):
        print(f"  {i}: {line.rstrip()}")
```

输出：

```
  1: 第一行
  2: 第二行
  3: 第三行
```

`with` 是 Kotlin 的 `use { }`：**块结束时自动调用清理逻辑**，就算中途抛异常也会执行。文件、数据库连接、HTTP 客户端、锁，全都该用 `with`。

打开模式：`"r"` 读（默认）、`"w"` 写（**会清空原文件**）、`"a"` 追加、`"rb"`/`"wb"` 二进制。

**直接对文件对象做 for 循环是逐行迭代，而且是惰性的**——读 10GB 日志也不会爆内存，这比 `read().split("\n")` 好得多。

自己写一个 `with` 能用的类，只要两个魔术方法：

```python
class Timer:
    def __enter__(self):
        print("  [Timer] 开始")
        return self

    def __exit__(self, exc_type, exc_val, tb):
        print(f"  [Timer] 结束 (异常类型={exc_type})")
        return False        # 返回 False = 不吞掉异常


with Timer():
    print("  干活中...")
```

输出：

```
  [Timer] 开始
  干活中...
  [Timer] 结束 (异常类型=None)
```

`__exit__` 的三个参数是异常信息（没异常时全是 `None`）；**返回 `False` 表示"我不处理，让异常继续往上抛"**——除非你真的想吞掉异常，否则永远返回 `False` 或者干脆不返回（默认就是 `None`，等效于 `False`）。

### 4.6 JSON：和 Python 类型的对应关系

```python
import json

data = {"name": "小林", "tags": ["python", "agent"], "score": 91.5, "ok": True,
        "extra": None}
text = json.dumps(data, ensure_ascii=False, indent=2)
print(text)
back = json.loads(text)
print("  往返后相等:", back == data)
print("  Python None <-> JSON null:", json.dumps({"a": None}))
print("  Python True <-> JSON true:", json.dumps({"a": True}))
```

输出：

```
{
  "name": "小林",
  "tags": [
    "python",
    "agent"
  ],
  "score": 91.5,
  "ok": true,
  "extra": null
}
  往返后相等: True
  Python None <-> JSON null: {"a": null}
  Python True <-> JSON true: {"a": true}
```

`dumps` = 对象 → 字符串（**d**ump **s**tring），`loads` = 字符串 → 对象。带文件版本是 `dump(obj, fp)` / `load(fp)`（没有 s）。

**`ensure_ascii=False` 必须写**，否则中文会被转义成 `\uXXXX`。实测对比：

```python
print(json.dumps({"name": "小林"}))
print(json.dumps({"name": "小林"}, ensure_ascii=False))
```

```
{"name": "\u5c0f\u6797"}
{"name": "小林"}
```

两者都是合法 JSON、解析回来一模一样，但第一种人没法读，日志和数据文件里全是这种东西会让排查变成噩梦。`indent=2` 是美化缩进，存文件时加上，网络传输时省掉。

对应关系要记住：`None ↔ null`、`True ↔ true`（**Python 大写、JSON 小写**）、`dict ↔ object`、`list ↔ array`。**元组会被转成 array，但转回来变成 list**，不是无损往返。

JSON 比你想的严格：

```python
try:
    json.loads("{'name': '小林'}")      # 单引号不是合法 JSON
except json.JSONDecodeError as e:
    print("  单引号解析失败:", e)
```

输出：

```
  单引号解析失败: Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
```

**JSON 只认双引号，且不允许尾随逗号。** 这在处理 LLM 输出时是高频事故——模型很爱吐单引号或者带尾逗号的"JSON"。前面章节里用 `ast.literal_eval` 兜底解析、以及严格要求模型输出格式，就是为了对付这个。

### 4.7 `logging`：别用 `print` 调试

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)-7s [%(name)s] %(message)s",
    datefmt="%H:%M:%S",
)
log = logging.getLogger("notekit")
log.debug("这条看不见，因为级别是 INFO")
log.info("加载了 %d 条笔记", 3)
log.warning("磁盘快满了")
try:
    1 / 0
except ZeroDivisionError:
    log.exception("算错了")        # 自动带上堆栈
```

输出（这几行是写到 **stderr** 的）：

```
17:49:28 INFO    [notekit] 加载了 3 条笔记
17:49:28 WARNING [notekit] 磁盘快满了
17:49:28 ERROR   [notekit] 算错了
Traceback (most recent call last):
  File "/private/tmp/verify/py/c_io_json.py", line 124, in <module>
    1 / 0
    ~~^~~
ZeroDivisionError: division by zero
```

五个级别从低到高：`DEBUG` → `INFO` → `WARNING` → `ERROR` → `CRITICAL`。`level=logging.INFO` 意思是"INFO 及以上才输出"，所以 `log.debug(...)` 那行看不见——**这就是 logging 相对 print 的核心优势：一个开关控制所有输出的详细程度**，上线改成 WARNING 就安静了，出问题改成 DEBUG 就啥都看得见。

三个细节：

- **用 `log.info("加载了 %d 条", 3)` 而不是 f-string**。这种"延迟格式化"在日志级别不够时**根本不会做字符串拼接**，省性能；而且日志聚合系统能按模板归类。
- **`log.exception(...)` 只能在 `except` 块里用**，它会自动附上完整堆栈——比 `log.error(str(e))` 有用一百倍，后者丢掉了出错位置。
- **`getLogger("名字")` 按模块取 logger**，标准做法是每个文件顶部写 `log = logging.getLogger(__name__)`，这样日志里自动带上模块名，能按模块单独调级别。

> **一个观察输出时的注意点**：上面这些日志行在终端里是和 `print` 混在一起的，但**如果你把输出重定向到文件（`> out.txt`），日志会消失**——因为它默认走 stderr，而 print 走 stdout。要一起收集得写 `> out.txt 2>&1`。同样，你看到的日志行和 print 行的**先后顺序在重定向后可能对不上**，这是两个流各自缓冲导致的，不是你的代码有问题。

### 4.8 ✅ Checkpoint 4：异常 / 文件 / JSON / 日志

```bash
uv run --python 3.12 python c_io_json.py
```

除上面已贴的各段外，你还会看到最后两行：

```
--- 8. __name__ ---
  当前 __name__ = __main__
  这就是 if __name__ == '__main__' 的原理：被 import 时它等于模块名
```

这是下一节的引子。**自查**：`workspace/` 目录被创建了，里面有 `notes.txt`（3 行）和 `data.json`。

---
## 五、模块与包：新手最大的一道坎

**如果你只读这一章的一节，读这节。** `ImportError` / `ModuleNotFoundError` 是 Python 新手 90% 的挫败感来源，而它的规则其实只有三条。

### 5.1 是什么：`import` 到底干了什么

Android 里 `import` 是给编译器看的，纯粹是"写全类名太长"的简写。**Python 的 `import` 是一条真正执行的语句**，它做三件事：

1. **找**：在 `sys.path` 列出的目录里按名字找 `.py` 文件或包目录
2. **执行**：把那个文件**从头到尾跑一遍**（所以文件里的 `print` 会打印出来）
3. **缓存**：把结果记进 `sys.modules`，**同一个模块第二次 import 不会再执行**

建一个目录来验证。文件结构：

```
impdemo/
├── main.py
└── pkg/
    ├── __init__.py
    ├── tools.py
    └── sub/
        └── deep.py
```

`pkg/__init__.py`：

```python
print("  [import] pkg/__init__.py 被执行了")
VERSION = "1.0"
```

`pkg/tools.py`：

```python
print("  [import] pkg/tools.py 被执行了")


def greet(name):
    return f"你好 {name}"


if __name__ == "__main__":
    print("直接运行 tools.py 时才会打印这句")
```

`pkg/sub/deep.py`：

```python
from ..tools import greet          # 相对导入：.. 表示上一层包


def shout(name):
    return greet(name).upper()
```

`main.py`：

```python
import sys

print("1) 模块搜索路径的第一项(脚本所在目录):", sys.path[0])

print("2) 第一次 import:")
import pkg.tools

print("3) 第二次 import（不会再执行一遍）:")
import pkg.tools

print("4) 调用:", pkg.tools.greet("小林"))

from pkg.sub.deep import shout
print("5) 相对导入也能用:", shout("小林"))

print("6) pkg 的属性:", pkg.VERSION)
print("7) 已加载的 pkg 相关模块:", sorted(m for m in sys.modules if m.startswith("pkg")))
```

跑：

```bash
uv run --python 3.12 python impdemo/main.py
```

真实输出：

```
1) 模块搜索路径的第一项(脚本所在目录): /private/tmp/verify/py/impdemo
2) 第一次 import:
  [import] pkg/__init__.py 被执行了
  [import] pkg/tools.py 被执行了
3) 第二次 import（不会再执行一遍）:
4) 调用: 你好 小林
5) 相对导入也能用: 你好 小林
6) pkg 的属性: 1.0
7) 已加载的 pkg 相关模块: ['pkg', 'pkg.sub', 'pkg.sub.deep', 'pkg.tools']
```

**这个输出里有五个关键信息**：

1. **`sys.path[0]` 是"被运行的那个脚本所在的目录"**——这是理解所有 import 问题的钥匙。不是你敲命令时所在的目录，是**脚本文件所在的目录**。
2. **import `pkg.tools` 会先执行 `pkg/__init__.py`**（两行 print 的顺序就是证据）。所以 `__init__.py` 里别写耗时的东西。
3. **第 3 步什么都没打印**——第二次 import 直接从 `sys.modules` 拿缓存。这意味着**模块顶层代码在一次进程里只执行一次**，你可以放心地在多个文件里 import 同一个模块。
4. **`pkg.VERSION` 能访问**：`__init__.py` 里定义的东西就是"包本身"的属性。
5. 第 7 行显示 `pkg.sub` 也被加载了——**导入子模块会连带导入所有父包**。

### 5.2 怎么用：三条规则

**规则一：`__init__.py` 标记"这是一个包"。**

它可以是空文件（现代 Python 里包目录没有它也能被识别，叫"命名空间包"，但**别依赖这个特性**）。它的正经用途是**对外暴露简洁的 API**：

```python
# pkg/__init__.py
from .tools import greet      # 这样外面可以写 from pkg import greet
```

第 10 节的 `notekit/__init__.py` 就是这么用的。

**规则二：绝对导入用在跨包，相对导入用在包内部。**

```python
from pkg.tools import greet    # 绝对导入：从根开始写全路径
from .tools import greet       # 相对导入：. 是当前包，.. 是上一层
```

**包内部的互相引用一律用相对导入**（`.` 开头）。理由：包被改名或移动时不用改代码。第 10 节的 `notekit` 里所有内部引用都是 `from .models import Note` 这种形式。

**规则三：`if __name__ == "__main__":` 是"这个文件是被直接运行的吗"。**

每个模块都有一个 `__name__` 变量：被直接运行时它是字符串 `"__main__"`，被 import 时它是模块名（比如 `"pkg.tools"`）。所以：

```python
if __name__ == "__main__":
    main()          # 只在直接运行时执行，被 import 时不执行
```

**这不是仪式，是必需的**：不写的话，别人 import 你的模块会意外触发你的主流程。第 8 节还会看到，多进程场景下不写这句会导致**代码被重复执行**。

### 5.3 踩坑：三种 ImportError 的成因

**坑一：直接运行包里的文件。**

```bash
uv run --python 3.12 python impdemo/pkg/tools.py
```

输出：

```
  [import] pkg/tools.py 被执行了
直接运行 tools.py 时才会打印这句
```

这次侥幸成功了（因为 `tools.py` 没有相对导入）。但换成有相对导入的 `deep.py`：

```bash
uv run --python 3.12 python impdemo/pkg/sub/deep.py
```

真实报错：

```
Traceback (most recent call last):
  File "/private/tmp/verify/py/impdemo/pkg/sub/deep.py", line 1, in <module>
    from ..tools import greet          # 相对导入：.. 表示上一层包
    ^^^^^^^^^^^^^^^^^^^^^^^^^
ImportError: attempted relative import with no known parent package
```

**原因**：你直接运行一个文件时，Python 把它当成"顶层脚本"，它的 `__package__` 是空的，**"上一层包"这个概念不存在**，`..` 无处可解析。

**解法：用 `-m` 以模块方式运行。**

```bash
cd impdemo
uv run --python 3.12 python -m pkg.tools
```

输出：

```
  [import] pkg/__init__.py 被执行了
  [import] pkg/tools.py 被执行了
直接运行 tools.py 时才会打印这句
```

**注意和前面的差别**：这次 `pkg/__init__.py` 也被执行了——因为 `-m pkg.tools` 是"把它当成包里的模块来运行"，Python 知道它的父包是 `pkg`，相对导入就能工作了。

> **记住这条经验法则**：`python 路径/文件.py` 用于运行**入口脚本**；`python -m 包.模块` 用于运行**包里的模块**。二选一，别混。

**坑二：`ModuleNotFoundError: No module named 'xxx'`，但我明明装了。**

99% 是**装到了另一个环境**。你用系统 pip 装的，却用 `.venv` 里的 python 跑（或者反过来）。诊断命令：

```bash
uv run python -c "import sys; print(sys.executable); print(sys.path)"
```

看第一行是不是你以为的那个 python。**用 `uv run` 前缀就基本不会遇到这个问题。**

**坑三：循环导入（circular import）。**

`a.py` 里 `import b`，`b.py` 里 `import a`，报错通常是 `ImportError: cannot import name 'X' from partially initialized module`。原因是 a 还没执行完就去执行 b，b 又要 a 里还没定义的东西。

三个解法，按推荐顺序：

1. **重新设计**：把公共部分抽到第三个模块 `common.py`。循环导入几乎总是"分层没分好"的症状——第 10 节的 `notekit` 是严格单向的：`models` ← `store` ← `cli`/`web`，`models` 不认识任何人。
2. **把 import 挪进函数里**（延迟到调用时才导入）。第 10 节的 `cli.py` 里 `import uvicorn` 就写在 `serve()` 函数内部——顺带好处是不跑 `serve` 命令时不用付它的加载时间。
3. 只在类型标注里需要的话，用 `if TYPE_CHECKING:` 包起来（运行时不导入）。

### 5.4 ✅ Checkpoint 5：import 机制

按上面结构建好 `impdemo/`，依次跑三条命令：

```bash
uv run --python 3.12 python impdemo/main.py                  # 应该 7 行全打印
uv run --python 3.12 python impdemo/pkg/sub/deep.py          # 应该报 ImportError
cd impdemo && uv run --python 3.12 python -m pkg.tools       # 应该成功，且多打印 __init__ 那行
```

**第二条必须失败**——这是本节最重要的实验。如果它成功了，说明你的 `deep.py` 里没写相对导入。

---
## 六、装饰器、生成器、闭包：读懂满屏的 `@`

FastAPI、pytest、Typer、LangChain 的代码里到处是 `@`。这一节把它拆开，你会发现没什么魔法。新建 `d_advanced.py`。

### 6.1 闭包：函数记住了它出生的环境

```python
def make_counter():
    n = 0

    def inc():
        nonlocal n          # 不写 nonlocal 就会报 UnboundLocalError
        n += 1
        return n

    return inc


c = make_counter()
print(c(), c(), c())
```

输出：

```
1 2 3
```

`inc` 是定义在 `make_counter` **内部**的函数，它"记住"了外层的变量 `n`——即使 `make_counter` 已经返回了。这就是**闭包**，和 Kotlin 的 lambda 捕获变量是一回事。

**`nonlocal` 是必须的**。Python 的规则是"**函数里只要对一个名字赋值，这个名字就默认是局部变量**"。所以不写 `nonlocal n`，`n += 1` 会被理解成"给局部变量 n 加一"，而局部的 n 还没有值 → `UnboundLocalError`。这个报错第一次遇到会很懵，记住原因就行。（对应的还有 `global`，用来改模块级变量——但 `global` 基本上都是设计缺陷的信号，别用。）

### 6.2 装饰器：`@` 只是语法糖

```python
import functools
import time


def timed(fn):
    @functools.wraps(fn)            # 不加这行，fn.__name__ 会变成 'wrapper'
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        result = fn(*args, **kwargs)
        ms = (time.perf_counter() - t0) * 1000
        print(f"  [timed] {fn.__name__} 用了 {ms:.1f} ms")
        return result

    return wrapper


@timed
def slow_add(a, b):
    time.sleep(0.05)
    return a + b


print("  结果:", slow_add(1, 2))
print("  函数名还在:", slow_add.__name__)
```

输出：

```
  [timed] slow_add 用了 53.5 ms
  结果: 3
  函数名还在: slow_add
  等价写法: slow_add = timed(slow_add)
```

**装饰器就三句话**：

1. 它是一个**接受函数、返回函数**的函数。
2. `@timed` 写在 `def slow_add` 上面，完全等价于 `slow_add = timed(slow_add)`——**没有任何魔法，就是个赋值**。
3. 返回的 `wrapper` 用 `*args, **kwargs` 接住所有参数再转发，这样它能装饰任何签名的函数（3.1 节的伏笔在这里回收）。

**`@functools.wraps(fn)` 别省**。不加它，`slow_add.__name__` 会变成 `'wrapper'`，文档字符串也丢了——调试时看到一堆叫 wrapper 的函数会很痛苦，而且有些框架（比如 FastAPI）依赖函数元数据工作。

对照 Kotlin：装饰器最接近的类比是**注解 + 编译期代码生成**（比如 Room 的 `@Dao`），但 Python 的是**运行时**发生的，而且你自己十行就能写一个。

### 6.3 带参数的装饰器：三层套娃

```python
def retry(times=3):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for i in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except Exception as e:
                    print(f"  第{i}次失败: {e}")
                    if i == times:
                        raise
            return None
        return wrapper
    return deco


calls = {"n": 0}


@retry(times=3)
def flaky():
    calls["n"] += 1
    if calls["n"] < 3:
        raise RuntimeError("网络抖动")
    return "成功了"


print("  结果:", flaky())
```

输出：

```
  第1次失败: 网络抖动
  第2次失败: 网络抖动
  结果: 成功了
```

为什么要三层？因为 `@retry(times=3)` 里 **`retry(times=3)` 先被调用**，它得返回一个"真正的装饰器"，那个装饰器再接收函数。所以层次是：`retry(参数)` → `deco(函数)` → `wrapper(调用时的参数)`。

**这个 `retry` 是生产代码里的常客**——调 LLM API 会超时、会限流，不重试的调用等于没有容错。注意 `if i == times: raise`：**最后一次失败要把异常抛出去，不能悄悄返回 None**，否则调用方拿到 None 会在别的地方炸，排查成本翻倍。

### 6.4 标准库自带的装饰器：`lru_cache`

```python
@functools.lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)


t0 = time.perf_counter()
print("  fib(90) =", fib(90))
print(f"  用了 {(time.perf_counter()-t0)*1000:.2f} ms（没有缓存的话要跑到宇宙尽头）")
print("  缓存命中情况:", fib.cache_info())
```

输出：

```
  fib(90) = 2880067194370816120
  用了 0.02 ms（没有缓存的话要跑到宇宙尽头）
  缓存命中情况: CacheInfo(hits=88, misses=91, maxsize=None, currsize=91)
```

**一行装饰器把指数级算法变成线性的。** 没有缓存的递归 `fib(90)` 要算约 2⁹⁰ 次；有了缓存，`misses=91`（每个 n 只真算一次）、`hits=88`（其余全部命中缓存），总共 0.02 毫秒。

顺便注意：`fib(90) = 2880067194370816120` 这个数**已经超过了 Kotlin `Int` 的范围**（在 `Long` 范围内），Python 完全不用操心（2.3 节）。

**用它的三个前提**：函数必须是**纯函数**（同样输入必然同样输出，没有副作用）；参数必须**可哈希**（不能是 list 或 dict）；`maxsize` 设成 `None` 意味着**永不淘汰**，有内存泄漏风险——线上代码请给个具体数字，比如 `maxsize=1024`。

### 6.5 生成器：`yield` 是暂停键

```python
def read_lines(n):
    print("  [生成器] 被调用时函数体一行都没执行")
    for i in range(1, n + 1):
        print(f"  [生成器] 生产第 {i} 行")
        yield f"line-{i}"


gen = read_lines(3)
print("  拿到的是:", type(gen).__name__)
print("  取第一个:", next(gen))
print("  取第二个:", next(gen))
print("  剩下的一次拿完:", list(gen))
```

输出：

```
  拿到的是: generator
  [生成器] 被调用时函数体一行都没执行
  [生成器] 生产第 1 行
  取第一个: line-1
  [生成器] 生产第 2 行
  取第二个: line-2
  [生成器] 生产第 3 行
  剩下的一次拿完: ['line-3']
```

**请仔细看输出顺序，它揭示了生成器的全部行为**：

- 调用 `read_lines(3)` 时，**函数体一行都没执行**（第一个 print 出现在"拿到的是"之后！），只是返回了一个 generator 对象。
- 每次 `next(gen)`，函数从上次 `yield` 的位置**继续**执行，遇到下一个 `yield` 就**暂停并交出值**。
- 局部变量（这里的 `i`）在暂停期间**保持不变**。

一句话：**只要函数里有 `yield`，它就不再是普通函数，而是一个"可以暂停和恢复的过程"。** 这和 Kotlin 的 `sequence { yield(...) }` 是同一个概念。

**它的价值是内存：**

```python
import sys
print("  列表 [x*x for x in range(100000)] 占:",
      sys.getsizeof([x * x for x in range(100000)]), "字节")
print("  生成器 (x*x for x in range(100000)) 占:",
      sys.getsizeof(x * x for x in range(100000)), "字节")
```

输出：

```
  列表 [x*x for x in range(100000)] 占: 800984 字节
  生成器 (x*x for x in range(100000)) 占: 200 字节
```

**80 万字节 vs 200 字节，差 4000 倍。** 列表把 10 万个结果全算出来存下；生成器只存"怎么算下一个"。所以处理大文件、大数据集、或者**流式接收 LLM 的输出**时，一律用生成器。

注意语法差别只有括号：`[...]` 是列表推导式，`(...)` 是生成器表达式。

**代价要知道**：生成器**只能遍历一次**（遍历完就空了，上面 `list(gen)` 之后 gen 就废了），也**不能取长度、不能按下标访问**。需要反复用就老实生成列表。

### 6.6 `async`/`await`：等待时去干别的

```python
import asyncio


def sync_fetch(name, delay):
    time.sleep(delay)
    return f"{name} 完成"


async def async_fetch(name, delay):
    await asyncio.sleep(delay)       # 注意是 asyncio.sleep 不是 time.sleep
    return f"{name} 完成"


async def main():
    t0 = time.perf_counter()
    results = await asyncio.gather(
        async_fetch("模型A", 0.3),
        async_fetch("模型B", 0.3),
        async_fetch("模型C", 0.3),
    )
    print("  异步并发:", results)
    print(f"  异步耗时 {(time.perf_counter()-t0):.2f} 秒")


t0 = time.perf_counter()
print("  同步串行:", [sync_fetch(n, 0.3) for n in ("模型A", "模型B", "模型C")])
print(f"  同步耗时 {(time.perf_counter()-t0):.2f} 秒")
asyncio.run(main())
```

输出：

```
  同步串行: ['模型A 完成', '模型B 完成', '模型C 完成']
  同步耗时 0.91 秒
  异步并发: ['模型A 完成', '模型B 完成', '模型C 完成']
  异步耗时 0.30 秒
```

**0.91 秒 → 0.30 秒**。三个各等 0.3 秒的任务，同步是加法（0.3×3），异步是取最大值（3 个等待重叠在一起）。

四个概念：

| 写法 | 含义 |
|---|---|
| `async def f():` | 声明协程函数，**调用它不会执行**，只返回一个协程对象 |
| `await x` | "我要等 x，等的期间事件循环去跑别的" |
| `asyncio.gather(a, b, c)` | 同时启动多个，全部完成后一起返回 |
| `asyncio.run(main())` | 从普通同步代码进入异步世界的**唯一入口** |

对照 Android：`async def` ≈ `suspend fun`，`await` ≈ 直接调用 suspend 函数，`asyncio.gather` ≈ `coroutineScope { async{}; async{} }`，`asyncio.run` ≈ `runBlocking`。**概念是一对一的，你已经会了。**

### 6.7 最常见的 async 错误：混进阻塞调用

```python
async def wrong_one(delay):
    time.sleep(delay)        # 错！应该是 await asyncio.sleep(delay)


async def wrong():
    t0 = time.perf_counter()
    await asyncio.gather(*[wrong_one(0.2) for _ in range(3)])
    print(f"  用错 time.sleep 的耗时 {(time.perf_counter()-t0):.2f} 秒 —— 完全没并发")


asyncio.run(wrong())
```

输出：

```
  用错 time.sleep 的耗时 0.61 秒 —— 完全没并发
```

**0.61 ≈ 0.2 × 3**，一点都没快。因为 `time.sleep` 是**阻塞**的：它把整个线程（也就是整个事件循环）卡住，其它协程根本没机会跑。

**这是 async 代码里最高频的 bug，而且它不报错，只是悄悄地不生效。** 记住这条对照表：

| 阻塞版（不能在 async 里用） | 异步版 |
|---|---|
| `time.sleep(1)` | `await asyncio.sleep(1)` |
| `requests.get(...)` | `await httpx.AsyncClient().get(...)` |
| `open(...).read()` | `await aiofiles.open(...)` 或丢给线程池 |
| 任何 CPU 密集的循环 | `await asyncio.to_thread(...)` / 进程池 |

**判断方法**：`async def` 函数里出现的每一个"会等待"的调用，前面都必须有 `await`。看到裸的 `time.sleep` 或 `requests.` 就是错的。实在没有异步版本（比如某个只有同步 SDK 的库），用 `await asyncio.to_thread(阻塞函数, 参数)` 把它扔到线程里去。

### 6.8 ✅ Checkpoint 6：装饰器 / 生成器 / 异步

```bash
uv run --python 3.12 python d_advanced.py
```

完整真实输出：

```
--- 1. 闭包 ---
1 2 3
--- 2. 装饰器 ---
  [timed] slow_add 用了 53.5 ms
  结果: 3
  函数名还在: slow_add
  等价写法: slow_add = timed(slow_add)
--- 3. 带参数的装饰器 ---
  第1次失败: 网络抖动
  第2次失败: 网络抖动
  结果: 成功了
--- 4. lru_cache ---
  fib(90) = 2880067194370816120
  用了 0.02 ms（没有缓存的话要跑到宇宙尽头）
  缓存命中情况: CacheInfo(hits=88, misses=91, maxsize=None, currsize=91)
--- 5. 生成器 ---
  拿到的是: generator
  [生成器] 被调用时函数体一行都没执行
  [生成器] 生产第 1 行
  取第一个: line-1
  [生成器] 生产第 2 行
  取第二个: line-2
  [生成器] 生产第 3 行
  剩下的一次拿完: ['line-3']
--- 6. 生成器省内存 ---
  列表 [x*x for x in range(100000)] 占: 800984 字节
  生成器 (x*x for x in range(100000)) 占: 200 字节
--- 7. 同步 vs 异步 ---
  同步串行: ['模型A 完成', '模型B 完成', '模型C 完成']
  同步耗时 0.91 秒
  异步并发: ['模型A 完成', '模型B 完成', '模型C 完成']
  异步耗时 0.30 秒
  用错 time.sleep 的耗时 0.61 秒 —— 完全没并发
  记住: async 函数里所有阻塞调用都必须换成 await 版本，否则白写。
```

**自查三处**：`[生成器] 被调用时...` 出现在 `拿到的是: generator` **之后**（惰性求值）；`0.91` 秒 vs `0.30` 秒（异步生效）；`0.61` 秒（用错 sleep 就白写）。耗时数字会有几十毫秒波动，量级对上就行。

---
## 七、类型提示与 mypy：把编译器请回来

3.3 节我们证明了"类型提示运行时不生效"。这一节把它变得有用。新建 `j_typing.py`。

### 7.1 基础写法速查

```python
name: str = "小林"
count: int = 0
ratio: float = 0.5
ok: bool = True
tags: list[str] = []                    # 3.9+ 可以直接用内置类型
scores: dict[str, int] = {}
pair: tuple[int, str] = (1, "a")
maybe: str | None = None                # 3.10+，等价于 Optional[str]
either: int | str = 1                   # 3.10+，等价于 Union[int, str]


def f(items: list[str], sep: str = ",") -> str:
    return sep.join(items)


def g() -> None:                        # 没有返回值就标 None
    print("hi")
```

三个演进要记住（这解释了为什么你在网上会看到两套写法）：

| 老写法（3.8 及以前） | 新写法 | 从哪个版本起 |
|---|---|---|
| `from typing import List` → `List[str]` | `list[str]` | 3.9 |
| `Optional[str]` | `str \| None` | 3.10 |
| `Union[int, str]` | `int \| str` | 3.10 |
| `TypeVar("T")` + `Generic[T]` | `def f[T](x: T) -> T` | 3.12 |

**新项目一律用新写法**，`ruff` 的 `UP` 规则组会自动帮你改。规则号是实测出来的——对 `Optional[str]` 报 **`UP045` Use `X | None` for type annotations**，对 `Union[int, str]` 报 **`UP007` Use `X | Y` for type annotations**，都带 `[*]` 标记（表示 `ruff check --fix` 能自动修）。第 10.12 节会把 `UP` 加进 `notekit` 的规则集。

### 7.2 `TypedDict`：给字典标上"有哪些键"

```python
from typing import TypedDict


class ToolCall(TypedDict):
    name: str
    args: dict[str, str]


def run_tool(call: ToolCall) -> str:
    return f"调用 {call['name']}，参数 {call['args']}"


print(" ", run_tool({"name": "get_weather", "args": {"city": "北京"}}))
```

输出：

```
  调用 get_weather，参数 {'city': '北京'}
```

Python 里到处是字典（JSON 解析结果、LLM 返回的工具调用），`dict[str, Any]` 这种标注等于没标。`TypedDict` 让 mypy 知道**有哪些键、每个键什么类型**，这样写错键名会被抓出来。

**它只是静态标注，运行时就是普通 dict**——没有任何开销，也没有任何校验。要**运行时**校验就用 pydantic（第 9.2 节）。

### 7.3 `Protocol`：结构化的接口

```python
from collections.abc import Iterable
from typing import Protocol


class Speaker(Protocol):
    """任何有 speak() -> str 的对象都算 Speaker，不需要显式继承。
    相当于 Kotlin 的 interface，但是『结构性』的：长得像就算是。
    """

    def speak(self) -> str: ...


class Dog:
    def speak(self) -> str:
        return "汪"


class Robot:
    def speak(self) -> str:
        return "beep"


def make_noise(things: Iterable[Speaker]) -> list[str]:
    return [t.speak() for t in things]


print(" ", make_noise([Dog(), Robot()]))
```

输出：

```
  ['汪', 'beep']
```

**这是 3.7 节鸭子类型的"正式化"**。`Dog` 和 `Robot` **都没有继承 `Speaker`**，但 mypy 认为它们符合——因为它们的方法签名对得上。这叫**结构化子类型**（structural subtyping）。

和 Kotlin 的 `interface` 的差别：Kotlin 是**名义**的（必须写 `: Speaker` 才算），Python 的 `Protocol` 是**结构**的（长得像就算）。好处是你可以给**第三方库的类**追加协议——你没法改人家的代码去实现你的接口，但可以定义一个 Protocol 说"我需要的是有 `read()` 方法的东西"。

`... ` 是 `Ellipsis` 字面量，在这里当"函数体省略"用（写 `pass` 也行，但 `...` 是 Protocol 和存根文件的惯例）。

`Iterable` 从 `collections.abc` 导入而不是 `typing`——后者从 3.9 起废弃了。**参数类型尽量标最宽松的抽象类型**（`Iterable` 而不是 `list`），返回值标最具体的（`list[str]` 而不是 `Iterable[str]`），这样调用方最自由。

### 7.4 泛型：3.12 起简单多了

```python
from collections.abc import Callable


def first_or(items: list[str], default: str) -> str:
    return items[0] if items else default


def apply_twice[T](fn: Callable[[T], T], value: T) -> T:
    """3.12 起可以直接写 [T]，不用先 TypeVar("T")。"""
    return fn(fn(value))


print(" ", first_or([], "空的"))
print(" ", apply_twice(lambda s: s + "!", "hi"))
print(" ", apply_twice(lambda n: n * 2, 3))
```

输出：

```
  空的
  hi!!
  12
```

`def apply_twice[T](...)` 就是 Kotlin 的 `fun <T> applyTwice(...)`，写法几乎一样（3.12 之前要先 `T = TypeVar("T")`，啰嗦得多）。

`Callable[[T], T]` 读作"接受一个 T、返回一个 T 的函数"——相当于 Kotlin 的 `(T) -> T`。第一个 `[]` 里是参数列表。

`lambda 参数: 表达式` 是匿名函数，**只能写一个表达式**（不能有语句、不能多行）。需要多行就老老实实 `def` 一个有名字的函数——这是 Python 有意的设计选择。

### 7.5 mypy：把错误抓回来

```python
def add(a: int, b: int) -> int:
    return a + b


print("  add('1', '2') =", add("1", "2"), " <- 运行得很欢，但 mypy 会报错")
```

Python 跑起来是这样：

```
  add('1', '2') = 12  <- 运行得很欢，但 mypy 会报错
```

`"1" + "2"` 是字符串拼接，结果 `"12"`。**Python 一声不吭。** 现在装上 mypy 再看：

```bash
uv run --python 3.12 --with mypy mypy j_typing.py
```

真实输出：

```
j_typing.py:70: error: Argument 1 to "add" has incompatible type "str"; expected "int"  [arg-type]
j_typing.py:70: error: Argument 2 to "add" has incompatible type "str"; expected "int"  [arg-type]
Found 2 errors in 1 file (checked 1 source file)
```

**这就是你在 Kotlin 里免费得到的东西，在 Python 里要花一条命令换。**

**结论（请当成纪律）**：类型提示本身不做任何事，`mypy`（或 pyright）才是执行者。**任何认真的 Python 项目都必须把 mypy 接进 CI**，否则类型标注只是好看的注释。第 10.12 节会把它配进 `pyproject.toml`。

`--with mypy` 是 uv 的临时依赖语法：装一次、用完扔，不写进项目依赖。适合这种一次性检查。

### 7.6 ✅ Checkpoint 7：类型提示 + mypy 抓错

```bash
uv run --python 3.12 python j_typing.py
uv run --python 3.12 --with mypy mypy j_typing.py
```

第一条的完整输出：

```
--- 1. TypedDict ---
  调用 get_weather，参数 {'city': '北京'}
--- 2. Protocol ---
  ['汪', 'beep']
--- 3. 泛型 ---
  空的
  hi!!
  12
--- 4. 运行时不检查 ---
  add('1', '2') = 12  <- 运行得很欢，但 mypy 会报错
  所以类型提示要配合 mypy/pyright 才有价值，
  它不是 Kotlin 那种编译期强制，而是『可选的静态检查』。
```

第二条必须**报 2 个错**。**一个成功、一个报错，两个都对，这才算这节学通了**——同一份代码，Python 说没问题，mypy 说有两处类型错误，而 mypy 是对的。

---
## 八、标准库与并发：GIL 到底挡了什么

### 8.1 标准库速查：这些不用装

Python 号称"自带电池"，下面这些 `import` 就能用。新建 `f_stdlib.py`。

**内置函数（连 import 都不用）**：

```python
names = ["小林", "小王", "小李"]
scores = [91, 78, 85]
for i, (n, s) in enumerate(zip(names, scores), start=1):
    print(f"  第{i}名候选: {n} {s}")
print("  按分数倒序:", sorted(zip(names, scores), key=lambda p: -p[1]))
print("  any/all:", any(s > 90 for s in scores), all(s > 60 for s in scores))
print("  max 带 key:", max(zip(names, scores), key=lambda p: p[1]))
print("  sum:", sum(scores), "| min:", min(scores))
```

输出：

```
  第1名候选: 小林 91
  第2名候选: 小王 78
  第3名候选: 小李 85
  按分数倒序: [('小林', 91), ('小李', 85), ('小王', 78)]
  any/all: True True
  max 带 key: ('小林', 91)
  sum: 254 | min: 78
```

| 函数 | 干什么 | Kotlin 对应 |
|---|---|---|
| `enumerate(xs, start=1)` | 同时拿下标和元素 | `withIndex()` |
| `zip(a, b)` | 把两个序列配对（**按最短的截断**） | `zip` |
| `sorted(xs, key=..., reverse=True)` | 返回**新**列表（`xs.sort()` 是原地改） | `sortedBy` |
| `any` / `all` | 有没有一个 / 是不是全部 | `any` / `all` |
| `max` / `min` / `sum` / `len` | 顾名思义，都支持 `key=` | 同名 |

`key=lambda p: p[1]` 是"按什么排"，加个负号 `-p[1]` 或写 `reverse=True` 就是倒序。注意 `enumerate` 的第二个参数叫 `start`，默认从 0 开始——**要从 1 开始给人看，必须显式写 `start=1`**。

**`collections`：四个高频容器**

```python
import collections

counter = collections.Counter("mississippi")
print("  Counter 最常见的3个:", counter.most_common(3))
dd = collections.defaultdict(list)
for word in ["apple", "avocado", "banana", "blueberry"]:
    dd[word[0]].append(word)
print("  defaultdict 分组:", dict(dd))
dq = collections.deque(maxlen=3)
for i in range(6):
    dq.append(i)
print("  deque(maxlen=3) 只留最后3个:", list(dq), "  <- 做对话历史窗口很好用")
Point = collections.namedtuple("Point", "x y")
p = Point(1, 2)
print("  namedtuple:", p, p.x, p.y)
```

输出：

```
  Counter 最常见的3个: [('i', 4), ('s', 4), ('p', 2)]
  defaultdict 分组: {'a': ['apple', 'avocado'], 'b': ['banana', 'blueberry']}
  deque(maxlen=3) 只留最后3个: [3, 4, 5]   <- 做对话历史窗口很好用
  namedtuple: Point(x=1, y=2) 1 2
```

- **`Counter`**：计数神器，`most_common(n)` 直接给出 Top-N。做词频、统计工具调用次数都用它。
- **`defaultdict(list)`**：访问不存在的键时**自动创建默认值**，省掉 `if k not in d: d[k] = []` 这种样板代码。
- **`deque(maxlen=N)`**：双端队列，**满了自动挤掉最老的**。这正是"只保留最近 N 轮对话"的标准实现——比手写 `if len(history) > N: history.pop(0)` 又快又不会错。
- **`namedtuple`**：轻量只读结构。现在一般优先用 `@dataclass`，但读老代码会遇到。

**`dict` 的常用操作**

```python
d = {"a": 1, "b": 2}
print("  get 带默认值:", d.get("c", 0))
print("  setdefault:", d.setdefault("c", 3), d)
print("  合并(3.9+):", d | {"d": 4})
print("  items 遍历:", [f"{k}={v}" for k, v in d.items()])
print("  dict 保持插入顺序(3.7+):", list({"z": 1, "a": 2, "m": 3}))
print("  pop 带默认值:", d.pop("nope", "没有这个键"))
```

输出：

```
  get 带默认值: 0
  setdefault: 3 {'a': 1, 'b': 2, 'c': 3}
  合并(3.9+): {'a': 1, 'b': 2, 'c': 3, 'd': 4}
  items 遍历: ['a=1', 'b=2', 'c=3']
  dict 保持插入顺序(3.7+): ['z', 'a', 'm']
  pop 带默认值: 没有这个键
```

**`d["missing"]` 会抛 `KeyError`，`d.get("missing")` 返回 `None`。** 处理 LLM 返回的 JSON 时永远用 `.get()` 加默认值——模型少给一个字段是常态。

**`dict` 从 3.7 起保证保持插入顺序**（输出第五行 `['z', 'a', 'm']` 就是证据，不是排序后的 `['a','m','z']`）。这个特性让"去重且保持顺序"有了一个惯用写法，第 10.3 节的标签去重就是这么做的。

**字符串处理**

```python
import re

line = "  Name: 小林 , Age: 28  "
print("  strip:", repr(line.strip()))
print("  split:", [s.strip() for s in line.split(",")])
print("  startswith/in:", line.strip().startswith("Name"), "Age" in line)
print("  replace:", "a-b-c".replace("-", "/"))
print("  partition:", "key: value".partition(":"))
print("  join:", ", ".join(names))
print("  ljust 对齐:", "|" + "名字".ljust(6) + "|")
print("  正则:", re.findall(r"\d+", line), re.sub(r"\s+", " ", line).strip())
m = re.search(r"Name:\s*(\S+)", line)
print("  分组:", m.group(1) if m else None)
```

输出：

```
  strip: 'Name: 小林 , Age: 28'
  split: ['Name: 小林', 'Age: 28']
  startswith/in: True True
  replace: a/b/c
  partition: ('key', ':', ' value')
  join: 小林, 小王, 小李
  ljust 对齐: |名字    |
  正则: ['28'] Name: 小林 , Age: 28
  分组: 小林
```

`strip()` 去两端空白、`split(",")` 切分、`join` 拼接、`partition(":")` **按第一个分隔符切成三段**（前、分隔符本身、后）——`partition` 比 `split` 安全，因为它永远返回 3 个元素，不用担心索引越界。

正则用 `r"..."`（raw 字符串，反斜杠不转义）：`re.findall` 找全部、`re.search` 找第一个（**返回 `None` 或匹配对象，必须判空**）、`re.sub` 替换、`m.group(1)` 取第一个捕获组。

**`match` 语句：3.10+ 的加强版 `when`**

```python
def describe(x):
    match x:
        case 0:
            return "零"
        case int() | float() if x < 0:
            return "负数"
        case int():
            return f"正整数 {x}"
        case [a, b]:
            return f"两个元素的列表: {a},{b}"
        case {"type": t, **rest}:
            return f"字典，type={t}，其余={rest}"
        case str() as s:
            return f"字符串 长度{len(s)}"
        case _:
            return "别的什么"


for v in [0, -3, 7, [1, 2], {"type": "tool", "name": "search"}, "hello", 3.14j]:
    print(f"  {v!r:>34} -> {describe(v)}")
```

输出：

```
                                   0 -> 零
                                  -3 -> 负数
                                   7 -> 正整数 7
                              [1, 2] -> 两个元素的列表: 1,2
  {'type': 'tool', 'name': 'search'} -> 字典，type=tool，其余={'name': 'search'}
                             'hello' -> 字符串 长度5
                               3.14j -> 别的什么
```

`match` 比 Kotlin 的 `when` 强的地方是**结构解构**：`case [a, b]` 直接匹配"两个元素的列表"并把元素绑到 `a`/`b`；`case {"type": t, **rest}` 匹配"含 type 键的字典"，把值绑到 `t`、剩下的收进 `rest`。**解析 LLM 返回的工具调用 JSON 时非常顺手。**

三个细节：`case int()` 是类型匹配（**带括号**，别写成 `case int`——那是匹配"等于 int 这个类对象"）；`if x < 0` 是守卫条件；`case _` 是兜底（相当于 `else`）。**从上往下第一个匹配的生效**，所以 `case 0` 必须写在 `case int()` 前面。

**时间：最容易出 bug 的地方**

```python
from datetime import datetime, timedelta, timezone

now = datetime.now(timezone.utc)
print("  带时区的现在:", now.isoformat()[:19] + "...")
print("  格式化:", now.strftime("%Y-%m-%d %H:%M"))
print("  加一天:", (now + timedelta(days=1)).strftime("%Y-%m-%d"))
print("  解析字符串:", datetime.strptime("2026-08-16", "%Y-%m-%d").date())
naive = datetime.now()
try:
    _ = naive < now
except TypeError as e:
    print("  报错:", e)
```

输出：

```
  带时区的现在: 2026-08-16T09:49:43...
  格式化: 2026-08-16 09:49
  加一天: 2026-08-17
  解析字符串: 2026-08-16
  不带时区的(naive)和带时区的不能直接比 —— 这是最常见的时间 bug
  报错: can't compare offset-naive and offset-aware datetimes
```

**这个报错请记住**：Python 的 datetime 分两种——**naive**（不带时区，`datetime.now()`）和 **aware**（带时区，`datetime.now(timezone.utc)`）。两者**不能比较、不能相减**，直接抛 `TypeError`。

**纪律：程序内部一律用带时区的 UTC 时间**（`datetime.now(UTC)`），只在显示给用户的最后一步转本地时区。第 10.3 节的 `Note.created_at` 就是这么做的。

（`UTC` 这个简写从 3.11 起可以直接 `from datetime import UTC`，比 `timezone.utc` 短。ruff 的 `UP017` 规则会自动帮你改。）

**`itertools`：迭代工具箱**

```python
import itertools

print("  chain:", list(itertools.chain([1, 2], [3, 4])))
print("  islice(取前3个，可用于无限序列):", list(itertools.islice(itertools.count(10), 3)))
print("  批量分组(3.12+ batched):", [list(b) for b in itertools.batched(range(7), 3)])
```

输出：

```
  chain: [1, 2, 3, 4]
  islice(取前3个，可用于无限序列): [10, 11, 12]
  批量分组(3.12+ batched): [[0, 1, 2], [3, 4, 5], [6]]
```

`chain` 把多个序列串成一个、`islice` 对**无限序列**也能安全取前 N 个、`batched(xs, n)` 按 n 个一组切分（**3.12 才有**，之前得自己写）。批量调用 API、批量做 embedding 时 `batched` 特别有用——注意最后一组可以不满（上面是 `[6]`）。

### 8.2 ✅ Checkpoint 8：标准库

```bash
uv run --python 3.12 python f_stdlib.py
```

7 个小节全部打印，最后一行应该是 `批量分组(3.12+ batched): [[0, 1, 2], [3, 4, 5], [6]]`。**如果这行报 `AttributeError: module 'itertools' has no attribute 'batched'`，说明你的 Python 不是 3.12+**，回到第 1 节。

---
### 8.3 是什么：GIL

**GIL（Global Interpreter Lock，全局解释器锁）** 是 CPython（就是官方那个 Python）里的一把大锁：**同一时刻只允许一个线程执行 Python 字节码。**

这句话的后果非常反直觉：**在 Python 里开 4 个线程做纯计算，不会比 1 个线程快，甚至可能更慢**（因为多了切换开销）。这和 Android 上开线程池加速计算的经验完全相反。

为什么会有它？简化了 CPython 内部内存管理（引用计数不用逐对象加锁），代价是牺牲了多核并行。**注意 GIL 是 CPython 的实现细节**，不是 Python 语言规范；3.13 起有一个实验性的"free-threaded"构建可以关掉它，但目前生产环境还不是默认。

### 8.4 怎么用：三种并发，各管一段

**别背结论，看实测。** 新建 `g_gil.py`——**注意所有多进程代码必须写在 `if __name__ == "__main__":` 里面**（原因在 8.5 讲，那是我踩过的坑）：

```python
"""G: GIL 实测 —— 线程 / 进程 / 异步该用哪个。

注意：多进程代码必须写在 if __name__ == "__main__": 里面，
     否则子进程会重新 import 这个文件，把上面的代码再跑一遍(甚至无限递归)。
"""
import multiprocessing
import os
import threading
import time


def cpu_heavy(n=6_000_000):
    """纯计算：一直在算，不等任何东西。"""
    total = 0
    for i in range(n):
        total += i * i
    return total


def io_heavy(delay=0.3):
    """纯等待：什么都不算，就是等。"""
    time.sleep(delay)
    return delay


def timed(label, fn):
    t0 = time.perf_counter()
    fn()
    print(f"  {label:<24} {time.perf_counter() - t0:.2f} 秒")


def run_threads(fn, k=4):
    ts = [threading.Thread(target=fn) for _ in range(k)]
    for t in ts:
        t.start()
    for t in ts:
        t.join()


if __name__ == "__main__":
    print(f"CPU 核心数: {os.cpu_count()}")

    print("--- CPU 密集型：多线程完全没用 ---")
    timed("单线程 跑 4 遍", lambda: [cpu_heavy() for _ in range(4)])
    timed("4 个线程", lambda: run_threads(cpu_heavy, 4))
    with multiprocessing.Pool(4) as pool:
        timed("4 个进程", lambda: pool.map(cpu_heavy, [6_000_000] * 4))

    print("--- IO 密集型：多线程立刻有效 ---")
    timed("单线程 跑 4 遍", lambda: [io_heavy() for _ in range(4)])
    timed("4 个线程", lambda: run_threads(io_heavy, 4))
```

跑：

```bash
uv run --python 3.12 python g_gil.py
```

真实输出（8 核 M 系列 Mac）：

```
CPU 核心数: 8
--- CPU 密集型：多线程完全没用 ---
  单线程 跑 4 遍                0.70 秒
  4 个线程                    0.67 秒
  4 个进程                    0.22 秒
--- IO 密集型：多线程立刻有效 ---
  单线程 跑 4 遍                1.22 秒
  4 个线程                    0.30 秒
```

**这四行数字是本节的全部结论**：

| 场景 | 单线程 | 4 线程 | 4 进程 | 结论 |
|---|---|---|---|---|
| **CPU 密集**（纯计算） | 0.70s | **0.67s** | **0.22s** | 线程几乎没提升；**进程 3 倍多** |
| **IO 密集**（等待） | 1.22s | **0.30s** | —— | 线程**4 倍**，立刻有效 |

- **CPU 密集用多线程 = 白干**（0.70 → 0.67，在测量噪声范围内）。GIL 让 4 个线程排队执行。
- **CPU 密集用多进程 = 真的快**（0.70 → 0.22，约 3.2 倍）。每个进程有自己的解释器和自己的 GIL，真正跑在不同核上。代价是进程启动慢、内存翻倍、**参数和返回值要能被 pickle 序列化**（lambda、局部函数、打开的文件对象都不行）。
- **IO 密集用多线程 = 真的快**（1.22 → 0.30，约 4 倍）。因为线程在 `time.sleep` / 等网络时**会释放 GIL**，让别的线程跑。

第三种选择是 **`asyncio`**（6.6 节），它也解决 IO 等待，但不开线程——单线程内靠事件循环切换。相比多线程：省内存（协程比线程轻几个数量级）、没有锁的问题、能开成千上万个；代价是**整条调用链都得是异步的**（一个阻塞调用就毁掉全部，6.7 节的实测）。

**决策表（背这个就够了）**：

| 你的任务 | 用什么 | 本章哪里有例子 |
|---|---|---|
| 等网络 / 等 API / 等数据库（**Agent 的主要场景**） | **`asyncio`**，其次线程池 | 6.6、9.3、11.4 |
| 纯计算（矩阵、编解码、大量字符串处理） | **多进程**（`multiprocessing` / `ProcessPoolExecutor`） | 8.4 |
| 只有同步 SDK，但要在异步代码里用 | `await asyncio.to_thread(...)` | 6.7 表格 |
| Web 服务要吃满多核 | **多进程**（`uvicorn --workers 4`） | 12.2 |

> **一个反直觉但重要的推论**：写 Agent 时你几乎永远不需要 `multiprocessing`。Agent 的时间 99% 花在**等大模型返回**上，那是 IO 等待，`asyncio` 就够了。GIL 对你的实际影响远小于传闻。

### 8.5 踩坑实录：多进程会把你的文件重跑一遍

我最初把 GIL 实验和标准库演示写在**同一个文件**里，结果输出是这样的：**第 1~7 节的内容打印了三遍。**

原因：`multiprocessing` 在 macOS 和 Windows 上默认用 **spawn** 方式创建子进程——子进程会**重新 import 主模块**来获得函数定义。于是模块顶层的所有代码（包括那 7 节演示）在每个子进程里又跑了一遍。

**解法就是 5.2 节的规则三**：把要执行的代码放进 `if __name__ == "__main__":`。子进程 import 主模块时 `__name__` 是 `"__mp_main__"` 而不是 `"__main__"`，所以那一块不会被执行。

**这不是风格问题，是正确性问题。** 我最后把 GIL 实验单独拆成 `g_gil.py`，整个执行部分都在 `if __name__ == "__main__":` 里面（上面的代码就是拆完的版本）。**任何用到 `multiprocessing` 的文件都必须这么写**，否则你会看到重复输出，最坏情况是无限递归创建进程。

---
## 九、主要库：一张表 + 三个最小例子

到这里语法部分结束。这一节先概览生态，第 10 节把它们全用起来。

### 9.1 六个库，六件事

| 库 | 干什么 | Android 类比 | 为什么是它 |
|---|---|---|---|
| **pydantic** | 数据校验与序列化 | Moshi/Gson + 校验 | 事实标准，FastAPI 和 LangChain 都建在它上面 |
| **typer** | 命令行参数解析 | —— | 靠**类型提示**自动生成 CLI 和 `--help` |
| **rich** | 终端彩色输出、表格、进度条 | —— | 让 CLI 好看，typer 自带 |
| **httpx** | HTTP 客户端 | Retrofit/OkHttp | 比 `requests` 多了**异步**支持 |
| **FastAPI** | Web 框架 | Spring Boot | 靠类型提示自动校验 + 自动生成 API 文档 |
| **pytest** | 测试框架 | JUnit | 断言直接写 `assert` |
| **ruff** | 格式化 + 静态检查 | ktlint + detekt | Rust 写的，快到几乎无感 |
| **mypy** | 类型检查 | Kotlin 编译器（的类型部分） | 见第 7 节 |

还有几个你会经常听到的：`python-dotenv`（读 `.env`）、`uvicorn`（跑 FastAPI 的服务器，相当于 Tomcat）、`numpy`/`pandas`（数值和表格计算）、`openai`（各家大模型的兼容 SDK）。

### 9.2 pydantic：带校验的 data class

```python
from pydantic import BaseModel, Field, ValidationError


class NoteCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    body: str = ""
    tags: list[str] = Field(default_factory=list)


ok = NoteCreate(title="学 Python", tags=["py"])
print(ok)
print(ok.model_dump())

try:
    NoteCreate(title="")
except ValidationError as e:
    print("校验失败:", e.errors()[0]["type"], e.errors()[0]["loc"])
```

**pydantic 和 `@dataclass` 的关键差别：它会在运行时真的校验和转换类型。** `@dataclass` 只是生成样板代码，你传什么它都收。

三个能力：**`Field(min_length=..., max_length=..., ge=..., le=...)`** 声明约束；**类型会在"安全"的前提下自动转换**（把 `"42"` 传给 `int` 字段会得到 `42`，但把 `123` 传给 `str` 字段**会被拒绝**——具体规则见下面的 Checkpoint 9）；**`model_dump()`** 转成 dict、**`model_validate(d)`** 从 dict 还原、**`model_dump_json()`** 直接出 JSON 字符串。

pydantic 有 v1 和 v2 两代，**API 名字全变了**（`.dict()` → `.model_dump()`、`.parse_obj()` → `.model_validate()`、`@validator` → `@field_validator`）。**网上搜到的教程一半是 v1 的**，看到 `.dict()` 就知道那是旧的。本章全部用 v2。

### 9.3 httpx：同步和异步两副面孔

```python
import httpx

# 同步
with httpx.Client(base_url="http://127.0.0.1:8123", timeout=5.0) as client:
    r = client.get("/health")
    print(r.status_code, r.json())

# 异步
async def main():
    async with httpx.AsyncClient(base_url="http://127.0.0.1:8123") as client:
        r = await client.get("/health")
        return r.json()
```

**四个必须知道的点**：

1. **用 `with httpx.Client(...)` 而不是每次 `httpx.get(...)`**。Client 会复用 TCP 连接（连接池），高频调用时差别很大——相当于复用 OkHttpClient 而不是每次新建。
2. **`timeout` 必须显式传**。不传的默认值是 5 秒，但更重要的是养成习惯：**任何网络调用都要有超时**，否则一个卡住的请求能挂死整个服务。
3. **4xx/5xx 不会自动抛异常**，`r.status_code` 就是 404 而已。想抛得自己调 `r.raise_for_status()`。这和 Retrofit 的行为不同，是新手最容易漏的地方。
4. **`httpx` vs `requests`**：API 几乎一样，但 httpx 支持 `async`。**新项目直接用 httpx。**

### 9.4 FastAPI：类型提示当校验规则

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, Query

api = FastAPI(title="notekit API", version="0.1.0")


@api.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

FastAPI 的核心卖点：**你写的类型提示就是校验规则和 API 文档**。声明 `note_id: int`，传 `abc` 会自动返回 422 和详细错误；声明请求体是某个 pydantic 模型，字段校验自动生效；**`/docs` 自动生成可交互的 API 文档**，不用写一行 Swagger 注解。

第 10.9 节会完整实现，第 10.10 节会看到 422 的真实响应体。

### 9.5 ✅ Checkpoint 9：pydantic 的校验与转换规则

```bash
uv run --python 3.12 --with pydantic python -c "
from pydantic import BaseModel, Field, ValidationError

class NoteCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    tags: list[str] = Field(default_factory=list)

class Score(BaseModel):
    n: int

print('正常:', NoteCreate(title='学 Python', tags=['py']))
print('字符串数字会转成 int:', Score(n='42'))
try:
    NoteCreate(title=123)
except ValidationError as e:
    print('int 传给 str 字段被拒绝:', e.errors()[0]['type'])
try:
    NoteCreate(title='')
except ValidationError as e:
    print('空标题被拒绝:', e.errors()[0]['type'])
try:
    Score(n='abc')
except ValidationError as e:
    print('转不了的字符串被拒绝:', e.errors()[0]['type'])
"
```

真实输出：

```
正常: title='学 Python' tags=['py']
字符串数字会转成 int: n=42
int 传给 str 字段被拒绝: string_type
空标题被拒绝: string_too_short
转不了的字符串被拒绝: int_parsing
```

**这四种结果值得对照记住**：pydantic 的默认（宽松）模式**只做"无损且无歧义"的转换**——`"42"` → `42` 可以（JSON 里数字常被当字符串传），但 `123` → `"123"` **不行**（那会掩盖类型错误），`"abc"` → int 更不行。约束（`min_length`）在转换之后才检查。

每个错误都有一个稳定的 `type` 字符串（`string_type` / `string_too_short` / `int_parsing`），**这个字段可以直接用来做前端错误提示的分支判断**，比解析错误文本可靠得多。第 10.10 节会在 HTTP 422 的响应体里再看到它。

---
## 十、动手实战：从空目录敲出 `notekit`

**这一节是本章的主线。** 我们要做一个笔记工具 `notekit`，它有两副面孔——命令行和 HTTP API——**但共用同一套业务逻辑**。做完你会得到一个"结构正确的 Python 项目"的样板，以后所有项目都可以照着搭。

分层设计（**严格单向依赖，这是避免循环导入的根本办法**）：

```
models.py   ← 数据长什么样（pydantic）        谁都不依赖
   ↑
store.py    ← 数据存哪里、怎么增删改查        只依赖 models
   ↑
config.py   ← 配置从哪读（环境变量/.env）     谁都不依赖
   ↑
cli.py      ← 命令行界面（typer + rich）      依赖 store/models/config
web.py      ← HTTP 接口（FastAPI）            依赖 store/models/config
```

对照 Android：`models` 是数据类、`store` 是 Repository、`cli`/`web` 是两个不同的"UI 层"。**注意 `store` 完全不知道谁在用它**，所以 CLI 和 Web 能共享它，测试也能直接测它。

### 10.1 `uv init`：生成项目骨架

```bash
uv init --package notekit --python 3.12
cd notekit
```

真实输出：

```
Initialized project `notekit` at `/private/tmp/verify/py/notekit`
```

生成的文件（我用一个同样的 `initdemo` 演示，结构完全一致）：

```
initdemo/
├── .git/               # 顺手初始化了 git 仓库
├── .gitignore
├── .python-version     # 内容就是 3.12
├── README.md
├── pyproject.toml
└── src/
    └── initdemo/
        └── __init__.py
```

`pyproject.toml` 的初始内容：

```toml
[project]
name = "initdemo"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []

[project.scripts]
initdemo = "initdemo:main"

[build-system]
requires = ["uv_build>=0.11.32,<0.12.0"]
build-backend = "uv_build"
```

**逐块解释这个文件**（它就是你的 `build.gradle.kts`）：

| 块 | 作用 | Gradle 对应 |
|---|---|---|
| `[project]` name/version | 包名和版本 | `group` / `version` |
| `requires-python` | 最低 Python 版本 | `compileSdk` / `jvmTarget` |
| `dependencies` | 运行时依赖 | `implementation` |
| `[project.scripts]` | **装完之后有哪些命令行命令** | 无直接对应（有点像 manifest 声明入口） |
| `[build-system]` | 用什么工具打包 | Gradle 插件 |

`[project.scripts]` 里的 `initdemo = "initdemo:main"` 读法是"**命令名 = 模块:函数**"。装好之后敲 `initdemo` 就等于调用 `initdemo` 包里的 `main()` 函数。生成的 `__init__.py` 里正好有一个：

```python
def main() -> None:
    print("Hello from initdemo!")
```

试一下（**第一次 `uv run` 会顺手建好虚拟环境并把项目装进去**）：

```bash
uv run initdemo
```

真实输出（我用 `initcheck` 这个名字跑的，你的名字换成你的）：

```
Using CPython 3.12.13 interpreter at: /opt/homebrew/opt/python@3.12/bin/python3.12
Creating virtual environment at: .venv
   Building initcheck @ file:///private/tmp/verify/py/initcheck
      Built initcheck @ file:///private/tmp/verify/py/initcheck
Installed 1 package in 3ms
Hello from initcheck!
```

**注意前四行**：`uv run` 发现还没有 `.venv`，就自动建了一个、把你的项目以"可编辑模式"装进去、然后才执行命令。**你没有敲任何"创建环境"或"安装"的命令**——这就是 1 节说的 `uv run` 会自己保证环境一致。对照 Gradle：相当于 `./gradlew run` 会自动 sync 依赖。

**为什么是 `src/` 布局（src-layout）？** 源码放在 `src/包名/` 下而不是直接放项目根目录，好处是**测试时被迫用"已安装的包"而不是"当前目录的文件"**——这样能提前发现"我忘了把某个文件加进包里"这类打包错误。这是现在 Python 社区的推荐做法。

`.gitignore` 也自动生成好了：

```
# Python-generated files
__pycache__/
*.py[oc]
build/
dist/
wheels/
*.egg-info

# Virtual environments
.venv
```

### 10.2 装依赖

```bash
uv add "pydantic>=2" "typer" "fastapi" "uvicorn[standard]" "httpx" "python-dotenv"
uv add --dev pytest ruff mypy
```

`uv add` 会做三件事：写进 `pyproject.toml` → 更新 `uv.lock` → 装进 `.venv`。相当于改 `build.gradle` + sync 一步完成。真实输出长这样（以 `uv add "rich>=14"` 为例）：

```
Resolved 5 packages in 1.08s
   Building initcheck @ file:///private/tmp/verify/py/initcheck
      Built initcheck @ file:///private/tmp/verify/py/initcheck
Prepared 1 package in 9ms
Uninstalled 1 package in 1ms
Installed 5 packages in 14ms
 ~ initcheck==0.1.0 (from file:///private/tmp/verify/py/initcheck)
 + markdown-it-py==4.2.0
 + mdurl==0.1.2
 + pygments==2.20.0
 + rich==15.0.0
```

**我只要了 `rich` 一个包，装进来 4 个**——`markdown-it-py`/`mdurl`/`pygments` 是 rich 自己的依赖。`~ initcheck` 那行是你自己的项目被重新装了一遍（因为 `pyproject.toml` 变了）。

**`--dev` 的依赖只在开发时用**（测试、检查工具），打包发布时不会带上——就是 `testImplementation`。它们会被写进 `[dependency-groups]` 的 `dev` 组。

**`uvicorn[standard]` 里的方括号是"额外功能"（extras）**：装 uvicorn 时顺带装上性能更好的可选组件（`uvloop`、`httptools`、`watchfiles`）。这个语法你会经常遇到，比如 `pydantic[email]`、`fastapi[all]`。

装完看一眼依赖树：

```bash
uv tree --depth 1
```

真实输出：

```
Resolved 40 packages in 2ms
notekit v0.1.0
├── fastapi v0.141.1
├── httpx v0.28.1
├── pydantic v2.13.4
├── python-dotenv v1.2.2
├── typer v0.27.1
├── uvicorn[standard] v0.52.3
├── httpx v0.28.1 (group: dev)
├── mypy v2.3.1 (group: dev)
├── pytest v9.1.1 (group: dev)
└── ruff v0.16.3 (group: dev)
```

**我直接声明了 6 个包，实际装了 40 个**——其余都是传递依赖。`--depth 1` 只看直接依赖；去掉它能看到完整的树。

（你跑的时候版本号会更新，这正常。锁文件的意义就是**你和别人跑的是同一套版本**，见 11.7。）

### 10.3 第一层：`models.py`（数据长什么样）

`src/notekit/models.py`：

```python
"""数据模型：用 Pydantic 定义"一条笔记长什么样"。"""

from datetime import UTC, datetime

from pydantic import BaseModel, Field, field_validator


def _now() -> datetime:
    return datetime.now(UTC)


class Note(BaseModel):
    """一条笔记。Pydantic 会在创建时自动校验+转换类型。"""

    id: int
    title: str = Field(min_length=1, max_length=100)
    body: str = ""
    tags: list[str] = Field(default_factory=list)
    done: bool = False
    created_at: datetime = Field(default_factory=_now)

    @field_validator("tags")
    @classmethod
    def strip_tags(cls, v: list[str]) -> list[str]:
        """自定义校验：去空格、去空串、去重，且保持顺序。"""
        seen: dict[str, None] = {}
        for tag in v:
            t = tag.strip()
            if t:
                seen[t] = None
        return list(seen)


class NoteCreate(BaseModel):
    """新建笔记时客户端要传的字段——注意没有 id 和 created_at，那是服务端生成的。"""

    title: str = Field(min_length=1, max_length=100)
    body: str = ""
    tags: list[str] = Field(default_factory=list)


class NoteUpdate(BaseModel):
    """更新时所有字段都可选：只传想改的那几个。
    `str | None` 是 Python 3.10+ 的写法，等价于老写法 Optional[str]。
    """

    title: str | None = Field(default=None, min_length=1, max_length=100)
    body: str | None = None
    tags: list[str] | None = None
    done: bool | None = None
```

**五个设计决策，每个都有理由**：

1. **三个模型而不是一个**。`Note` 是完整实体（有 id、有创建时间）、`NoteCreate` 是"客户端能传什么"（**没有 id**——不能让客户端指定 id）、`NoteUpdate` 是"能改什么"（全部可选）。这是 Web API 的标准做法，避免客户端伪造服务端字段。
2. **`default_factory=_now`** 而不是 `default=_now()`。后者会在**类定义时**求值一次，导致所有笔记的创建时间都一样——这就是 2.10 节可变默认参数陷阱的另一种形态。
3. **`datetime.now(UTC)`** 而不是 `datetime.now()`。8.1 节实测过 naive 和 aware 不能比较，存 UTC 是唯一不会出错的选择。
4. **`field_validator` 里用 dict 当有序集合**。`seen[t] = None` 然后 `list(seen)` —— 因为 8.1 节说过 dict 从 3.7 起保持插入顺序，所以这是"去重且保序"的标准惯用法（`set` 会丢顺序）。
5. **`@field_validator` 下面必须紧跟 `@classmethod`**，顺序不能反。这是 pydantic v2 的要求（v1 不需要，是常见的迁移坑）。

### 10.4 第二层：`store.py`（数据存哪里）

`src/notekit/store.py`：

```python
"""存储层：把笔记存进一个 JSON 文件。只有这一层知道文件在哪。"""

import json
from pathlib import Path

from .models import Note, NoteCreate, NoteUpdate


class NoteStore:
    def __init__(self, path: Path):
        self.path = path
        self._notes: list[Note] = []
        self._load()

    # ---------- 私有方法：约定用下划线开头 ----------
    def _load(self) -> None:
        if not self.path.exists():
            self._notes = []
            return
        raw = json.loads(self.path.read_text(encoding="utf-8"))
        self._notes = [Note.model_validate(item) for item in raw]

    def _save(self) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        payload = [n.model_dump(mode="json") for n in self._notes]
        self.path.write_text(json.dumps(payload, ensure_ascii=False, indent=2), encoding="utf-8")

    def _next_id(self) -> int:
        return max((n.id for n in self._notes), default=0) + 1

    # ---------- 公开 API ----------
    def add(self, data: NoteCreate) -> Note:
        note = Note(id=self._next_id(), **data.model_dump())
        self._notes.append(note)
        self._save()
        return note

    def list_all(self, tag: str | None = None, done: bool | None = None) -> list[Note]:
        result = self._notes
        if tag is not None:
            result = [n for n in result if tag in n.tags]
        if done is not None:
            result = [n for n in result if n.done is done]
        return result

    def get(self, note_id: int) -> Note | None:
        return next((n for n in self._notes if n.id == note_id), None)

    def update(self, note_id: int, data: NoteUpdate) -> Note | None:
        note = self.get(note_id)
        if note is None:
            return None
        changes = data.model_dump(exclude_unset=True)  # 只取调用方真正传了的字段
        updated = note.model_copy(update=changes)
        Note.model_validate(updated.model_dump())  # 改完再校验一次
        self._notes[self._notes.index(note)] = updated
        self._save()
        return updated

    def delete(self, note_id: int) -> bool:
        note = self.get(note_id)
        if note is None:
            return False
        self._notes.remove(note)
        self._save()
        return True

    def search(self, keyword: str) -> list[Note]:
        kw = keyword.lower()
        return [n for n in self._notes if kw in n.title.lower() or kw in n.body.lower()]
```

**这一层有六个值得停下来看的点**：

1. **`model_dump(mode="json")`** 而不是 `model_dump()`。前者把 `datetime` 转成字符串，后者留着 `datetime` 对象——而 `json.dumps` 不认识 `datetime`，会抛 `TypeError: Object of type datetime is not JSON serializable`。**这是新手 100% 会撞一次的错。**

2. **`Note(id=..., **data.model_dump())`**。`**字典` 是"把字典展开成关键字参数"，和 3.1 节的 `**kwargs` 是一对（一个打包、一个解包）。

3. **`max((...), default=0)`**：空列表调 `max()` 会抛 `ValueError`，`default=0` 是空集合时的兜底。**任何对可能为空的集合调 `max`/`min` 都要写 `default=`。**

4. **`next((n for n in ... if ...), None)`**：这是"找第一个满足条件的，找不到返回 None"的标准写法。里面是生成器表达式（6.5 节），**所以是短路的**——找到就停，不会遍历完整个列表。

5. **`update` 用 `exclude_unset=True`——这是 PATCH 语义的核心。** 假设调用方只传了 `{"done": true}`：
   - `model_dump()` 会返回**所有**字段（`title=None, body=None, tags=None, done=True`），拿去更新会把标题清空！
   - `model_dump(exclude_unset=True)` 只返回 `{"done": True}`，其余字段保持原值。

   **`exclude_unset` 区分的是"显式传了 None"和"根本没传"**——这正是 2.5 节说的"要区分 0/None 必须用 `is None`"在真实业务里的形态。`model_copy(update=...)` 生成一个改了这几个字段的新对象（不改原对象）。

6. **`model_copy` 不会重新校验，所以后面补一句 `Note.model_validate(...)`。** 这是 pydantic 的一个真实陷阱：`model_copy(update={"title": ""})` 会**默默地**产生一个标题为空的对象，绕过 `min_length=1`。补一次显式校验才安全。

> **一个我真踩了的坑**：这个类的列表方法最初我命名为 `list`，结果 `store.py` 里下面的 `def search(...) -> list[Note]:` 直接报 `TypeError: 'function' object is not subscriptable`。原因是**类体内部 `list` 这个名字已经被方法定义遮住了**，`list[Note]` 变成了"给方法对象加下标"。改名成 `list_all` 就好了。**教训：别用 `list`/`dict`/`type`/`id`/`str` 这些内置名字当方法名或变量名。**（顺带一提，`id` 作为 pydantic 字段名是安全的，因为字段名不参与类体的名字解析。）

### 10.5 第三层：`config.py`（配置与密钥）

`src/notekit/config.py`：

```python
"""配置：从环境变量 / .env 读，绝不硬编码。"""

import os
from pathlib import Path

from dotenv import load_dotenv

load_dotenv()  # 把 .env 里的键值对塞进 os.environ


def data_file() -> Path:
    """笔记存哪个文件。默认 ./notes.json，可用 NOTEKIT_DATA 覆盖。"""
    return Path(os.getenv("NOTEKIT_DATA", "notes.json"))


def api_key() -> str | None:
    """LLM 的 Key。只从环境变量读，代码里一个字符都不留。"""
    return os.getenv("LLM_API_KEY")
```

建 `.env`（**只在本机，绝不提交**）：

```
LLM_API_KEY=sk-你的真实密钥
NOTEKIT_DATA=./data/notes.json
```

**立刻把它加进 `.gitignore`**：

```bash
echo ".env" >> .gitignore
```

现在 `.gitignore` 末尾应该是：

```
# 密钥绝对不能进版本库
.env
```

验证读取逻辑（**注意：只打印长度和前几位，永远不要把完整密钥打印出来**，日志和终端记录都会留痕）：

```bash
uv run --python 3.12 --with python-dotenv python -c "
import os
from dotenv import load_dotenv
load_dotenv()
k = os.getenv('DEMO_API_KEY')
print('读到了吗:', k is not None)
print('长度:', len(k))
print('前 6 位:', k[:6] + '...')
print('数据文件:', os.getenv('NOTEKIT_DATA'))
print('没配的键返回:', os.getenv('NOT_SET_KEY'))
"
```

用一个假密钥 `DEMO_API_KEY=sk-this-is-a-fake-key-for-teaching` 演示，真实输出：

```
读到了吗: True
长度: 34
前 6 位: sk-thi...
数据文件: /tmp/verify/py/envdemo/notes.json
没配的键返回: None
```

**密钥纪律（这几条是硬性的，前面章节反复强调过）**：

- **密钥只放 `.env` 或环境变量，代码里一个字符都不留。**
- **`.env` 必须写进 `.gitignore`。** 提交历史里的密钥即使后来删了也还在历史里，等于已经泄露。
- **前端代码里永远不能出现 LLM 密钥**——浏览器里的东西所有人都能看到，必须由后端代理。
- **一旦泄露，立刻去平台吊销并重发**，别指望"应该没人看到"。
- **打印/记日志时只打印长度和前缀**（像上面那样）。这也是为什么我在这里用了一个假密钥来演示。

`os.getenv("KEY", "默认值")` 是"没配就用默认值"，不传默认值时返回 `None`。**永远不要写 `os.environ["KEY"]`**——那在没配时直接抛 `KeyError`，错误信息还不告诉你是哪个配置漏了。

### 10.6 ✅ Checkpoint 10：三层能单独跑通

```bash
uv run python -c "
from pathlib import Path
from notekit.models import NoteCreate, NoteUpdate
from notekit.store import NoteStore

s = NoteStore(Path('/tmp/ck10.json'))
a = s.add(NoteCreate(title='第一条', tags=[' py ', 'py', '', 'agent']))
print('id 自增:', a.id, '| 标签去重去空格:', a.tags)
b = s.add(NoteCreate(title='第二条'))
print('第二条 id:', b.id)
u = s.update(a.id, NoteUpdate(done=True))
print('只改 done，标题没丢:', u.done, u.title)
print('按标签过滤:', [n.title for n in s.list_all(tag='py')])
print('重新打开还在:', [n.title for n in NoteStore(Path('/tmp/ck10.json')).list_all()])
"
```

预期输出（这是设计意图的完整体现）：

```
id 自增: 1 | 标签去重去空格: ['py', 'agent']
第二条 id: 2
只改 done，标题没丢: True 第一条
按标签过滤: ['第一条']
重新打开还在: ['第一条', '第二条']
```

**第三行是关键**：`update` 只传了 `done`，标题必须还是"第一条"。如果你看到标题变成 `None`，说明 `exclude_unset=True` 漏了。

---
### 10.7 第四层：`cli.py`（命令行界面）

`src/notekit/cli.py`：

```python
"""命令行入口：用 Typer 把函数变成子命令。"""

import typer
from rich.console import Console
from rich.table import Table

from . import config
from .models import NoteCreate, NoteUpdate
from .store import NoteStore

app = typer.Typer(help="notekit —— 一个命令行笔记工具", no_args_is_help=True)
console = Console()


def _store() -> NoteStore:
    return NoteStore(config.data_file())


@app.command()
def add(
    title: str = typer.Argument(..., help="笔记标题"),
    body: str = typer.Option("", "--body", "-b", help="笔记正文"),
    tag: list[str] = typer.Option([], "--tag", "-t", help="标签，可重复"),
) -> None:
    """新增一条笔记。"""
    note = _store().add(NoteCreate(title=title, body=body, tags=tag))
    console.print(f"[green]已添加[/green] #{note.id} {note.title}")


@app.command("list")
def list_notes(
    tag: str = typer.Option(None, "--tag", "-t"),
    undone: bool = typer.Option(False, "--undone", help="只看未完成"),
) -> None:
    """列出笔记。"""
    notes = _store().list_all(tag=tag, done=False if undone else None)
    if not notes:
        console.print("[yellow]一条笔记都没有[/yellow]")
        raise typer.Exit(code=0)

    table = Table("ID", "状态", "标题", "标签")
    for n in notes:
        table.add_row(str(n.id), "✔" if n.done else "…", n.title, ",".join(n.tags))
    console.print(table)


@app.command()
def done(note_id: int = typer.Argument(..., help="要标记完成的笔记 ID")) -> None:
    """把一条笔记标为完成。"""
    note = _store().update(note_id, NoteUpdate(done=True))
    if note is None:
        console.print(f"[red]找不到 #{note_id}[/red]")
        raise typer.Exit(code=1)
    console.print(f"[green]已完成[/green] #{note.id} {note.title}")


@app.command()
def delete(note_id: int) -> None:
    """删除一条笔记。"""
    if not _store().delete(note_id):
        console.print(f"[red]找不到 #{note_id}[/red]")
        raise typer.Exit(code=1)
    console.print(f"[green]已删除[/green] #{note_id}")


@app.command()
def search(keyword: str) -> None:
    """按关键词搜索标题和正文。"""
    hits = _store().search(keyword)
    console.print(f"命中 {len(hits)} 条")
    for n in hits:
        console.print(f"  #{n.id} {n.title} — {n.body[:20]}")


@app.command()
def serve(
    host: str = typer.Option("127.0.0.1", help="监听地址，默认只允许本机访问"),
    port: int = typer.Option(8000, help="监听端口"),
    reload: bool = typer.Option(False, "--reload", help="改代码自动重启，仅开发用"),
) -> None:
    """启动 Web API 服务。"""
    import uvicorn

    console.print(f"[cyan]http://{host}:{port}/docs[/cyan] 可以看交互式文档")
    uvicorn.run("notekit.web:api", host=host, port=port, reload=reload)


def main() -> None:
    app()
```

**Typer 的核心思想：函数签名就是命令行接口。** 参数名变成选项名、类型注解变成解析规则、docstring 变成帮助文本。你不用写任何"解析 argv"的代码。

| 写法 | 命令行上长什么样 |
|---|---|
| `typer.Argument(...)` | **位置参数**，必填：`notekit add "标题"` |
| `typer.Option("", "--body", "-b")` | **选项**，可选：`--body "正文"` 或 `-b "正文"` |
| `tag: list[str] = typer.Option([])` | **可重复选项**：`-t a -t b` → `['a','b']` |
| `undone: bool = typer.Option(False)` | **开关**：写了 `--undone` 就是 True |
| `@app.command("list")` | 命令名和函数名不同（因为 `list` 是内置名） |

四个细节：

1. **`typer.Argument(...)` 里的 `...`** 是真实的 Python 对象 `Ellipsis`，Typer 用它表示"必填"。看着奇怪，但这是约定。
2. **`raise typer.Exit(code=1)`** 是退出并返回非 0 状态码。**这对脚本很重要**——`notekit delete 99 && echo ok` 不会打印 ok。Android 里你不用管进程退出码，命令行工具必须管。
3. **`import uvicorn` 写在 `serve()` 函数内部**，不在文件顶部。因为 uvicorn 启动慢（要加载一堆东西），放在函数里可以让 `notekit add` 这类命令**不为它付启动开销**。这就是 5.3 节说的延迟导入。
4. **`@app.command()` 的顺序决定 `--help` 里的显示顺序**，不是字母序。

> **一个 ruff 会报的警告**：`typer.Option(...)` 写在默认值位置，会被 ruff 的 `B008`（"不要在默认参数里调用函数"）规则拦下——因为 2.10 节那个可变默认参数陷阱正是这条规则要防的。但 Typer 的设计**要求**这么写。解决办法是在 `pyproject.toml` 里为这个文件单独放行：
> ```toml
> [tool.ruff.lint.per-file-ignores]
> "src/notekit/cli.py" = ["B008"]
> ```
> **这是 lint 工具的正确用法：不是关掉规则，而是为有正当理由的地方精确放行。**

> **`rich` 为什么不在 `dependencies` 里？** 因为它是 `typer` 的依赖，被顺带装进来了（`uv tree --depth 2` 能看到 `typer v0.27.1 → rich v15.0.0`）。**但直接 `import` 一个"别人的依赖"是有风险的**——typer 哪天换掉 rich，你的代码就断了。教学项目里这么写是为了少一行，**真实项目请把你直接 import 的每个包都显式写进 `dependencies`**（`uv add rich`）。这条规则叫"显式依赖优于传递依赖"，和 Gradle 里 `implementation` 不该被下游依赖者当 API 用是同一个道理。

**还差最后一根线：把 `main` 接到包上。** `pyproject.toml` 里写的是 `notekit = "notekit:main"`，也就是"命令 `notekit` 去调 **`notekit` 包**（不是 `notekit.cli` 模块）里的 `main`"。而 `uv init --package` 生成的 `src/notekit/__init__.py` 里那个 `main` 还是打印 Hello 的占位版本，**要把它换掉**：

```python
"""notekit —— 一个命令行 + Web 的笔记工具。"""

__version__ = "0.1.0"


def main() -> None:
    from .cli import main as cli_main

    cli_main()
```

**两点**：

- **`from .cli import ...` 写在函数里**，理由和 `import uvicorn` 一样——`import notekit` 只是为了读 `__version__` 时，不该顺带把 typer/rich 全加载一遍。
- **`__version__` 在这里定义**，§11.6 会讲怎么让它和 `pyproject.toml` 里的版本号保持单一来源（现在是两处，**这是个已知的隐患**）。

跑起来看看。先让数据存到一个临时文件（避免污染当前目录）：

```bash
export NOTEKIT_DATA=/tmp/ck_cli.json
uv run notekit --help
```

真实输出（这个漂亮的框是 rich 画的，Typer 自动集成）：

```
 Usage: notekit [OPTIONS] COMMAND [ARGS]...

 notekit —— 一个命令行笔记工具

╭─ Options ────────────────────────────────────────────────────────────────────╮
│ --install-completion          Install completion for the current shell.      │
│ --show-completion             Show completion for the current shell, to copy │
│                               it or customize the installation.             │
│ --help                        Show this message and exit.                   │
╰──────────────────────────────────────────────────────────────────────────────╯
╭─ Commands ───────────────────────────────────────────────────────────────────╮
│ add     新增一条笔记。                                                       │
│ list    列出笔记。                                                           │
│ done    把一条笔记标为完成。                                                 │
│ delete  删除一条笔记。                                                       │
│ search  按关键词搜索标题和正文。                                             │
│ serve   启动 Web API 服务。                                                  │
╰──────────────────────────────────────────────────────────────────────────────╯
```

**`--install-completion` 是白送的**：装上之后按 Tab 能补全子命令。

### 10.8 ✅ Checkpoint 11：完整走一遍 CLI

```bash
export NOTEKIT_DATA=/tmp/ck_cli.json
uv run notekit add "学 Python" --body "从 Android 转过来" -t python -t 学习 -t " python "
uv run notekit add "写个 Agent" -t agent
uv run notekit add "看电影"
uv run notekit list
uv run notekit done 3
uv run notekit list --undone
uv run notekit list --tag python
uv run notekit search android
uv run notekit delete 99; echo "退出码: $?"
```

真实输出：

```
已添加 #1 学 Python
已添加 #2 写个 Agent
已添加 #3 看电影
┏━━━━┳━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━━━━┓
┃ ID ┃ 状态 ┃ 标题       ┃ 标签        ┃
┡━━━━╇━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━━━━┩
│ 1  │ …    │ 学 Python  │ python,学习 │
│ 2  │ …    │ 写个 Agent │ agent       │
│ 3  │ …    │ 看电影     │             │
└────┴──────┴────────────┴─────────────┘
已完成 #3 看电影
┏━━━━┳━━━━━━┳━━━━━━━━━━━━┳━━━━━━━━━━━━━┓
┃ ID ┃ 状态 ┃ 标题       ┃ 标签        ┃
┡━━━━╇━━━━━━╇━━━━━━━━━━━━╇━━━━━━━━━━━━━┩
│ 1  │ …    │ 学 Python  │ python,学习 │
│ 2  │ …    │ 写个 Agent │ agent       │
└────┴──────┴────────────┴─────────────┘
┏━━━━┳━━━━━━┳━━━━━━━━━━━┳━━━━━━━━━━━━━┓
┃ ID ┃ 状态 ┃ 标题      ┃ 标签        ┃
┡━━━━╇━━━━━━╇━━━━━━━━━━━╇━━━━━━━━━━━━━┩
│ 1  │ …    │ 学 Python │ python,学习 │
└────┴──────┴───────────┴─────────────┘
命中 1 条
  #1 学 Python — 从 Android 转过来
找不到 #99
退出码: 1
```

**四处要对上的地方**：
1. **`-t python -t 学习 -t " python "` 三个标签只留下两个**（`python,学习`）——`field_validator` 的去空格+去重生效了。
2. **`done 3` 之后 `list --undone` 只剩 2 条**。
3. **`search android` 命中标题里没有 android 的那条**——因为它搜正文，而且大小写不敏感（正文是"从 **A**ndroid 转过来"）。
4. **`delete 99` 退出码是 1**，不是 0。

再看落盘的 JSON（`ensure_ascii=False` 的效果——中文是中文，不是 `\uXXXX`）：

```bash
cat /tmp/ck_cli.json
```

```json
[
  {
    "id": 1,
    "title": "学 Python",
    "body": "从 Android 转过来",
    "tags": [
      "python",
      "学习"
    ],
    "done": false,
    "created_at": "2026-08-16T10:18:01.144424Z"
  },
  {
    "id": 2,
    "title": "写个 Agent",
    "body": "",
    "tags": [
      "agent"
    ],
    "done": false,
    "created_at": "2026-08-16T10:18:01.270025Z"
  },
  {
    "id": 3,
    "title": "看电影",
    "body": "",
    "tags": [],
    "done": true,
    "created_at": "2026-08-16T10:18:01.392051Z"
  }
]
```

**校验也生效了。** 试着加一条空标题的：

```bash
uv run notekit add ""
```

Typer/rich 会打印一大段带框的 traceback，最后是关键的几行：

```
ValidationError: 1 validation error for NoteCreate
title
  String should have at least 1 character [type=string_too_short,
input_value='', input_type=str]
    For further information visit
https://errors.pydantic.dev/2.13/v/string_too_short
```

**这个未捕获异常暴露了一个真实的产品缺陷**：给用户看 traceback 是很差的体验。生产级的做法是在 `add` 里捕获它：

```python
from pydantic import ValidationError

try:
    note = _store().add(NoteCreate(title=title, body=body, tags=tag))
except ValidationError as e:
    console.print(f"[red]参数不对：[/red]{e.errors()[0]['msg']}")
    raise typer.Exit(code=2) from e
```

**`e.errors()` 返回一个字典列表**，每项有 `type`/`loc`/`msg`/`input`——就是 Checkpoint 9 里那些 `string_too_short` 的来源。这也是下一节 HTTP 422 响应体的内容。（我在示例项目里故意没加这个 try，好让你亲眼看见"不处理会怎样"。）

### 10.9 第五层：`web.py`（同一套逻辑，换成 HTTP）

`src/notekit/web.py`：

```python
"""Web 层：用 FastAPI 把同一套业务逻辑再暴露成 HTTP 接口。"""

import asyncio
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Query

from . import config
from .models import Note, NoteCreate, NoteUpdate
from .store import NoteStore

api = FastAPI(title="notekit API", version="0.1.0")


def get_store() -> NoteStore:
    """依赖注入：FastAPI 会在每次请求时调用它，把返回值塞进接口参数。
    测试时可以整体替换掉它 —— 见 tests/test_api.py。
    """
    return NoteStore(config.data_file())


# Annotated[类型, Depends(...)] 是 FastAPI 现在推荐的写法：
# 比 `store: NoteStore = Depends(get_store)` 更好，因为默认值位置留给了真正的默认值。
StoreDep = Annotated[NoteStore, Depends(get_store)]


@api.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}


@api.get("/slow")
async def slow(seconds: Annotated[float, Query(le=2.0)] = 0.3) -> dict[str, float]:
    """故意慢的接口，用来模拟"调一次大模型要等几百毫秒"。"""
    await asyncio.sleep(seconds)
    return {"waited": seconds}


@api.get("/notes", response_model=list[Note])
def list_notes(
    store: StoreDep,
    tag: Annotated[str | None, Query(description="按标签过滤")] = None,
    done: Annotated[bool | None, Query(description="按完成状态过滤")] = None,
) -> list[Note]:
    return store.list_all(tag=tag, done=done)


@api.post("/notes", response_model=Note, status_code=201)
def create_note(data: NoteCreate, store: StoreDep) -> Note:
    return store.add(data)


@api.get("/notes/{note_id}", response_model=Note)
def get_note(note_id: int, store: StoreDep) -> Note:
    note = store.get(note_id)
    if note is None:
        raise HTTPException(status_code=404, detail=f"没有 id={note_id} 的笔记")
    return note


@api.patch("/notes/{note_id}", response_model=Note)
def update_note(note_id: int, data: NoteUpdate, store: StoreDep) -> Note:
    note = store.update(note_id, data)
    if note is None:
        raise HTTPException(status_code=404, detail=f"没有 id={note_id} 的笔记")
    return note


@api.delete("/notes/{note_id}", status_code=204)
def delete_note(note_id: int, store: StoreDep) -> None:
    if not store.delete(note_id):
        raise HTTPException(status_code=404, detail=f"没有 id={note_id} 的笔记")
```

**只有 70 行，但你已经拿到了一套完整的 REST API + 自动生成的交互式文档 + 自动的参数校验。** 这是 FastAPI 最值钱的地方：**你的类型注解同时是校验规则和 API 文档**。

对照 Retrofit 理解：Retrofit 里你写接口声明、它生成客户端；FastAPI 里你写函数、它生成服务端 + OpenAPI 文档（相当于自动生成的 Retrofit 接口定义）。

**FastAPI 怎么知道每个参数从哪来？靠类型：**

| 参数写法 | FastAPI 从哪取值 |
|---|---|
| `note_id: int`（且路径里有 `{note_id}`） | **路径参数** |
| `data: NoteCreate`（是个 BaseModel） | **请求体 JSON** |
| `tag: Annotated[str \| None, Query(...)]` | **查询字符串** `?tag=x` |
| `store: StoreDep`（带 `Depends`） | **依赖注入**，调 `get_store()` |

**四个关键设计**：

1. **`Annotated[NoteStore, Depends(get_store)]` 起个别名 `StoreDep`。** 这样每个接口只写 `store: StoreDep` 一次。**依赖注入的价值在测试**：测试时把 `get_store` 换成"返回临时目录里的 store"，业务代码一行都不用改（下一节就用到）。这就是 Android 里的 Hilt/Dagger，但不需要注解处理器和代码生成。

2. **`response_model=Note`** 声明返回什么。FastAPI 会**按这个模型过滤返回值**——即使你的函数多返回了字段，也不会泄露给客户端。**这是一层安全网**（比如你的内部模型多了个 `password_hash` 字段）。

3. **`status_code=201` / `204`**。REST 约定：201 = 创建成功、204 = 删除成功且无返回体。不写默认是 200。

4. **`raise HTTPException(404, detail=...)` 而不是 `return {"error": ...}`。** 抛异常能从任意深度的调用栈里直接跳出并返回正确状态码，不用层层 `if` 往上传。

**`Query(le=2.0)` 是校验器**：`le` = less or equal。传 `?seconds=99` 会被自动拒绝，返回 422——**你没写一行校验代码**。

启动它：

```bash
export NOTEKIT_DATA=/tmp/ck_http.json
uv run notekit serve --port 8124
```

真实输出：

```
http://127.0.0.1:8124/docs 可以看交互式文档
INFO:     Started server process [27506]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8124 (Press CTRL+C to quit)
```

**浏览器打开 `http://127.0.0.1:8124/docs`**，你会看到一个可以直接点"Try it out"发请求的页面。这是 FastAPI 从你的类型注解自动生成的（Swagger UI），一行配置都没写。

> **安全提醒**：`serve` 的默认 `host` 是 `127.0.0.1`，**只有本机能访问**。这是故意的。如果你改成 `0.0.0.0`，同一个局域网里的任何人都能读写你的笔记——**而这套 API 完全没有任何认证**。真要对外提供服务，必须先加上认证（FastAPI 的 `OAuth2PasswordBearer` 或最简单的 API Key 依赖），并放在 HTTPS 后面。第 12 节在容器里用 `0.0.0.0` 是因为容器网络是隔离的，且只通过 compose 映射到宿主机的 `127.0.0.1`。

---
### 10.10 ✅ Checkpoint 12：用 curl 走完整个 API

**另开一个终端**（第一个还在跑服务），逐条发请求：

```bash
B=http://127.0.0.1:8124

curl -s $B/health; echo
curl -s -X POST $B/notes -H 'Content-Type: application/json' \
  -d '{"title":"用 curl 建的","body":"HTTP 也能操作同一份数据","tags":["api"," api ","web"]}' \
  -w ' <- HTTP %{http_code}\n'
curl -s -X POST $B/notes -H 'Content-Type: application/json' -d '{"title":"第二条"}' -w ' <- HTTP %{http_code}\n'
curl -s $B/notes; echo
curl -s -X PATCH $B/notes/1 -H 'Content-Type: application/json' -d '{"done":true}' -w ' <- HTTP %{http_code}\n'
curl -s "$B/notes?tag=web&done=true"; echo
curl -s $B/notes/99999 -w ' <- HTTP %{http_code}\n'
curl -s -X POST $B/notes -H 'Content-Type: application/json' -d '{"title":""}' -w ' <- HTTP %{http_code}\n'
curl -s "$B/slow?seconds=99" -w ' <- HTTP %{http_code}\n'
curl -s -X DELETE $B/notes/2 -w 'HTTP %{http_code}\n'
curl -s -X DELETE $B/notes/2 -w ' <- HTTP %{http_code}\n'
curl -s $B/openapi.json | python3 -c "import json,sys; print(sorted(json.load(sys.stdin)['paths'].keys()))"
```

真实输出：

```
{"status":"ok"}
{"id":1,"title":"用 curl 建的","body":"HTTP 也能操作同一份数据","tags":["api","web"],"done":false,"created_at":"2026-08-16T10:20:11.109647Z"} <- HTTP 201
{"id":2,"title":"第二条","body":"","tags":[],"done":false,"created_at":"2026-08-16T10:20:11.117481Z"} <- HTTP 201
[{"id":1,"title":"用 curl 建的","body":"HTTP 也能操作同一份数据","tags":["api","web"],"done":false,"created_at":"2026-08-16T10:20:11.109647Z"},{"id":2,"title":"第二条","body":"","tags":[],"done":false,"created_at":"2026-08-16T10:20:11.117481Z"}]
{"id":1,"title":"用 curl 建的","body":"HTTP 也能操作同一份数据","tags":["api","web"],"done":true,"created_at":"2026-08-16T10:20:11.109647Z"} <- HTTP 200
[{"id":1,"title":"用 curl 建的","body":"HTTP 也能操作同一份数据","tags":["api","web"],"done":true,"created_at":"2026-08-16T10:20:11.109647Z"}]
{"detail":"没有 id=99999 的笔记"} <- HTTP 404
{"detail":[{"type":"string_too_short","loc":["body","title"],"msg":"String should have at least 1 character","input":"","ctx":{"min_length":1}}]} <- HTTP 422
{"detail":[{"type":"less_than_equal","loc":["query","seconds"],"msg":"Input should be less than or equal to 2","input":"99","ctx":{"le":2.0}}]} <- HTTP 422
HTTP 204
{"detail":"没有 id=2 的笔记"} <- HTTP 404
['/health', '/notes', '/notes/{note_id}', '/slow']
```

**八处必须对上的地方**：

1. **POST 返回 201**，不是 200。
2. **`["api"," api ","web"]` 变成了 `["api","web"]`**——`models.py` 里那个 `field_validator` 在 HTTP 层同样生效。**这就是"业务逻辑只写一遍"的兑现**：CLI 和 HTTP 走的是同一个模型。
3. **PATCH 只传 `done`，标题和正文都还在**——`exclude_unset=True` 生效。
4. **`?tag=web&done=true` 组合过滤生效**，`done=true` 这个字符串被自动转成了 Python 的 `True`。
5. **404 的 detail 是我写的中文消息**。
6. **422 的响应体是结构化的**，`loc: ["body","title"]` 精确指出"请求体里的 title 字段"。**注意这和 Checkpoint 9 里 pydantic 直接抛的 `string_too_short` 是同一个东西**——FastAPI 只是把它包装成了 HTTP 响应。客户端可以按 `type` 字段分支处理。
7. **`?seconds=99` 也是 422**，`loc: ["query","seconds"]`——`Query(le=2.0)` 这一个声明就换来了校验 + 报错 + 文档。
8. **DELETE 成功是 204 且没有响应体**，再删同一个变 404。

`/openapi.json` 里有全部 4 个路径的机器可读描述。**这个文件可以直接喂给代码生成器**，给 Android 端生成 Retrofit 接口——这就是 OpenAPI 的意义。

跑完记得回第一个终端按 `Ctrl+C` 停掉服务。

### 10.11 测试：`pytest`

Android 里你写 JUnit，Python 里写 pytest。**pytest 比 JUnit 简单得多：函数名以 `test_` 开头、用普通的 `assert`，没有注解、没有类、没有 `assertEquals`。**

`tests/test_store.py`：

```python
from pathlib import Path

import pytest
from pydantic import ValidationError

from notekit.models import NoteCreate, NoteUpdate
from notekit.store import NoteStore


@pytest.fixture
def store(tmp_path: Path) -> NoteStore:
    """fixture = 每个测试跑之前自动准备好的东西。
    tmp_path 是 pytest 内置的临时目录，测试结束自动清理——不会污染你的真实数据。
    """
    return NoteStore(tmp_path / "notes.json")


def test_add_assigns_incrementing_id(store: NoteStore):
    a = store.add(NoteCreate(title="第一条"))
    b = store.add(NoteCreate(title="第二条"))
    assert a.id == 1
    assert b.id == 2


def test_tags_are_stripped_and_deduped(store: NoteStore):
    note = store.add(NoteCreate(title="标签测试", tags=[" py ", "py", "", "agent"]))
    assert note.tags == ["py", "agent"]


def test_empty_title_is_rejected():
    with pytest.raises(ValidationError):
        NoteCreate(title="")


def test_data_survives_reload(tmp_path: Path):
    path = tmp_path / "notes.json"
    NoteStore(path).add(NoteCreate(title="持久化"))
    reopened = NoteStore(path)
    assert [n.title for n in reopened.list_all()] == ["持久化"]


def test_update_only_touches_given_fields(store: NoteStore):
    note = store.add(NoteCreate(title="原标题", body="原正文"))
    updated = store.update(note.id, NoteUpdate(done=True))
    assert updated is not None
    assert updated.done is True
    assert updated.title == "原标题"  # 没传的字段不能被清空
    assert updated.body == "原正文"


def test_update_missing_id_returns_none(store: NoteStore):
    assert store.update(999, NoteUpdate(done=True)) is None


def test_delete(store: NoteStore):
    note = store.add(NoteCreate(title="待删"))
    assert store.delete(note.id) is True
    assert store.delete(note.id) is False
    assert store.list_all() == []


def test_filter_by_tag_and_done(store: NoteStore):
    store.add(NoteCreate(title="A", tags=["x"]))
    b = store.add(NoteCreate(title="B", tags=["x", "y"]))
    store.add(NoteCreate(title="C", tags=["y"]))
    store.update(b.id, NoteUpdate(done=True))

    assert [n.title for n in store.list_all(tag="x")] == ["A", "B"]
    assert [n.title for n in store.list_all(done=False)] == ["A", "C"]
    assert [n.title for n in store.list_all(tag="x", done=True)] == ["B"]


@pytest.mark.parametrize(
    "keyword,expected",
    [("python", ["学 Python"]), ("PYTHON", ["学 Python"]), ("正文", ["带正文的"]), ("不存在", [])],
)
def test_search_is_case_insensitive(store: NoteStore, keyword, expected):
    store.add(NoteCreate(title="学 Python"))
    store.add(NoteCreate(title="带正文的", body="这里有正文两个字"))
    assert [n.title for n in store.search(keyword)] == expected
```

**pytest 的四个核心概念**：

| 概念 | 是什么 | Android 对应 |
|---|---|---|
| `test_xxx` 函数 | 一个测试用例 | `@Test fun` |
| `assert x == y` | 断言，失败时 pytest 自动打印两边的实际值 | `assertEquals` |
| `@pytest.fixture` | 提供"预备好的对象"，**参数名匹配即注入** | `@Before` + 依赖注入 |
| `@pytest.mark.parametrize` | 一个函数跑多组数据 | `@ParameterizedTest` |

**fixture 的注入靠参数名**：`def test_delete(store: NoteStore)` 里的参数叫 `store`，pytest 就去找名叫 `store` 的 fixture 调用它，把返回值传进来。**不需要任何注解或注册**——这是 pytest 最反直觉也最好用的设计。

**`tmp_path` 是白送的**：pytest 内置 fixture，给每个测试一个独立的空临时目录，跑完自动清理。注意 `test_data_survives_reload` 直接要了 `tmp_path` 而没要 `store`——**fixture 可以按需组合**，谁要什么写什么。**测试里读写文件绝不要用固定路径**，会互相污染。

**`assert` 的魔法**：pytest 重写了 assert 语句，失败时会打印 `assert 1 == 2` 两边的真实值，不需要你写消息。这比 JUnit 的 `assertEquals(expected, actual)` 参数顺序困扰要省心。

**`with pytest.raises(ValidationError):`** 断言"这段代码必须抛这个异常"，相当于 JUnit 的 `assertThrows`。**如果没抛，测试失败**——这是测校验逻辑的正确姿势。

**注意测试函数没写返回类型 `-> None`**，而源码里每个函数都写了。这不是疏忽：`[tool.mypy]` 只在跑 `mypy src` 时检查 `src` 目录，测试目录不在检查范围内。**这是一个常见的实用取舍**——测试代码的类型严格度可以低一些，换取写起来快。

`tests/test_api.py`——**这里体现依赖注入的价值**：

```python
from pathlib import Path

import pytest
from fastapi.testclient import TestClient

from notekit.store import NoteStore
from notekit.web import api, get_store


@pytest.fixture
def client(tmp_path: Path) -> TestClient:
    """把 get_store 换成指向临时目录的版本 —— 测试不碰你的真实 notes.json。"""
    store = NoteStore(tmp_path / "notes.json")
    api.dependency_overrides[get_store] = lambda: store
    yield TestClient(api)
    api.dependency_overrides.clear()


def test_health(client: TestClient):
    r = client.get("/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}


def test_create_and_list(client: TestClient):
    r = client.post("/notes", json={"title": "第一条", "tags": ["py"]})
    assert r.status_code == 201
    body = r.json()
    assert body["id"] == 1
    assert body["done"] is False

    r = client.get("/notes")
    assert [n["title"] for n in r.json()] == ["第一条"]


def test_validation_error_returns_422(client: TestClient):
    r = client.post("/notes", json={"title": ""})
    assert r.status_code == 422
    assert r.json()["detail"][0]["loc"] == ["body", "title"]


def test_missing_field_returns_422(client: TestClient):
    r = client.post("/notes", json={"body": "没有标题"})
    assert r.status_code == 422


def test_get_404(client: TestClient):
    r = client.get("/notes/999")
    assert r.status_code == 404
    assert "没有 id=999" in r.json()["detail"]


def test_patch_partial(client: TestClient):
    client.post("/notes", json={"title": "原标题", "body": "原正文"})
    r = client.patch("/notes/1", json={"done": True})
    assert r.status_code == 200
    assert r.json()["done"] is True
    assert r.json()["title"] == "原标题"


def test_delete_then_404(client: TestClient):
    client.post("/notes", json={"title": "待删"})
    assert client.delete("/notes/1").status_code == 204
    assert client.delete("/notes/1").status_code == 404


def test_filter_query_params(client: TestClient):
    client.post("/notes", json={"title": "A", "tags": ["x"]})
    client.post("/notes", json={"title": "B", "tags": ["y"]})
    client.patch("/notes/2", json={"done": True})

    assert [n["title"] for n in client.get("/notes?tag=x").json()] == ["A"]
    assert [n["title"] for n in client.get("/notes?done=true").json()] == ["B"]
```

**四个要点**：

1. **`TestClient(api)` 不启动真实服务器、不开端口、不走网络**，直接在进程内调用你的接口函数。所以 8 个 HTTP 测试总共只花几十毫秒。对照 Android：这相当于用 MockWebServer，但连 mock server 都不用起。
2. **`api.dependency_overrides[get_store] = lambda: store`** 就是"把依赖换掉"。业务代码一个字都不改，测试就用上了临时文件——**这正是 10.9 节说的依赖注入的兑现**。如果没有 `Depends`，`web.py` 里会直接 `NoteStore(config.data_file())`，测试就只能靠改环境变量绕，脆弱得多。
3. **`yield` 前是准备、`yield` 后是清理**（相当于 JUnit 的 `@After`）。**必须清理 `dependency_overrides`**：它是挂在全局 `api` 对象上的字典，不清就会泄漏到后面的测试。这是"fixture 里改了全局状态就必须还原"的通例。
4. **`test_validation_error_returns_422` 断言的是 `loc == ["body","title"]`**——直接对上了 Checkpoint 12 里 curl 看到的响应体。**测试和手动验证看到的是同一个东西**，这才叫测试可信。

### 10.12 三道质量门

先看**完整的 `pyproject.toml`**——这就是最终成品，一个文件管住依赖、入口、打包、三个工具的全部配置：

```toml
[project]
name = "notekit"
version = "0.1.0"
description = "一个命令行 + Web 的笔记工具"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "httpx>=0.28.1",
    "pydantic>=2.13.4",
    "python-dotenv>=1.2.2",
    "typer>=0.27.1",
    "uvicorn[standard]>=0.30",
]

[project.scripts]
notekit = "notekit:main"

[build-system]
requires = ["uv_build>=0.11.32,<0.12.0"]
build-backend = "uv_build"

[dependency-groups]
dev = [
    "httpx>=0.28.1",
    "mypy>=2.3.1",
    "pytest>=9.1.1",
    "ruff>=0.16.3",
]

[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.ruff.lint.per-file-ignores]
# Typer 的设计就是把 typer.Option(...) 放在默认参数里，B008 在这里是误报
"src/notekit/cli.py" = ["B008"]

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.mypy]
python_version = "3.12"
warn_return_any = true
```

**对照 Android 的感受**：Gradle 里这些配置要散在 `build.gradle.kts`、`gradle.properties`、`detekt.yml`、`.editorconfig` 好几个文件里。**Python 现在的方向是全部收进 `pyproject.toml`**（`[tool.xxx]` 是各工具约定的命名空间）。少数工具还有自己的配置文件，但主流工具都支持这里。

**`select` 里每个字母是一组规则**：

| 代号 | 管什么 | 例子 |
|---|---|---|
| `E` | 代码风格（pycodestyle） | 行太长、空格不对 |
| `F` | 真实错误（pyflakes） | 用了没定义的变量、导入了没用的包 |
| `I` | import 排序（isort） | 标准库/第三方/本地分组并排序 |
| `UP` | 用新语法（pyupgrade） | `List[str]` → `list[str]` |
| `B` | 常见 bug（bugbear） | **可变默认参数**、循环里的闭包陷阱 |
| `SIM` | 能写更简单（simplify） | `if x: return True else: return False` |

**`F` 和 `B` 是最值钱的两组**——它们抓的是真 bug，不是风格。`I` 让你再也不用手动整理 import。ruff 还有几十组规则可选（`ANN` 强制类型注解、`S` 安全检查 bandit、`PT` pytest 风格……），**建议从这 6 组起步，跑顺了再逐个加**——一次全开会得到几百个报错然后放弃。

**`warn_return_any = true` 是 mypy 最该开的一项**：函数声明返回 `int` 却实际返回了 `Any`（比如从 `json.loads()` 出来的东西）时报警。**`Any` 会像病毒一样传播**，它经过的地方类型检查全部失效，这个开关能挡住它。

三条命令：

```bash
uv run ruff check .      # 语法/风格/常见 bug —— ktlint + detekt
uv run mypy src          # 类型检查 —— Kotlin 编译器帮你做的那部分
uv run pytest -q         # 测试 —— JUnit
```

顺手还能自动修：

```bash
uv run ruff check --fix .    # 自动修能修的（比如 import 排序）
uv run ruff format .         # 格式化 —— 相当于 ktlintFormat
```

### 10.13 ✅ Checkpoint 13：三道门全绿

```bash
uv run ruff check .
uv run mypy src
uv run pytest -q
```

真实输出：

```
All checks passed!
Success: no issues found in 6 source files
.................... [100%]
20 passed, 1 warning in 0.22s
```

> **只有最后那个耗时数字会和你不一样**（我在同一台机器上反复跑，0.18s / 0.22s / 0.23s 都出现过）。**要一字不差对上的是前三行和 `20 passed, 1 warning`**——`20` 和 `1` 这两个数字必须相同，否则说明你少写了测试或多出了警告。

**这三行是你以后每次提交前都要看到的东西。** 建议直接串起来当一条命令：

```bash
uv run ruff check . && uv run mypy src && uv run pytest -q
```

`&&` 保证前一步失败就不继续——这就是 10.7 节说的"退出码要正确"在真实工作流里的用途。

想看每个测试跑了什么，加 `-v`（真实输出）：

```
cachedir: .pytest_cache
rootdir: /private/tmp/verify/py/notekit
configfile: pyproject.toml
testpaths: tests
plugins: anyio-4.14.2
collecting ... collected 20 items

tests/test_api.py::test_health PASSED                                    [  5%]
tests/test_api.py::test_create_and_list PASSED                           [ 10%]
tests/test_api.py::test_validation_error_returns_422 PASSED              [ 15%]
tests/test_api.py::test_missing_field_returns_422 PASSED                 [ 20%]
tests/test_api.py::test_get_404 PASSED                                   [ 25%]
tests/test_api.py::test_patch_partial PASSED                             [ 30%]
tests/test_api.py::test_delete_then_404 PASSED                           [ 35%]
tests/test_api.py::test_filter_query_params PASSED                       [ 40%]
tests/test_store.py::test_add_assigns_incrementing_id PASSED             [ 45%]
tests/test_store.py::test_tags_are_stripped_and_deduped PASSED           [ 50%]
tests/test_store.py::test_empty_title_is_rejected PASSED                 [ 55%]
tests/test_store.py::test_data_survives_reload PASSED                    [ 60%]
tests/test_store.py::test_update_only_touches_given_fields PASSED        [ 65%]
tests/test_store.py::test_update_missing_id_returns_none PASSED          [ 70%]
tests/test_store.py::test_delete PASSED                                  [ 75%]
tests/test_store.py::test_filter_by_tag_and_done PASSED                  [ 80%]
tests/test_store.py::test_search_is_case_insensitive[python-expected0] PASSED [ 85%]
tests/test_store.py::test_search_is_case_insensitive[PYTHON-expected1] PASSED [ 90%]
tests/test_store.py::test_search_is_case_insensitive[正文-expected2] PASSED [ 95%]
tests/test_store.py::test_search_is_case_insensitive[不存在-expected3] PASSED [100%]
```

**三个值得注意的细节**：

1. **`test_search_is_case_insensitive[python-expected0]` 这种带方括号的名字**：`parametrize` 的每组数据都是一个独立测试，方括号里是参数值。**16 个测试函数变成了 20 个用例**，这就是为什么数字是 20。你可以只跑其中一个：
   ```bash
   uv run pytest "tests/test_store.py::test_search_is_case_insensitive[PYTHON-expected1]"
   ```
2. **中文参数在测试 ID 里显示成 `正文`**——pytest 对非 ASCII 参数会转义。**想让 ID 好看，给 parametrize 加 `ids=` 参数**：`@pytest.mark.parametrize(..., ids=["小写","大写","搜正文","搜不到"])`。
3. **`expected0/1/2/3` 是自动编号**：因为第二个参数是列表（不是简单标量），pytest 没法用它生成可读名字，就用序号代替。

**那 1 个 warning 是什么？** 别忽略警告，看一眼完整内容：

```
=============================== warnings summary ===============================
.venv/lib/python3.12/site-packages/fastapi/testclient.py:1
  .../fastapi/testclient.py:1: StarletteDeprecationWarning: Using `httpx` with
  `starlette.testclient` is deprecated; install `httpx2` instead.
    from starlette.testclient import TestClient as TestClient  # noqa
```

这是"依赖的新版本改了推荐用法"的提示，**不影响正确性，而且发生在 fastapi 内部（不是我的代码）**。处理原则：警告要读、要记下来，但不必立刻改——除非它说的是"下个大版本会删掉"。真要消掉，按提示装 `httpx2`。

**如果想让警告变成硬性失败**（成熟项目常这么做），在 `pyproject.toml` 加：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
filterwarnings = ["error"]     # 任何警告都当错误
```

**但对这个项目会立刻挂**——因为警告来自第三方库，你改不了。所以更实用的是精确忽略：`filterwarnings = ["error", "ignore::DeprecationWarning:starlette.*"]`。**这也是一条通用原则：全局收紧 + 精确放行，而不是全局放松。**

---
## 十一、发布：把项目变成"别人能装的东西"

Android 里你把代码打成 `.apk` 传应用商店，或打成 `.aar` 传 Maven。Python 里你把代码打成 **`.whl`（wheel）** 传 **PyPI**。这一节走完从"我本地能跑"到"全世界 `pip install` 就能用"的整条路。

| Android | Python |
|---|---|
| `.aar` / `.jar` | **`.whl`（wheel）** —— 装好即用的二进制包 |
| 源码 zip | **`.tar.gz`（sdist）** —— 源码包，装的时候要现场构建 |
| Maven Central / jitpack | **PyPI**（pypi.org） |
| `./gradlew assembleRelease` | **`uv build`** |
| `./gradlew publish` | **`uv publish`** / `twine upload` |
| `gradle/libs.versions.toml` + `gradle.lockfile` | `pyproject.toml` + **`uv.lock`** |

### 11.1 `uv build`：打包

```bash
uv build
```

真实输出：

```
Building source distribution (uv build backend)...
Building wheel from source distribution (uv build backend)...
Successfully built dist/notekit-0.1.0.tar.gz
Successfully built dist/notekit-0.1.0-py3-none-any.whl
```

```bash
ls -l dist
```

```
-rw-r--r--  1 c  wheel  7085 Aug 16 18:23 notekit-0.1.0-py3-none-any.whl
-rw-r--r--  1 c  wheel  4897 Aug 16 18:23 notekit-0.1.0.tar.gz
```

**两个产物，两种用途**：

- **wheel（`.whl`）**：装的时候直接解压就完事，快。文件名 `notekit-0.1.0-py3-none-any.whl` 是 `包名-版本-Python标签-ABI标签-平台标签`：
  - `py3` = 任何 Python 3 都能用
  - `none` = 不依赖特定的 C ABI
  - `any` = 任何操作系统/CPU 架构
  - **`py3-none-any` 叫"纯 Python 包"**，一个文件通吃全平台。对比 numpy 的 wheel 会长成 `numpy-2.5.2-cp312-cp312-macosx_11_0_arm64.whl`——里面有编译好的 C 代码，所以要区分 Python 版本、操作系统和 CPU。
- **sdist（`.tar.gz`）**：源码包。当某个平台没有现成 wheel 时，pip 会下 sdist 然后现场构建。**这就是为什么有时 `pip install` 会突然开始编译半小时**。

### 11.2 拆开 wheel 看里面到底装了什么

**wheel 本质就是一个 zip**，可以直接看：

```bash
unzip -l dist/notekit-0.1.0-py3-none-any.whl
```

真实输出：

```
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  01-01-1980 00:00   notekit/
      161  01-01-1980 00:00   notekit/__init__.py
     2907  01-01-1980 00:00   notekit/cli.py
      510  01-01-1980 00:00   notekit/config.py
     1501  01-01-1980 00:00   notekit/models.py
     2473  01-01-1980 00:00   notekit/store.py
     2532  01-01-1980 00:00   notekit/web.py
        0  01-01-1980 00:00   notekit-0.1.0.dist-info/
       81  01-01-1980 00:00   notekit-0.1.0.dist-info/WHEEL
       42  01-01-1980 00:00   notekit-0.1.0.dist-info/entry_points.txt
      358  01-01-1980 00:00   notekit-0.1.0.dist-info/METADATA
      737  01-01-1980 00:00   notekit-0.1.0.dist-info/RECORD
---------                     -------
    11302                     12 files
```

**四个要注意的点**：

1. **`tests/` 没有进去，`.env` 也没有。** 这是 src-layout + 默认规则的好处：只有 `src/notekit/` 下的东西被打包。**测试代码不该发给用户**，密钥更不该。
2. **`src/` 这一层在包里消失了**——装到用户机器上就是 `site-packages/notekit/`。`src/` 只是开发时的目录约定。
3. **所有文件的时间戳都是 `01-01-1980 00:00`**。这不是 bug，是**可复现构建**（reproducible build）：同样的源码，任何人任何时候打出的 wheel 字节完全一致，能算出同样的哈希。zip 格式最早支持的时间就是 1980 年，所以取它当固定值。
4. **`dist-info/` 里的四个文件是包的"身份证"**：

`entry_points.txt`——**这是命令行命令的来源**：

```
[console_scripts]
notekit = notekit:main
```

`METADATA`——**依赖声明就在这里，pip 靠它决定还要装什么**：

```
Metadata-Version: 2.3
Name: notekit
Version: 0.1.0
Summary: 一个命令行 + Web 的笔记工具
Requires-Dist: fastapi>=0.115
Requires-Dist: httpx>=0.28.1
Requires-Dist: pydantic>=2.13.4
Requires-Dist: python-dotenv>=1.2.2
Requires-Dist: typer>=0.27.1
Requires-Dist: uvicorn[standard]>=0.30
Requires-Python: >=3.12
Description-Content-Type: text/markdown
```

**注意 `Requires-Dist` 里是 `>=` 范围，不是精确版本。** 这是一条重要区别：

> **库（library）声明宽松范围，应用（application）用锁文件钉死。**
>
> 如果你的 wheel 里写死 `pydantic==2.13.4`，任何同时用到别的 pydantic 版本的项目都装不上你——**版本冲突地狱**。所以 `pyproject.toml` 的 `dependencies` 写 `>=`。而 `uv.lock` 记录精确版本，只用于**你自己开发和部署**，不进 wheel。

`WHEEL`——描述打包格式：

```
Wheel-Version: 1.0
Generator: uv 0.11.32
Root-Is-Purelib: true
Tag: py3-none-any
```

`RECORD` 是文件清单 + 每个文件的哈希，卸载时靠它知道该删哪些文件。

再看 sdist 里有什么（**多了 `pyproject.toml` 和 `README.md`**，因为要能重新构建）：

```bash
tar -tzf dist/notekit-0.1.0.tar.gz
```

```
notekit-0.1.0/PKG-INFO
notekit-0.1.0/
notekit-0.1.0/README.md
notekit-0.1.0/pyproject.toml
notekit-0.1.0/src
notekit-0.1.0/src/notekit/__init__.py
notekit-0.1.0/src/notekit/cli.py
notekit-0.1.0/src/notekit/config.py
notekit-0.1.0/src/notekit/models.py
notekit-0.1.0/src/notekit/store.py
notekit-0.1.0/src/notekit/web.py
```

> **想控制打包内容**，在 `pyproject.toml` 里配 `[tool.uv.build-backend]`（不同后端配置项不同，比如 hatchling 用 `[tool.hatch.build]`）。**打包完一定要 `unzip -l` 看一眼**——"忘了把数据文件打进去"是发布环节最常见的 bug，而且只有用户会撞到。

### 11.3 ✅ Checkpoint 14：在一个全新环境里装上自己的包

这一步是**真正的验证**：新建一个干净的虚拟环境，只用那个 wheel 文件安装，看它能不能跑。

```bash
cd /tmp/verify/py
uv venv fresh --python 3.12
uv pip install --python fresh/bin/python notekit/dist/notekit-0.1.0-py3-none-any.whl
```

真实输出（截取头尾）：

```
Using CPython 3.12.13 interpreter at: /opt/homebrew/opt/python@3.12/bin/python3.12
Creating virtual environment at: fresh
Activate with: source fresh/bin/activate

Using Python 3.12.13 environment at: fresh
Resolved 29 packages in 1.55s
Prepared 1 package in 45ms
Installed 29 packages in 48ms
 + annotated-doc==0.0.5
 + annotated-types==0.8.0
 + anyio==4.14.2
 ...
 + notekit==0.1.0 (from file:///private/tmp/verify/py/notekit/dist/notekit-0.1.0-py3-none-any.whl)
 ...
 + uvloop==0.22.1
 + watchfiles==1.2.0
 + websockets==17.0.1
```

**我只装了 1 个 wheel，pip 却装了 29 个包**——它读了 `METADATA` 里的 `Requires-Dist`，递归把依赖全解决了。**注意 29 < 40**：因为 dev 依赖（pytest/ruff/mypy）没有进 wheel，`--dev` 起作用了。

命令行命令也自动出现了：

```bash
ls fresh/bin/ | grep -i note
```

```
notekit
```

**这就是 `entry_points.txt` 的效果**：安装器读到 `[console_scripts] notekit = notekit:main`，就在 `bin/` 下生成一个可执行脚本。真的跑一下：

```bash
NOTEKIT_DATA=/tmp/fresh.json ./fresh/bin/notekit add "从 wheel 装的" -t 发布
NOTEKIT_DATA=/tmp/fresh.json ./fresh/bin/notekit list
```

```
已添加 #1 从 wheel 装的
┏━━━━┳━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━┓
┃ ID ┃ 状态 ┃ 标题          ┃ 标签 ┃
┡━━━━╇━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━┩
│ 1  │ …    │ 从 wheel 装的 │ 发布 │
└────┴──────┴───────────────┴──────┘
```

再确认它加载的确实是**装进 site-packages 的那份**，不是当前目录的源码：

```bash
cd /tmp && /tmp/verify/py/fresh/bin/python -c "import notekit; print(notekit.__version__); print(notekit.__file__)"
```

```
0.1.0
/private/tmp/verify/py/fresh/lib/python3.12/site-packages/notekit/__init__.py
```

**路径里有 `site-packages` 就说明成功了。** 我特意先 `cd /tmp` 再跑——如果在项目目录里跑，`__file__` 可能指向 `src/notekit/`（5.1 节说过 `sys.path` 里有当前目录），那就验证不了打包是否正确。**这是"src-layout 让你提前发现打包错误"的兑现方式。**

> **一个坑，我真撞了**：我一开始想用 `./fresh/bin/python -m pip install ...`，结果报
> ```
> /private/tmp/verify/py/fresh/bin/python: No module named pip
> ```
> **`uv venv` 建的虚拟环境默认不装 pip**（因为 uv 自己就管安装，省时间省空间）。解决办法二选一：用 `uv pip install --python fresh/bin/python ...`（推荐，快得多），或者建环境时加 `uv venv fresh --seed` 把 pip 装进去。

### 11.4 `uv tool install`：把 CLI 装成全局命令

上面那样装进虚拟环境，只有激活那个环境才能用。**如果你的包是个命令行工具，你想在任何地方都能敲 `notekit`**，用 `uv tool`：

```bash
uv tool install --force .
```

```
Installed 1 executable: notekit
```

```bash
which notekit
```

```
/Users/c/.local/bin/notekit
```

```bash
uv tool list
```

```
notekit v0.1.0
- notekit
```

**`uv tool` 的原理**：给每个工具建一个**独立隐藏的虚拟环境**，然后只把可执行文件软链到 `~/.local/bin`。**所以工具之间的依赖永远不会互相冲突**——`ruff` 要 click 8、`notekit` 要 click 9，各自装各自的，互不干扰。

这就是 1.x 节说过的 `uvx`/`uv tool run` 的持久化版本：

| 命令 | 什么时候用 |
|---|---|
| `uvx ruff check .` | **临时跑一次**，不留痕迹 |
| `uv tool install ruff` | **天天用**，装成全局命令 |
| `uv add --dev ruff` | **项目内用**，版本随项目锁定（推荐） |

卸载：

```bash
uv tool uninstall notekit
```

```
Uninstalled 1 executable: notekit
```

**对照 Android**：这相当于把一个工具装进 `~/.gradle` 全局可用，而不是每个项目 `build.gradle` 里声明一遍。**给团队用的内部 CLI 工具，`uv tool install` 是最省心的分发方式**——不用打包传 PyPI，直接 `uv tool install git+https://内部仓库地址` 就行。

### 11.5 发到 PyPI

**先做四件事，一件都别漏**：

1. **改包名**。PyPI 上包名全球唯一，`notekit` 大概率被占了。先去 pypi.org 搜一下。
2. **补全 `pyproject.toml` 的元信息**——没有这些，你的包在 PyPI 页面上会很难看，也搜不到：

```toml
[project]
name = "notekit-你的后缀"
version = "0.1.0"
description = "一个命令行 + Web 的笔记工具"
readme = "README.md"
requires-python = ">=3.12"
license = "MIT"
authors = [{ name = "你的名字", email = "you@example.com" }]
keywords = ["notes", "cli", "fastapi"]
classifiers = [
    "Programming Language :: Python :: 3.12",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
]

[project.urls]
Homepage = "https://github.com/你/notekit"
Issues = "https://github.com/你/notekit/issues"
```

3. **写 README.md**。它会直接显示在 PyPI 页面上（`readme = "README.md"` 那行的作用）。**我的示例项目 README 是空的——这是我的偷懒，真发布前必须写。**
4. **先发到 TestPyPI 试一次**。这是一个和 PyPI 完全隔离的沙箱，专门给你练手：

```bash
# 1) 去 test.pypi.org 注册，创建一个 API token（不是密码）
# 2) 发布
uv publish --publish-url https://test.pypi.org/legacy/ --token pypi-你的token
# 3) 从 TestPyPI 装回来验证
uv pip install --index-url https://test.pypi.org/simple/ \
  --extra-index-url https://pypi.org/simple/ notekit-你的后缀
```

**`--extra-index-url` 是必需的**：TestPyPI 上没有 fastapi、pydantic 这些真实依赖，得让它去真 PyPI 找。

确认没问题了再发正式版：

```bash
uv publish --token pypi-你的正式token
```

**发布前的安全和纪律检查（这几条会真出事）**：

- **PyPI 的版本号不可覆盖、不可重发。** 发错了只能 yank（标记为不推荐）然后发 `0.1.1`。**没有"删了重发"这个操作。** 所以 TestPyPI 那一步别跳。
- **用 API token，不要用密码**，而且 token 要**限定到单个项目**。token 就是密钥——**只放环境变量 `UV_PUBLISH_TOKEN`，绝不写进代码、CI 配置文件或 `pyproject.toml`**。CI 里用平台的 secrets 功能。
- **发布前 `unzip -l` 检查 wheel 里没有 `.env`、没有密钥、没有内部数据。** 一旦发上去，就算 yank 了，别人也已经下载/镜像/索引过了。**PyPI 上泄露密钥等于公开广播。**
- **CI 里发布推荐用 Trusted Publishing**（PyPI 的 OIDC 机制），这样连 token 都不用存——CI 直接用身份凭证换取一次性上传权限。
- **注意包名安全**：别装名字看着像知名包但拼写略有不同的东西（`requsets` vs `requests`）——这叫 typosquatting，是真实存在的供应链攻击。**你自己发布时也可以顺手把常见拼错的名字占下来防御。**

### 11.6 版本号怎么定：语义化版本

`0.1.0` 这三段叫 **SemVer（语义化版本）**，和 Android 一样的规矩：

| 段 | 什么时候加 | 例子 |
|---|---|---|
| **主版本** MAJOR | **破坏兼容**：删了函数、改了参数、改了返回格式 | `1.0.0` → `2.0.0` |
| **次版本** MINOR | **加功能且向后兼容** | `1.2.0` → `1.3.0` |
| **修订** PATCH | **只修 bug** | `1.2.3` → `1.2.4` |

**`0.x.y` 是特殊约定**："还没稳定，我随时可能破坏兼容"。所以别人依赖你的 0.x 版本时会写 `==0.1.*` 而不是 `>=0.1`。

**依赖声明的写法对应 SemVer**：

| 写法 | 含义 |
|---|---|
| `pydantic>=2` | 2 及以上任何版本（**赌 v2 系列不破坏兼容**） |
| `pydantic>=2.13,<3` | 推荐写法：允许小版本升级，挡住大版本 |
| `pydantic==2.13.4` | 钉死（**库里别这么写**，见 11.2） |
| `pydantic~=2.13.4` | 等价于 `>=2.13.4,<2.14`——只允许 patch 升级 |

**版本号写在哪？** 我的项目里写了两处：`pyproject.toml` 的 `version` 和 `__init__.py` 的 `__version__`。**两处不同步是经典 bug。** 更好的做法是让它只有一处：

```toml
[project]
dynamic = ["version"]      # 版本不写死在这

[tool.hatch.version]       # 用 hatchling 后端时
path = "src/notekit/__init__.py"
```

或者反过来，代码里从包元数据读：

```python
from importlib.metadata import version
__version__ = version("notekit")
```

### 11.7 `uv.lock` 和 `uv export`：让部署可复现

**`uv.lock` 记录了整棵依赖树的精确版本 + 哈希。** 它是 Gradle 的 `gradle.lockfile`，**必须提交进版本库**。

```bash
wc -c uv.lock
```

```
195509 uv.lock
```

**19 万字节记录 40 个包**——因为每个包都记了版本、来源 URL、所有平台的 wheel 哈希、以及它自己的依赖关系。

三条日常命令：

```bash
uv sync --frozen     # 严格按锁文件装，锁文件和 pyproject 不一致就报错（CI/部署用这个）
uv lock --check      # 只检查锁文件是不是最新的，不改任何东西
uv lock --upgrade    # 主动升级所有依赖并更新锁文件
```

`uv sync --frozen` 的真实输出：

```
Checked 39 packages in 8ms
```

`uv lock --check`：

```
Resolved 40 packages in 3ms
```

**`--frozen` 是部署的关键**：它保证"绝不悄悄改版本"。没有它，`uv sync` 可能因为 `pyproject.toml` 有变动而重新解析出新版本——**部署时最不想遇到的事就是"昨天好的今天挂了，因为某个依赖发了新版"**。这就是 12 节 Dockerfile 里用 `uv sync --frozen` 的原因。

**如果部署环境不装 uv 呢？** 导出成传统的 `requirements.txt`：

```bash
uv export --no-dev --no-emit-project -o requirements.txt
wc -l requirements.txt
```

```
     447 requirements.txt
```

**447 行记录 29 个运行时包**，因为每个包都带上了所有平台的哈希：

```
# This file was autogenerated by uv via the following command:
#    uv export --no-dev --no-emit-project -o requirements.txt
annotated-doc==0.0.5 \
    --hash=sha256:117bac03a25ede5df5440e855b32d556049ca169ead221505badf432fed4b101 \
    --hash=sha256:c7e58ce09192557605d8bbd92836d7e1d520ac9580096042c0bfd197efacf1bb
    # via
    #   fastapi
    #   typer
annotated-types==0.8.0 \
    --hash=sha256:13b2beaad985e05e2d6407ee4c4f35590b11f8d693a258a561055cac8f64cab7 \
    --hash=sha256:f072f4d804ea359e4eaf198b1af7a8b0943881a87f31bb764f8bf219bb9419e0
    # via pydantic
```

**`--hash=` 是供应链安全的关键**：pip 装的时候会校验下载文件的哈希，**不匹配就拒绝安装**。这能挡住"包被人在传输途中替换"或"PyPI 上的包被重新上传"。要强制校验，装的时候用 `pip install --require-hashes -r requirements.txt`。

**`# via` 注释告诉你这个包是谁要求的**——排查"这个包哪来的"时非常有用。

| 文件 | 提交进 git？ | 干什么用 |
|---|---|---|
| `pyproject.toml` | ✅ 必须 | 声明"我要什么"（范围） |
| `uv.lock` | ✅ 必须 | 记录"实际装了什么"（精确） |
| `requirements.txt` | ⚠️ 视情况 | 给不装 uv 的环境用，是**导出产物**，不要手改 |
| `.python-version` | ✅ 建议 | 记录项目用哪个 Python 版本 |
| `.venv/` | ❌ 绝不 | 本地环境，几百 MB，且不跨平台 |
| `.env` | ❌ 绝不 | **密钥** |
| `dist/` | ❌ 不 | 构建产物 |

---
## 十二、部署：让服务真的跑在别的机器上

Android 里"部署"就是传商店让用户下载。后端服务的部署是另一回事：**你要负责它一直活着**。这一节走三步——先把服务用生产方式跑起来，再装进 Docker 镜像，最后用 compose 管起来。

### 12.1 开发服务器 vs 生产服务器

前面一直用 `notekit serve`，它内部是 `uvicorn.run(...)`——**单进程**。生产环境要多进程：

```bash
uv run uvicorn notekit.web:api --host 127.0.0.1 --port 8125 --workers 4
```

真实输出：

```
INFO:     Uvicorn running on http://127.0.0.1:8125 (Press CTRL+C to quit)
INFO:     Started parent process [28336]
INFO:     Started server process [28339]
INFO:     Waiting for application startup.
INFO:     Started server process [28340]
INFO:     Started server process [28341]
INFO:     Started server process [28338]
INFO:     Waiting for application startup.
INFO:     Waiting for application startup.
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Application startup complete.
INFO:     Application startup complete.
INFO:     Application startup complete.
```

**一个父进程 + 4 个工作进程**，四个 `Application startup complete` 各一条。它们**共享同一个监听端口**，操作系统负责把连接分给谁。

**为什么要多进程？回到 8.3 节的 GIL。** 一个 Python 进程里的多个线程不能同时执行 Python 字节码，所以**单进程的 CPU 上限就是一个核**。多进程绕开 GIL，4 个 worker 能吃 4 个核。

**worker 数量怎么定？**

| 你的服务主要在干什么 | 推荐 worker 数 |
|---|---|
| **等外部 IO**（调大模型 API、查数据库）——**Agent 服务的典型情况** | `CPU 核数` 就够，甚至更少；瓶颈在网络不在 CPU |
| **CPU 密集**（本地推理、大量 JSON 解析/编码） | `CPU 核数`，最多 `核数 + 1` |
| 混合 | 从 `核数` 开始，压测后调整 |

**别照搬"`2 × 核数 + 1`"那个 Gunicorn 老公式**——那是给同步阻塞框架的。**FastAPI 的 `async def` 接口在单个 worker 内就能并发处理成百上千个等待中的请求**（6.6 节实测过：串行 2.47 秒的活并发只要 0.33 秒）。**先把 async 写对，再考虑加 worker。**

四个必须知道的部署细节：

1. **`--workers` 和 `--reload` 不能一起用**。`--reload` 是开发用的（改代码自动重启），生产环境绝不能开——它要监视文件系统，而且会让 worker 管理变得不可预测。
2. **多 worker 意味着进程间不共享内存**。`notekit` 的 `NoteStore` 是每个请求新建的（读文件），所以没事；但**如果你在模块级放了一个全局字典当缓存，4 个 worker 会有 4 份互不相同的缓存**。要共享状态就得用 Redis 或数据库。
3. **`notekit` 的 JSON 文件存储在多 worker 下会丢数据**——两个进程同时 `_save()` 会互相覆盖（读-改-写不是原子的）。**这是示例项目的已知局限，我没有修**。真实项目请用 SQLite（`sqlite3` 标准库，WAL 模式下支持并发读+单写）或 Postgres。**这是"教学代码"和"生产代码"最典型的一条分界线。**
4. **真实生产还要在 uvicorn 前面放一层 Nginx/Caddy**，负责 HTTPS、限流、静态文件和超时。uvicorn 直接对公网也能跑，但少了这些保护。

### 12.2 Docker：为什么要它

**Android 的类比不太现成，但可以这样理解**：Docker 镜像是一个"打包了操作系统 + Python + 你的依赖 + 你的代码"的完整快照。它解决的正是"在我机器上是好的"这个问题——因为镜像里连 Python 版本都是固定的。

三个词分清楚：

| 词 | 是什么 | 类比 |
|---|---|---|
| **Dockerfile** | 一份"怎么造镜像"的说明书 | `build.gradle` |
| **镜像 image** | 造出来的只读快照 | `.apk` 文件 |
| **容器 container** | 镜像跑起来的一个实例 | 手机上正在运行的 App |

一个镜像可以同时跑很多容器，就像一个 apk 可以装在很多台手机上。

### 12.3 多阶段 Dockerfile 逐行讲

`Dockerfile`：

```dockerfile
# ---------- 第一阶段 builder：装依赖，装完就丢 ----------
# 用官方带 uv 的镜像，省掉自己装 uv 的步骤
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app

# 关键技巧：先只拷依赖清单，再装依赖。
# 只要 pyproject.toml / uv.lock 没变，Docker 就复用这一层缓存，
# 改业务代码时不会重新装一遍包。
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

# 再拷源码，装项目本身
COPY src ./src
COPY README.md ./
RUN uv sync --frozen --no-dev

# ---------- 第二阶段 runtime：只带运行时需要的东西 ----------
# 用同系列的镜像做运行时，保证 .venv 里的二进制能对上
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

# 不用 root 跑服务 —— 容器安全第一条
RUN useradd --create-home --uid 10001 appuser

WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/.venv /app/.venv
COPY --from=builder --chown=appuser:appuser /app/src /app/src

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    NOTEKIT_DATA=/data/notes.json

RUN mkdir -p /data && chown appuser:appuser /data
VOLUME ["/data"]

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import httpx,sys; sys.exit(0 if httpx.get('http://127.0.0.1:8000/health').status_code==200 else 1)"

# host 必须是 0.0.0.0：容器内只听 127.0.0.1，外面永远连不上
CMD ["uvicorn", "notekit.web:api", "--host", "0.0.0.0", "--port", "8000"]
```

**十条要点，每条都是一个真实教训**：

**1. 为什么分两个阶段（multi-stage）？** 第一阶段可以随便装编译工具、留缓存、产生垃圾；第二阶段**只把 `.venv` 和 `src` 拷过来**，前面的中间产物全部丢弃，不进最终镜像。**镜像小 = 拉取快 = 部署快 = 攻击面小。**

**2. `COPY pyproject.toml uv.lock ./` 必须在 `COPY src` 之前。** 这是整个 Dockerfile 最重要的一行顺序。Docker 的每条指令是一"层"，**只要某层的输入没变就复用缓存，一层变了后面全部重来**。依赖清单变得少、源码变得多，所以先拷清单装依赖、后拷源码。**顺序写反的话，你每改一个字符都要重装 40 个包。** 下一节实测。

**3. `--no-install-project`**：第一次 sync 时只装依赖、**不装你自己的项目**（因为源码还没拷进来）。少了这个参数会报错。

**4. `--no-dev`**：不装 pytest/ruff/mypy。**生产镜像里不该有测试工具**——多余体积 + 多余攻击面。

**5. `UV_COMPILE_BYTECODE=1`**：装包时就把 `.py` 编译成 `.pyc`。**代价是构建慢一点，收益是容器启动快**（不用第一次运行时才编译）。容器可能频繁重启，这个交换很值。

**6. `UV_LINK_MODE=copy`**：uv 默认用硬链接省空间，但**跨 Docker 层的硬链接会出问题**，改成复制。不加会看到一堆警告。

**7. `useradd --uid 10001 appuser` + `USER appuser`：这是容器安全第一条。** 默认容器里的进程是 root，一旦被攻破，攻击者在容器里就是 root，更容易逃逸到宿主机。**指定固定 uid（10001）** 而不是让系统随便分，是为了让挂载卷的文件权限可预测。

**8. `PYTHONUNBUFFERED=1`：不加这个，你的日志会看不见。** Python 的 stdout 在管道里默认是块缓冲的（不是行缓冲），日志会攒在缓冲区里不输出，`docker logs` 就看不到——**容器崩溃时最后那几行关键日志正好丢了**。这是排查容器问题时最坑的一条。

**9. `HEALTHCHECK` 是给编排系统看的**。它定期跑那条命令，`exit 0` 算健康。Kubernetes/compose 靠它决定"这个容器是不是该重启 / 该不该往它转发流量"。**注意 `--start-period=5s`**：启动期内失败不算数，避免"还没启动完就被判死"。

**10. `--host 0.0.0.0` 在容器里是必须的。** 容器有自己的网络命名空间，`127.0.0.1` 指的是**容器内部**的回环口，宿主机的请求进不来。**这是新手部署 Docker 时最常见的"端口映射了但连不上"的原因。** 注意这和 10.9 节 `serve` 默认 `127.0.0.1` 不矛盾——那是本机开发，这是容器内部，容器的对外暴露由 `ports:` 那一层控制。

`.dockerignore`——**和 `.gitignore` 一样重要**：

```
.venv
.git
.gitignore
__pycache__
*.pyc
.pytest_cache
.mypy_cache
.ruff_cache
dist
data
notes.json
tests
.env
requirements.txt
```

**三个理由，按重要性排**：

1. **`.env` 必须排除——否则你的密钥会被烤进镜像**。镜像会被推到仓库、被别人拉取，**镜像层里的文件即使后续层删掉了也还在历史层里能翻出来**。这和 git 提交历史里的密钥是同一类问题。
2. **`.venv` 必须排除**——它是你 macOS 上的环境，里面的二进制在 Linux 容器里跑不了，还会覆盖掉容器里正确的 `.venv`。而且几百 MB，拖慢构建。
3. **`.git`、缓存目录排除**是为了减小构建上下文（Docker 会把整个目录打包发给 daemon）。

### 12.4 ✅ Checkpoint 15：构建镜像并验证缓存机制

```bash
docker build -t notekit:0.1.0 .
docker images notekit
```

真实输出：

```
IMAGE           ID             DISK USAGE   CONTENT SIZE   EXTRA
notekit:0.1.0   5e185f6836d7        355MB         82.9MB   U
```

**355MB 磁盘占用 / 82.9MB 压缩后内容**。其中你的代码只有 11KB（11.2 节量过），剩下全是 Debian 基础系统 + Python + 29 个依赖包。**想更小可以换 `alpine` 基础镜像，但 alpine 用 musl libc，很多带 C 扩展的包（numpy、pandas）没有现成 wheel，要现场编译——通常不值得。**

**现在验证 12.3 第 2 点说的层缓存**。我在 `cli.py` 末尾加了一行纯注释（**只改代码，不改任何依赖**）：

```python
# 一行注释，只改代码不改依赖
```

然后重新构建：

```bash
docker build -t notekit:0.1.1 .
```

关键的两行输出：

```
#8 [builder 4/7] RUN uv sync --frozen --no-install-project --no-dev
#8 CACHED

#11 [builder 7/7] RUN uv sync --frozen --no-dev
#11 ...（重新执行了）
```

**装 39 个依赖那一层是 `CACHED`，只有装项目本身那一层重跑了。** 这就是把 `COPY pyproject.toml uv.lock` 放在 `COPY src` 前面换来的收益——**改代码的构建从几十秒变成几秒**。

> **对照 Android**：这和 Gradle 的增量编译 + 构建缓存是同一个思路，区别是 Docker 的缓存粒度是"整层"，而且**一层失效后面全失效**。所以 Dockerfile 里指令的顺序原则是：**越少变的放越前面**。

### 12.5 compose：把运行参数写进文件

`docker run` 的参数一多就记不住了。compose 把它们写成文件：

`compose.yaml`：

```yaml
services:
  api:
    build: .
    image: notekit:0.1.0
    ports:
      - "8202:8000"
    volumes:
      - notekit-data:/data
    environment:
      # 生产环境的 key 从宿主机环境变量透传，绝不写进 compose 文件
      LLM_API_KEY: ${LLM_API_KEY:-}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx,sys; sys.exit(0 if httpx.get('http://127.0.0.1:8000/health').status_code==200 else 1)"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 5s

volumes:
  notekit-data:
```

**逐项解释**：

| 配置 | 含义 |
|---|---|
| `build: .` | 用当前目录的 Dockerfile 构建 |
| `ports: "8202:8000"` | **宿主机 8202 → 容器 8000**。左边是外面，右边是里面 |
| `volumes: notekit-data:/data` | 把一个命名卷挂到容器的 `/data`——**数据活过容器删除** |
| `environment: LLM_API_KEY: ${LLM_API_KEY:-}` | **从宿主机环境变量透传**，`:-` 是"没设置就用空字符串" |
| `restart: unless-stopped` | 容器挂了自动重启，除非你手动停 |
| `healthcheck` | 和 Dockerfile 里那条等价（compose 里写一遍可以覆盖镜像里的） |

**`${LLM_API_KEY:-}` 这一行是密钥纪律在容器环境的落地**：compose 文件是要提交进 git 的，所以**里面绝不能有真实值**，只能写"去宿主机环境变量拿"。宿主机上的值来自 `.env`（compose 会自动读同目录的 `.env`）或 CI 的 secrets。

**`:-` 的作用别小看**：不写它的话，宿主机没设这个变量时 compose 会报警告或传字面量。`:-` 明确表达"这个是可选的"。

跑起来：

```bash
docker compose up -d --build
docker compose ps
```

真实输出：

```
NAME            IMAGE           STATUS                   PORTS
notekit-api-1   notekit:0.1.0   Up 34 minutes (healthy)   0.0.0.0:8202->8000/tcp, [::]:8202->8000/tcp
```

**`(healthy)` 是 HEALTHCHECK 报的**——说明容器不只是"在跑"，而是"接口真的能响应"。**只看 `Up` 是不够的**：进程活着但接口 500 的情况很常见。

常用命令：

```bash
docker compose logs -f api       # 跟踪日志（PYTHONUNBUFFERED=1 才看得见）
docker compose exec api sh       # 进容器里看看
docker compose restart api       # 重启
docker compose down              # 停止并删除容器（卷保留）
docker compose down -v           # 连卷一起删（数据没了！）
```

**`docker compose down -v` 会删数据卷，这个操作不可逆。** 生产环境上执行前想清楚。

### 12.6 ✅ Checkpoint 16：容器里的三件事

**第一件：确认不是 root 在跑。**

```bash
docker exec notekit-api-1 id
```

```
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
```

**不是 `uid=0(root)`，安全要求达标。**

**第二件：确认容器里的 Python 版本。**

```bash
docker exec notekit-api-1 python -V
```

```
Python 3.12.12
```

**注意和我本机的 3.12.13 差了一个补丁号**——因为镜像里的 Python 来自镜像作者构建时的版本。**这正是 Docker 的价值：容器里的版本由镜像固定，不受我本机影响；反过来也说明"镜像固定"和"本机一致"是两件事，本机测过 ≠ 容器里一样。**

**第三件：确认数据真的持久化了。** 这是最容易出错的一环，要分两级验证：

```bash
# 写一条数据
curl -s -X POST http://127.0.0.1:8202/notes -H 'Content-Type: application/json' \
  -d '{"title":"容器里写的","tags":["docker"]}'
```

```
{"id":2,"title":"容器里写的","body":"","tags":["docker"],"done":false,"created_at":"2026-08-16T10:29:24.026027Z"}
```

```bash
# 一级：重启容器（进程重来，容器还是那个）
docker compose restart api && sleep 5
curl -s http://127.0.0.1:8202/notes

# 二级：删掉容器再重建（这才是真考验）
docker compose down
docker compose up -d && sleep 6
curl -s http://127.0.0.1:8202/notes
```

`down` 的真实输出（容器和网络都被删掉了）：

```
 Container notekit-api-1 Stopped
 Container notekit-api-1 Removing
 Container notekit-api-1 Removed
 Network notekit_default Removing
 Network notekit_default Removed
```

`up -d` 之后再查，数据还在：

```
[{"id":1,"title":"容器里的笔记","body":"","tags":["docker"],"done":false,"created_at":"2026-08-16T09:52:13.725737Z"},{"id":2,"title":"容器里写的","body":"","tags":["docker"],"done":false,"created_at":"2026-08-16T10:29:24.026027Z"}]
```

```
NAME            IMAGE           SERVICE   CREATED         STATUS                   PORTS
notekit-api-1   notekit:0.1.0   api       6 seconds ago   Up 6 seconds (healthy)   0.0.0.0:8202->8000/tcp, [::]:8202->8000/tcp
```

**容器是新建的（`CREATED 6 seconds ago`），但两条笔记都还在**——说明数据在卷里而不在容器的可写层里。**只测 `restart` 是不够的**：`restart` 不删容器，容器自己的文件系统还在，**即使你完全没配卷也能"看起来正常"**。必须测 `down` + `up`。

看一眼卷：

```bash
docker volume ls | grep note
```

```
local     notekit-data
local     notekit_notekit-data
```

> **这里有个真实的坑，我撞了。** 两个卷名，因为：
> - `notekit-data` 是我早先用 `docker run -v notekit-data:/data` 手动建的
> - `notekit_notekit-data` 是 compose 建的——**compose 会自动给卷名加上"项目名_"前缀**（项目名默认是目录名 `notekit`）
>
> **后果**：我以为在用同一个卷，实际是两个，数据"莫名消失"。排查了半天。
>
> **解决办法**：在 compose 里显式声明外部卷：
> ```yaml
> volumes:
>   notekit-data:
>     name: notekit-data       # 不加项目名前缀
> ```
> 或者干脆统一都用 compose 管，别混用 `docker run`。**看到"数据不见了"，第一件事是 `docker volume ls` 数一下有几个卷。**

### 12.7 部署形态怎么选

| 方式 | 适合 | 代价 |
|---|---|---|
| **裸机 + systemd** | 单机小服务、内部工具 | 要自己管 Python 版本、依赖、日志轮转 |
| **Docker + compose** | 单机到几台机器，**绝大多数场景的正确答案** | 要学 Docker |
| **Kubernetes** | 几十个服务、要自动扩缩容 | 复杂度高一个数量级，小团队通常不值 |
| **Serverless**（云函数） | 流量稀疏、突发 | 冷启动慢（Python 尤其）、有执行时长上限 |
| **PaaS**（Railway/Fly.io/云厂商容器服务） | 想少操心，接受平台绑定 | 贵一些，可控性低 |

**给 Agent 服务的具体建议**：Agent 的特点是**每个请求要等外部大模型几秒到几十秒**。这意味着：

1. **超时设置要往大调**——网关、反向代理、客户端三处的超时都要检查。默认的 30 秒经常不够。
2. **别用 Serverless 跑长 Agent 循环**——多步 Agent 很容易超过执行时长上限。
3. **必须用 async**（6.6/6.7 节）——同步阻塞会让一个 worker 只能同时处理一个请求。
4. **要有 `max_steps` 保险丝**（前面章节反复强调过）——Agent 循环不设上限，一个 bug 能把你的 API 额度烧光。
5. **密钥只从环境变量注入**，容器镜像里、compose 文件里、日志里都不能有。

---
## 十三、调试与踩坑：Python 特有的那些坑

Android 里编译器帮你挡掉的错误，Python 里会在**运行时**才炸。所以你需要两样东西：**会读报错**，和**知道哪些坑是 Python 特有的**。

### 13.1 读 Traceback：从下往上看

```python
import traceback


def layer3(data):
    return data["missing"]          # 这里才是真正的错


def layer2(data):
    return layer3(data)


def layer1(data):
    return layer2(data)


try:
    layer1({"a": 1})
except KeyError:
    print(traceback.format_exc())
```

真实输出：

```
Traceback (most recent call last):
  File "/private/tmp/verify/py/h_debug.py", line 22, in <module>
    layer1({"a": 1})
  File "/private/tmp/verify/py/h_debug.py", line 18, in layer1
    return layer2(data)
           ^^^^^^^^^^^^
  File "/private/tmp/verify/py/h_debug.py", line 14, in layer2
    return layer3(data)
           ^^^^^^^^^^^^
  File "/private/tmp/verify/py/h_debug.py", line 10, in layer3
    return data["missing"]          # 这里才是真正的错
           ~~~~^^^^^^^^^^^
KeyError: 'missing'
```

**读法（和 Java 栈轨迹相反！）**：

| 位置 | 内容 |
|---|---|
| **最后一行** | 错误类型 + 消息 —— **先看这个** |
| **最靠下的 `File` 行** | **出错的那一行**，也就是最深的调用 |
| 往上的 `File` 行 | 调用链，从入口一路调下来 |
| 第一行 | 最外层的调用点 |

**Java 的 stack trace 是最上面那行最深，Python 是最下面那行最深。** 这个方向差异会让你一开始很别扭，记住"Python 从下往上看"就行。

**`^^^^` 和 `~~~~` 是 Python 3.11+ 送的礼物**：它精确指出是表达式的哪一部分出错。上面 `~~~~^^^^^^^^^^^` 中，`~~~~` 标出 `data`、`^^^^` 标出 `["missing"]`——**告诉你是下标操作错了，不是 `data` 本身有问题**。一行里有多个下标/调用时，这个标记非常省时间。

### 13.2 七种最常见的报错，对号入座

```python
cases = [
    ("NameError", lambda: undefined_variable),
    ("TypeError", lambda: "1" + 1),
    ("AttributeError", lambda: None.strip()),
    ("KeyError", lambda: {"a": 1}["b"]),
    ("IndexError", lambda: [1, 2][5]),
    ("ValueError", lambda: int("abc")),
    ("ZeroDivisionError", lambda: 1 / 0),
]
for name, fn in cases:
    try:
        fn()
    except Exception as e:
        print(f"  {name:<20} -> {type(e).__name__}: {e}")
```

真实输出：

```
  NameError            -> NameError: name 'undefined_variable' is not defined
  TypeError            -> TypeError: can only concatenate str (not "int") to str
  AttributeError       -> AttributeError: 'NoneType' object has no attribute 'strip'
  KeyError             -> KeyError: 'b'
  IndexError           -> IndexError: list index out of range
  ValueError           -> ValueError: invalid literal for int() with base 10: 'abc'
  ZeroDivisionError    -> ZeroDivisionError: division by zero
```

| 报错 | 意思 | 最常见的真实原因 | Kotlin 里对应 |
|---|---|---|---|
| `NameError` | 用了不存在的名字 | **打错字**、忘了 import、变量定义在了 if 分支里 | 编译期就报错 |
| `TypeError` | 类型不对 | 把 `str` 和 `int` 相加、参数个数不对 | 编译期就报错 |
| `AttributeError: 'NoneType' object has no attribute ...` | **对 None 调方法** | **函数返回了 None 你没检查** | `NullPointerException` |
| `KeyError` | 字典里没这个键 | **解析大模型返回的 JSON 时字段名对不上** | 无（Kotlin map 返回 null） |
| `IndexError` | 下标越界 | 列表是空的 | `IndexOutOfBoundsException` |
| `ValueError` | 类型对但值不合法 | `int("abc")`、用户输入没校验 | `NumberFormatException` |
| `ModuleNotFoundError` | 找不到模块 | **装到了别的环境**（5.4 节） | 依赖没声明 |

**`AttributeError: 'NoneType' object has no attribute` 是你会见得最多的一个**，它就是 Python 的空指针异常。**根因几乎总是"某个函数返回了 `None` 而你直接用了"**——比如 `store.get(id)` 找不到时返回 `None`，你没判断就 `.title`。**这也是 mypy 最能帮到你的场景**：`Note | None` 类型的值，mypy 会强制你先判空（7 节）。

**排查 `AttributeError` 的两个高效手段**：

```python
print(type(x))       # 它到底是什么类型
print(dir(x))        # 它到底有哪些方法/属性
```

### 13.3 浅拷贝 vs 深拷贝

```python
import copy

original = {"name": "小林", "tags": ["a", "b"]}
shallow = original.copy()
deep = copy.deepcopy(original)
shallow["tags"].append("c")
print("    original:", original)
print("    shallow :", shallow)
print("    deep    :", deep)
```

真实输出：

```
  改了 shallow['tags'] 之后:
    original: {'name': '小林', 'tags': ['a', 'b', 'c']}
    shallow : {'name': '小林', 'tags': ['a', 'b', 'c']}
    deep    : {'name': '小林', 'tags': ['a', 'b']}
```

**`.copy()` 只复制第一层**：新字典是新的，但里面那个列表**还是同一个对象**，所以改了 `shallow` 的列表，`original` 也变了。**这就是 2.2 节"变量是标签不是盒子"的直接后果。**

| 方式 | 复制深度 | 什么时候用 |
|---|---|---|
| `d2 = d1` | **完全不复制**，只是多一个标签 | 想要同一个对象 |
| `d1.copy()` / `dict(d1)` / `list(l1)` / `l1[:]` | **一层** | 里面全是不可变值（数字、字符串）时够用 |
| `copy.deepcopy(d1)` | **递归全部** | 有嵌套的可变对象时**必须** |
| `note.model_copy()`（pydantic） | 一层（`deep=True` 才递归） | pydantic 模型 |

**真实场景**：给大模型准备 messages 列表时，你想在原始对话上加一条系统提示但不污染原列表——`messages.copy()` 是不够的，因为每条消息本身是字典。要么 `copy.deepcopy`，要么 `[*messages, new_msg]`（新建列表，元素共享但你不改它们）。

### 13.4 `[[0]*3]*3`：Python 最著名的坑

```python
grid_bad = [[0] * 3] * 3
grid_bad[0][0] = 9
print(grid_bad)

grid_ok = [[0] * 3 for _ in range(3)]
grid_ok[0][0] = 9
print(grid_ok)
```

真实输出：

```
  [[0]*3]*3 改一个 -> [[9, 0, 0], [9, 0, 0], [9, 0, 0]]   <- 三行是同一个列表!
  推导式生成   改一个 -> [[9, 0, 0], [0, 0, 0], [0, 0, 0]]
```

**`列表 * 3` 复制的是"引用"，不是"内容"。** 三行是同一个列表对象的三个标签，改一个全变。

**规则**：**要建二维结构，永远用推导式 `[[0]*n for _ in range(m)]`。** 注意内层的 `[0]*3` 是安全的——因为 `0` 是不可变的整数，共享也无所谓。**只有元素是可变对象（列表、字典、自定义类）时才会踩这个坑。**

**这和 2.10 节的可变默认参数、3.x 节的 `field(default_factory=list)` 是同一个根源**：Python 里"复制一个可变对象"必须显式做，不会帮你自动做。

### 13.5 遍历时修改列表

```python
nums = [1, 2, 4, 6, 7]
for n in nums[:]:            # 注意 [:] —— 遍历副本
    if n % 2 == 0:
        nums.remove(n)
print("  遍历副本再删，正确:", nums)

nums2 = [1, 2, 4, 6, 7]
for n in nums2:              # 直接遍历原列表
    if n % 2 == 0:
        nums2.remove(n)
print("  直接遍历边删，出错:", nums2)
```

真实输出：

```
  遍历副本再删，正确: [1, 7]
  直接遍历边删，出错: [1, 4, 7]   <- 4 被跳过了
  过程拆解(遍历原列表时):
    下标0 看到 1  列表=[1, 2, 4, 6, 7]
    下标1 看到 2  列表=[1, 2, 4, 6, 7]
    下标2 看到 6  列表=[1, 4, 6, 7]
  最推荐的写法是根本不删，而是生成新列表:
    [1, 7]
```

**看那个过程拆解就懂了**：删掉下标 1 的 `2` 之后，`4` 挪到了下标 1，但循环的指针已经走到下标 2 了——**`4` 被跳过，永远不会被检查**。

**注意 Python 和 Java 的区别**：Java 的 `ArrayList` 在遍历中修改会抛 `ConcurrentModificationException`——**它至少会报错**。Python **默默给你一个错的结果**，这更危险。（顺带说，Python 的 `dict` 在遍历中改大小会抛 `RuntimeError: dictionary changed size during iteration`，比 list 好一点。）

**三种正确写法，按推荐度排**：

```python
nums = [n for n in nums if n % 2 != 0]   # 1. 最好：生成新列表，意图最清楚
for n in nums[:]:  ...                    # 2. 遍历副本
for i in range(len(nums) - 1, -1, -1):   # 3. 倒着遍历（删了不影响没走过的下标）
```

**第一种是 Python 的惯用法**——"过滤"这个意图直接写成推导式，比"边遍历边删"清晰得多，也不会有性能问题（`list.remove` 是 O(n)，循环里删是 O(n²)）。

### 13.6 `is` 和 `==`：三个反直觉的实测

```python
a = 1000
b = 1000
print("a=1000; b=1000 -> a is b ?", a is b)

c, d = int("1000"), int("1000")
print("int('1000') is int('1000') ?", c is d, "| c == d ->", c == d)

print("100 is int('100') ?", 100 is int("100"))
```

真实输出：

```
  写在同一个文件里的 a=1000; b=1000 -> a is b ? True
    （CPython 会把同一个代码块里的相同常量合并成一个对象，所以是 True）
  运行时算出来的 int('1000') is int('1000') ? False
    但值相等吗？ c == d -> True
  小整数 100 is int('100') ? True   （-5~256 被解释器预先缓存了）
  同样的『1000』，三种写法给出三种 is 结果 —— 全是实现细节。
```

**同一个数值 1000，三种写法给出两种 `is` 结果，而 `100` 又不一样。** 原因全是 CPython 的实现细节：
- 同一个代码块里的字面量常量会被合并（编译器优化）
- 运行时计算出的整数是新对象
- **`-5` 到 `256` 的小整数被解释器预先缓存**，永远是同一个对象

**结论（这条是硬规则，别记那些细节）**：

> **判断值相等永远用 `==`。`is` 只用来判断"是不是同一个对象"，实践中基本只写 `x is None` / `x is True` / `x is False`。**

**为什么 `is None` 反而推荐用 `is`？** 因为 `None` 全局只有一个对象，`is None` 又快又准；而 `== None` 可能被自定义的 `__eq__` 改变行为。**同理，判断布尔值用 `x is True`**——`1 == True` 是 `True`（2.4 节），但 `1 is True` 是 `False`。这就是 `store.py` 里写 `n.done is done` 而不是 `n.done == done` 的原因。

### 13.7 `breakpoint()`：内置调试器

```python
def buggy(items):
    total = 0
    for it in items:
        breakpoint()        # 运行到这里会停下来，进入 pdb
        total += it["price"]
    return total
```

进入 pdb 后的常用命令：

| 命令 | 作用 |
|---|---|
| `n`（next） | 执行下一行，**不进入函数** |
| `s`（step） | 执行下一行，**进入函数内部** |
| `c`（continue） | 继续跑，直到下一个断点 |
| `p 变量名` | 打印变量 |
| `pp 变量名` | 漂亮地打印（嵌套结构会换行缩进） |
| `l`（list） | 看当前位置附近的代码 |
| `w`（where） | 看调用栈 |
| `q`（quit） | 退出 |

**`breakpoint()` 是 3.7+ 的内置函数**，不需要 import。IDE 里点行号打断点效果一样，原理就是这个。**在容器里、在 CI 里、在 SSH 上没有 IDE 的时候，`breakpoint()` 是唯一的选择。**

**还有一个更轻量的技巧**——f-string 的 `=` 语法（2.3 节）：

```python
print(f"{items=} {total=}")
```

比 `print("items:", items, "total:", total)` 少打一半字，而且不会写错变量名对应关系。**80% 的调试用这个就够了。**

### 13.8 全章踩坑总表

按"你多久会撞一次"排序：

| # | 坑 | 现象 | 正确做法 | 出处 |
|---|---|---|---|---|
| 1 | **缩进不一致** | `IndentationError` / `TabError` | **统一 4 空格**，编辑器设置"Tab 转空格" | §2.1 |
| 2 | **对 `None` 调方法** | `AttributeError: 'NoneType' ...` | 函数返回 `X \| None` 时**必须判空**；用 mypy 强制 | §13.2 |
| 3 | **可变默认参数** | 第二次调用带上了上次的数据 | `def f(x=None): x = x or []`；dataclass/pydantic 用 `default_factory` | §2.10 |
| 4 | **`ModuleNotFoundError`** | 明明装了却说找不到 | 确认在同一个环境：`uv run python -c "import sys; print(sys.executable)"` | §5.3 |
| 5 | **忘了 `encoding="utf-8"`** | Windows 上中文乱码/报错 | **每次读写文本文件都显式写** | §4.4 |
| 6 | **`model_dump()` 忘了 `mode="json"`** | `TypeError: Object of type datetime is not JSON serializable` | 要转 JSON 就用 `mode="json"` | §10.4 |
| 7 | **`==` 和 `is` 混用** | 时对时错，难复现 | 值比较用 `==`；只用 `is None` | §13.6 |
| 8 | **浅拷贝以为是深拷贝** | 改副本原件也变了 | 嵌套结构用 `copy.deepcopy` | §13.3 |
| 9 | **`[[0]*3]*3`** | 改一行三行全变 | `[[0]*3 for _ in range(3)]` | §13.4 |
| 10 | **遍历时删元素** | **静默**跳过元素，不报错 | 用推导式生成新列表 | §13.5 |
| 11 | **`except Exception` 太宽** | 真 bug（打错字、类型错）被当成"预期失败"吞掉 | 捕获具体异常；`try` 块越小越好（`Ctrl+C` 倒是吞不掉——实测 `KeyboardInterrupt` 不是 `Exception` 的子类） | §4.3 |
| 12 | **`async def` 里用 `time.sleep`** | 并发完全失效，实测 0.61s vs 0.30s | 用 `await asyncio.sleep`；阻塞库用 `asyncio.to_thread` | §6.7 |
| 13 | **CPU 密集用多线程** | 一点没快，实测 0.70 → 0.67 | GIL 决定的，用 `ProcessPoolExecutor` | §8.3 |
| 14 | **naive/aware datetime 混用** | `can't compare offset-naive and offset-aware datetimes` | **内部永远用 `datetime.now(UTC)`** | §8.1 |
| 15 | **类型注解以为会检查** | 传错类型照样跑，输出 `12` | **必须单独跑 `mypy`** | §7 |
| 16 | **相对导入 + 直接运行文件** | `ImportError: attempted relative import with no known parent package` | 用 `python -m pkg.module` | §5.3 |
| 17 | **循环导入** | `ImportError: cannot import name ... (most likely due to a circular import)` | **保持单向依赖**；必要时函数内 import | §5.3 |
| 18 | **用内置名当变量名** | `TypeError: 'function' object is not subscriptable` 等诡异错误 | 别用 `list`/`dict`/`id`/`type`/`str` | §10.4 |
| 19 | **`model_copy` 不校验** | 能造出违反约束的对象 | 改完补一次 `model_validate` | §10.4 |
| 20 | **PATCH 用了 `model_dump()`** | 只想改一个字段，结果其他字段被清空 | `model_dump(exclude_unset=True)` | §10.4 |
| 21 | **容器里监听 `127.0.0.1`** | 端口映射了但连不上 | 容器内用 `--host 0.0.0.0` | §12.3 |
| 22 | **容器日志看不见** | `docker logs` 是空的 | `PYTHONUNBUFFERED=1` | §12.3 |
| 23 | **Dockerfile 先拷源码后装依赖** | 改一个字符重装 40 个包 | **先 `COPY pyproject.toml uv.lock`** | §12.3/12.4 |
| 24 | **compose 卷名带项目前缀** | 数据"莫名消失" | `docker volume ls` 数一下；用 `name:` 固定 | §12.6 |
| 25 | **`.env` 进了 git 或镜像** | **密钥泄露** | `.gitignore` + `.dockerignore` 都要写 | §10.5/12.3 |
| 26 | **`uv venv` 没有 pip** | `No module named pip` | 用 `uv pip install`，或 `uv venv --seed` | §11.3 |
| 27 | **多 worker 共享内存缓存** | 4 个 worker 4 份缓存，行为不一致 | 用 Redis/数据库；或干脆无状态 | §12.1 |
| 28 | **JSON 文件存储 + 多进程** | 并发写互相覆盖，丢数据 | 用 SQLite/Postgres | §12.1 |

---
### 13.9 专家提示：从"能写"到"写得像样"

这些不是语法，是**工程习惯**。Android 开发里你已经有一套，这里是 Python 的对应版本。

**1. 把三道门做成一条命令，并接进 git hook / CI。**

```bash
uv run ruff check . && uv run mypy src && uv run pytest -q
```

**新项目第一天就把 mypy 打开，别等代码写了两万行再补。** 中途引入 mypy 的痛苦程度和中途给一个 Java 项目引入空安全检查差不多。用 `pre-commit` 工具可以让它在 `git commit` 时自动跑。

**2. 类型注解写在"边界"上，内部可以松。**

公开函数的参数和返回值**一定要写**（别人靠它理解你的代码，mypy 靠它检查）；函数内部的临时变量通常不用写（mypy 能推断）。**投入产出比最高的三处**：公开 API、返回 `X | None` 的函数、数据模型。

**3. 优先用标准库，其次用"事实标准"库。**

Python 的"电池"（标准库）非常全：`pathlib`/`json`/`datetime`/`itertools`/`collections`/`logging`/`sqlite3`/`asyncio`。**先问"标准库有没有"**。要装第三方时看三个指标：**最近有没有更新、GitHub star/issue 响应速度、有没有类型标注（`py.typed`）**。

**4. 日志用 `logging`，不用 `print`。**

`print` 没有级别、没有时间戳、没法关掉、进不了日志文件。**服务端代码里出现 `print` 就是技术债。** 而且记住 4.7 节的两点：用 `log.info("耗时 %d ms", ms)` 而不是 f-string（懒惰求值，级别不够时不做格式化），异常用 `log.exception()` 自动带上 traceback。

**5. 别在模块顶层做有副作用的事。**

模块顶层的代码在 `import` 时就会执行（5.1 节）。**在顶层连数据库、读远程配置、启动线程，会让导入变慢、让测试难写、让循环导入更容易出现。** 把它们放进函数里，或用 FastAPI 的 lifespan 事件。`config.py` 里那句 `load_dotenv()` 是个例外——它只读本地文件、幂等、且必须早于一切配置读取。

**6. 拥抱不可变。**

能用 `tuple` 就不用 `list`，能返回新对象就不改原对象（`model_copy` 而不是原地改）。Kotlin 里你已经习惯 `val` 优先，Python 没有 `val`，**但设计上可以做到同样的效果**——这能消掉一大半"改了副本原件也变了"类的 bug（13.3/13.4）。

**7. 虚拟环境一个项目一个，永远不要 `pip install` 到系统 Python。**

macOS/Linux 的系统 Python 是操作系统自己要用的，装乱了会修不回来（1 节量过：系统 Python 是 3.9.6）。**永远 `uv run` 或激活项目的 `.venv`。**

**8. 写代码时就想"这段怎么测"。**

`notekit` 的分层不是为了好看：`store` 不知道谁在用它，所以能直接测；`get_store` 是个依赖，所以能替换。**如果一段逻辑难测，通常说明它耦合太紧**——这是设计信号，不是测试的问题。

**9. 读官方文档，别只读教程。**

Python 的官方文档质量很高，尤其是标准库部分。pydantic、FastAPI、typer 的文档都是同一水平。**遇到问题优先查官方文档 + 版本号**——因为 pydantic v1→v2、typer、fastapi 都经历过 API 变化，网上的老答案会把你带偏。

**10. 定期 `uv lock --upgrade` 并跑测试。**

依赖长期不升会积累技术债，一次升级几十个包会痛不欲生。**每两周升一次、跑一次测试**，比半年后一次性升要轻松得多。

---

## 十四、面试视角

以下问题在 Python 后端 / AI 工程岗面试里出现频率很高。**每一条都能在本章找到实测依据。**

**1. Python 的 GIL 是什么？它意味着什么？**

GIL（全局解释器锁）是 **CPython 解释器**的一个互斥锁，保证同一时刻只有一个线程执行 Python 字节码。**注意它是"CPython 的实现细节"，不是 Python 语言规范**——Jython、PyPy 的情况不同，CPython 3.13 也有实验性的 free-threaded 构建。

后果（§8.3 实测）：**CPU 密集任务多线程完全不会变快**（单线程 0.70 秒 → 4 线程 0.67 秒），**多进程才有效**（0.22 秒）；**IO 密集任务多线程有效**（1.22 秒 → 0.30 秒），因为等 IO 时会释放 GIL。

**加分回答**：所以写 Agent 服务时你几乎永远不需要 `multiprocessing`——瓶颈在等大模型返回，那是 IO，用 `asyncio` 就够，而且比线程更省资源。

**2. `async`/`await` 和多线程的区别？什么时候 async 不管用？**

async 是**单线程内的协作式并发**：遇到 `await` 就主动让出控制权，让事件循环去跑别的任务。多线程是**操作系统抢占式调度**。async 没有线程切换开销，能轻松开几万个并发任务。

**async 失效的条件（§6.7 实测）**：`async def` 里调用了**阻塞**的代码。把 `await asyncio.sleep(0.3)` 换成 `time.sleep(0.3)`，2 个任务的耗时从 0.30 秒变成 0.61 秒——**完全串行，async 白写了**。原因是 `time.sleep` 不会让出事件循环。

**解决办法**：换成 async 版本的库（`requests` → `httpx.AsyncClient`），或用 `asyncio.to_thread(阻塞函数, 参数)` 把它扔到线程池。

**3. 可变默认参数为什么是坑？**

`def f(items=[])` 的默认值列表**在函数定义时只创建一次**，之后所有调用共享它。所以第二次调用会看到第一次留下的数据。

正确写法：`def f(items=None): items = [] if items is None else items`。dataclass 用 `field(default_factory=list)`，pydantic 用 `Field(default_factory=list)`。

**加分**：ruff 的 `B006`/`B008` 规则专门抓这个；但 Typer 的 API 设计要求在默认值里调 `Option()`，所以要用 `per-file-ignores` 精确放行（§10.7）——**这体现你懂"为什么放行"而不是"关掉烦人的警告"**。

**4. 类型提示会在运行时检查吗？**

**不会。** §7 实测：`add("1", "2")` 有类型注解 `(int, int) -> int`，Python 照样输出 `12`（字符串拼接）；同一份代码 mypy 报 2 个 `arg-type` 错误。

**注解只是元数据**，存在 `__annotations__` 里，运行时可以读（pydantic 和 FastAPI 就靠读它工作）。**要真正检查必须单独跑 mypy 或 pyright。**

**5. `==` 和 `is` 的区别？为什么 `x is None` 反而推荐？**

`==` 比较值（可被 `__eq__` 重载），`is` 比较是否同一个对象（比较内存地址，不可重载）。

§13.6 实测：同一个文件里 `a=1000; b=1000` → `a is b` 是 `True`（编译器常量合并）；`int('1000') is int('1000')` 是 `False`；`100 is int('100')` 是 `True`（小整数缓存 -5~256）。**这些全是 CPython 实现细节，不能依赖。**

`is None` 推荐是因为 `None` 全局单例，又快又不会被 `__eq__` 骗。**判断布尔也用 `is`**：`1 == True` 为真，但 `1 is True` 为假。

**6. 深拷贝和浅拷贝？**

`.copy()` 只复制第一层，嵌套的可变对象仍然共享（§13.3 实测：改 `shallow['tags']`，`original` 也变了）。`copy.deepcopy()` 递归复制。

**衍生问题**：`[[0]*3]*3` 为什么改一个变三个？因为 `*` 复制引用，三行是同一个列表对象（§13.4）。

**7. 装饰器是什么？为什么要 `functools.wraps`？**

装饰器是一个"接收函数、返回函数"的函数。`@timed def f(): ...` 完全等价于 `f = timed(f)`。

`functools.wraps` 把原函数的 `__name__`/`__doc__`/`__annotations__` 复制到包装函数上。**不加的话，被装饰函数的名字会变成 `wrapper`**，日志、调试、以及依赖签名的框架（FastAPI！）全都会出问题。

**带参数的装饰器需要三层嵌套**：`retry(times=3)` → 外层收参数、中层收函数、内层是包装（§6.3）。

**8. 生成器和列表的区别？**

生成器**惰性求值**，一次只算一个、不存全部结果。§6.5 实测：装 10 万个数的列表占 **800984 字节**，等价生成器占 **200 字节**；而且从输出顺序能看出 `yield` 之后函数真的暂停了。

**用在哪**：读大文件（一行一行）、流式处理大模型的输出（token 一个个来）、无限序列。**代价**：只能遍历一次，没有 `len()`，不能下标访问。

**9. `pyproject.toml`、`uv.lock`、`requirements.txt` 的关系？哪些要提交？**

- `pyproject.toml`：**声明"我要什么"**（范围，如 `pydantic>=2`），必须提交
- `uv.lock`：**记录"实际装了什么"**（精确版本 + 哈希，19 万字节记 40 个包），必须提交
- `requirements.txt`：`uv export` 的**导出产物**（447 行，带 `--hash`），给不装 uv 的环境用，不要手改

**关键原则**：**库声明宽松范围（避免版本冲突），应用用锁文件钉死（保证可复现）**。所以 wheel 的 `METADATA` 里是 `>=`，而部署时用 `uv sync --frozen`。

**10. Docker 多阶段构建解决什么问题？为什么 `COPY` 顺序重要？**

多阶段让构建工具和中间产物留在第一阶段，**最终镜像只带 `.venv` 和源码**——更小、更安全。

`COPY` 顺序：Docker 每条指令一层，**一层失效则后面全部失效**。所以要**先拷变化少的（`pyproject.toml`/`uv.lock`）装依赖，后拷变化多的源码**。§12.4 实测：只改了 `cli.py` 一行注释，装 39 个依赖那层显示 `CACHED`，只有装项目本身那层重跑。

**加分三点**：`USER appuser`（不用 root）、`PYTHONUNBUFFERED=1`（否则日志看不见）、容器里必须 `--host 0.0.0.0`（否则端口映射了也连不上）。

**11. PATCH 接口怎么区分"传了 null"和"没传"？**

pydantic 的 `model_dump(exclude_unset=True)`——只返回调用方**真正传了**的字段。用普通 `model_dump()` 会把没传的字段当 `None` 一起更新，**把用户的标题清空**（§10.4）。

**这是"要区分 0/None/未设置 就必须用 `is None` 而不是真值判断"这条原则在 API 层的具体形态。**

**12. 怎么测一个 FastAPI 应用？**

`TestClient(api)` **在进程内直接调用接口函数**，不开端口不走网络，所以 8 个 HTTP 测试几十毫秒跑完。

配合 `api.dependency_overrides[get_store] = lambda: store` 把真实存储换成临时目录里的（§10.11）——**业务代码一行不改**。fixture 里 `yield` 之后必须 `clear()`，否则污染后续测试。

**这就是依赖注入的真正价值**：不是为了"解耦"这个抽象说法，而是为了让测试能替换掉外部依赖。

**13. 你怎么保证密钥不泄露？**

四层，**每层都实操过**：
1. 密钥**只放 `.env` 或环境变量**，代码里一个字符都不留
2. `.env` 写进 **`.gitignore`**（进了提交历史就等于已泄露，删了也在历史里）
3. `.env` 写进 **`.dockerignore`**（否则被烤进镜像层，删了也能从历史层翻出来）
4. compose 里只写 `${LLM_API_KEY:-}` **透传**，绝不写真实值

**加分**：打日志只打长度和前缀；前端代码里永远不能出现 LLM 密钥（浏览器里所有人可见，必须后端代理）；一旦泄露立刻去平台吊销重发；PyPI 发布前 `unzip -l` 检查 wheel 里没有 `.env`。

**14. uvicorn 的 worker 数怎么定？**

**先问服务在干什么**。等外部 IO 为主（Agent 服务的典型情况）→ `CPU 核数` 甚至更少就够，瓶颈在网络；CPU 密集 → `核数` 到 `核数+1`。

**别照搬 `2×核数+1`**——那是给同步阻塞框架的公式。FastAPI 的 `async def` 在**单个 worker 内**就能并发处理大量等待中的请求（§6.6 实测：串行 2.47 秒的活并发 0.33 秒，快 7.6 倍）。**先把 async 写对，再考虑加 worker。**

**陷阱**：多 worker 不共享内存，模块级的全局缓存会变成 N 份；`notekit` 的 JSON 文件存储在多 worker 下会并发写覆盖丢数据（**这是示例的已知局限**），真实项目用 SQLite 或 Postgres。

**15. 循环导入怎么解决？**

根本办法是**保持单向依赖**。`notekit` 的依赖链是 `models ← store ← cli/web`，`models` 谁都不依赖——**所以不可能出现循环**。

真出现了，三个办法按推荐度排：**① 重新设计分层**（把公共部分抽到第三个模块）；② 把 import 挪进函数内部（延迟导入，`cli.py` 里的 `import uvicorn` 就是这么写的，顺带还加快了启动）；③ `if TYPE_CHECKING:` + 字符串注解（只为类型检查而 import）。

**加分**：报错信息是 `ImportError: cannot import name X (most likely due to a circular import)`，最后那句提示直接告诉你原因。

---

## 十五、划重点

**一、心态先调对。** 你已经会 90% 的编程了。这一章真正的新东西只有三块：**Python 的语法习惯**、**"Python 的 Gradle" = uv + `pyproject.toml`**、**一条从空目录到容器的完整通路**。其余都是你已有知识的换皮。

**二、最大的思维转变：编译器不再帮你兜底。**

| Kotlin 里编译器帮你挡的 | Python 里谁来挡 |
|---|---|
| 类型错误 | **mypy**（必须单独跑，§7 实测 Python 自己不管） |
| 空指针 | **mypy** 的 `X \| None` + 你自己判空 |
| 拼错的变量名 | **ruff** 的 `F` 规则 |
| 逻辑错误 | **pytest** |

**所以 `ruff check` + `mypy` + `pytest` 三道门不是"加分项"，是把编译器的活补回来。新项目第一天就配好。**

**三、五条会反复救你的规则**：

1. **`is` 只用来比 `None`**，值比较永远用 `==`。
2. **默认参数、类属性、二维列表：可变对象绝不共享**——用 `default_factory`、用推导式。
3. **要区分"没传"和"传了 None"就必须显式判断**（`is None` / `exclude_unset=True`）。
4. **时间内部永远用 `datetime.now(UTC)`**，只在给人看的时候转本地时区。
5. **文本文件读写永远显式 `encoding="utf-8"`**。

**四、并发的三句话（§6/§8 全部实测过）**：

- **CPU 密集 → 多进程**（多线程被 GIL 挡住，0.70 → 0.67 毫无改善；进程 0.22）
- **IO 密集 → async 优先，线程次之**（1.22 → 0.30）
- **`async def` 里一旦出现阻塞调用，async 就白写了**（0.30 → 0.61，完全串行）

**写 Agent 服务基本只需要 asyncio**——因为瓶颈永远是等大模型返回。

**五、工程链路一张图**：

```
空目录
  ↓ uv init --package
项目骨架（src-layout + pyproject.toml）
  ↓ uv add / uv add --dev
依赖装好，uv.lock 生成
  ↓ 写代码：models → store → config → cli/web → tests
  ↓ ruff check . && mypy src && pytest -q     ← 三道门
能跑且质量可控
  ↓ uv build
wheel（7085 字节，12 个文件，不含 tests 和 .env）
  ↓ uv pip install 到全新环境验证 / uv tool install / uv publish
别人能装上
  ↓ docker build（多阶段 + 层缓存 + 非 root + HEALTHCHECK）
镜像（355MB）
  ↓ docker compose up -d
服务在跑（healthy），数据在卷里活过容器重建
```

**这条链路上每一步我都跑过并贴了真实输出**（16 个 ✅ Checkpoint）。**你照着敲一遍，就有了一个可以复用一辈子的项目模板。**

**六、部署与发布的硬规则**：

- **库声明范围（`>=`），应用锁死（`uv.lock` + `--frozen`）。**
- **容器里 `--host 0.0.0.0` + `PYTHONUNBUFFERED=1`**，这两个不加就一定出问题。
- **`.env` 同时要进 `.gitignore` 和 `.dockerignore`。**
- **PyPI 版本号发出去就不能改**，所以先发 TestPyPI。
- **不用 root 跑容器**（`USER appuser`）。
- **持久化必须用 `down` + `up` 验证**，`restart` 验证不出问题。

**七、别把示例代码当生产代码。** 我在文中标出了 `notekit` 的两处已知局限：**JSON 文件存储在多 worker 下会并发写丢数据**（该用 SQLite/Postgres）、**Web API 完全没有认证**（对外提供服务前必须加）。**能识别"教学代码到生产代码之间还差什么"，本身就是工程能力。**

**八、接下来学什么。** 按投入产出比排序：

| 优先级 | 内容 | 为什么 |
|---|---|---|
| 高 | **`sqlite3` / SQLAlchemy** | JSON 文件存储撑不过多进程，这是下一个必然遇到的问题 |
| 高 | **`asyncio` 深入**（`TaskGroup`、超时、取消） | Agent 服务的核心，`gather` 只是入门 |
| 高 | **`pytest` 进阶**（`monkeypatch`、mock、`conftest.py`） | 测 LLM 调用必须会 mock |
| 中 | **FastAPI 的认证与中间件** | 对外提供服务的前提 |
| 中 | **`logging` 的结构化输出（JSON 日志）** | 生产排查问题的基础 |
| 中 | **CI（GitHub Actions）** | 把三道门自动化 |
| 低 | 元类、描述符、`__slots__` | 写框架才需要，写业务基本用不到 |

**别去追元编程。** 把上面"高"的三项吃透，你的 Python 水平就足以支撑本书全部章节的实战了。

---

## 十六、下一章预告

到这里，Python 这门语言和它的工程链路对你不再是障碍。

**回到主线**：本书正文从第 1 章的"什么是 Agent"一路讲到第 12 章的多 Agent 系统。**如果你是从这个附录进来的，建议按下面的顺序回到正文**：

| 你现在想干什么 | 去哪一章 |
|---|---|
| 搞清楚 Agent 到底是什么、和 Chatbot 差在哪 | **第 1 章** |
| 理解大模型内部怎么工作、参数怎么调 | **第 2、3 章** |
| **动手写第一个能用工具的 Agent（ReAct）** | **第 4 章** ← 大多数人应该从这里开始 |
| 不想写代码，先用平台搭一个试试 | **第 5 章** |
| 给 Agent 加记忆和知识库（RAG） | **第 8 章** |
| 让 Agent 之间互相通信（MCP / A2A） | **第 10 章** |
| 微调自己的模型 | **第 11 章** |
| 做多 Agent 协作系统 | **第 12 章** |

**你在本章练出的这些东西，正文里会立刻用到**：

- **§6 的 `async`/`await`** → 并发调多个大模型、并行跑多个工具
- **§9 的 pydantic** → 定义工具的参数 schema、校验模型返回的 JSON
- **§9 的 httpx** → 调所有大模型 API（记得设 `timeout`、复用 `Client`）
- **§4 的异常处理** → 每个工具独立 `try/except`，一个工具挂了不能拖垮整个 Agent
- **§8 的 `deque(maxlen=N)`** → 对话历史的滑动窗口
- **§8 的 `dict.get()` 纪律** → 解析模型返回的 JSON 时不炸 `KeyError`
- **§10 的 FastAPI + 分层** → 把 Agent 包装成 HTTP 服务
- **§12 的 Docker** → 把 Agent 部署出去
- **§13 的读 Traceback** → 你会用得非常频繁

**最后一句**：Python 的语法两天就能看完，但"写出别人能维护、跑得住、部署得出去的 Python"需要的是本章后半部分那些东西——**分层、类型检查、测试、锁文件、容器、密钥纪律**。这些你在 Android 开发里已经有对应的直觉，**现在只是换了一套工具名字**。
