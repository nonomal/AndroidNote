# 01 · Python 基础速成（面向 Agent 项目）

> 本篇不是完整的 Python 教程，而是"为了看懂后面三个项目，你必须掌握的最小知识集"。每个知识点都标注了它将在哪个项目中出现。学完本篇，你应该能读懂 80% 的项目代码。

## 0. 怎么练习

暂时不需要安装任何东西——下一篇才正式装环境。现在可以用浏览器打开任意在线 Python 运行环境（搜索 "python playground" 或使用 https://www.python.org/shell/ ），把本篇每一段代码敲进去运行一遍。

---

## 1. 变量与基本类型

Python 中不需要声明类型，直接赋值：

```python
city = "北京"          # 字符串 str
days = 3               # 整数 int
budget = 1500.5        # 浮点数 float
is_ok = True           # 布尔值 bool（True / False，注意首字母大写）
nothing = None         # 空值（类似其他语言的 null）
```

**f-string 字符串格式化**（三个项目里到处都是，用来拼 Prompt）：

```python
city = "北京"
days = 3
prompt = f"请为我规划{city}的{days}日行程"
print(prompt)   # 请为我规划北京的3日行程
```

`f"..."` 里的 `{}` 会被替换成变量的值。项目里给 LLM 的提示词几乎全是这样拼出来的。

## 2. 列表与字典

**列表 list**：有序的一组值。

```python
npcs = ["张三", "李四", "王五"]
print(npcs[0])        # 张三（下标从0开始）
npcs.append("赵六")   # 追加
print(len(npcs))      # 4（长度）

for name in npcs:     # 遍历
    print(name)
```

**字典 dict**：键值对，Agent 项目中所有"结构化数据"的基础。

```python
npc = {
    "name": "张三",
    "role": "Python工程师",
    "friendliness": 50,
}
print(npc["name"])            # 张三
npc["friendliness"] += 2      # 修改值
print(npc.get("age", 18))     # 18（键不存在时返回默认值，不报错）
```

**嵌套**：真实数据往往是"字典里套列表、列表里套字典"。旅行助手返回的行程计划就是这种结构：

```python
trip_plan = {
    "city": "北京",
    "days": [
        {"date": "2026-09-01", "attractions": ["故宫", "景山公园"]},
        {"date": "2026-09-02", "attractions": ["长城"]},
    ],
}
# 取第一天的第一个景点：
print(trip_plan["days"][0]["attractions"][0])   # 故宫
```

> 出现位置：项目一的行程 JSON、项目二的搜索结果、项目三的 NPC 状态，全部是嵌套 dict/list。

## 3. 条件与循环

```python
friendliness = 65

if friendliness >= 80:
    level = "挚友"
elif friendliness >= 60:
    level = "亲密"
else:
    level = "普通"

print(level)   # 亲密
```

注意 Python 用 **缩进**（4 个空格）表示代码块，没有大括号。缩进错了程序直接报错，这是新手最常见的坑。

```python
# range(n) 生成 0..n-1
for i in range(3):
    print(f"第{i+1}天")

# while 循环 + break
count = 0
while True:
    count += 1
    if count >= 3:
        break
```

## 4. 函数

```python
def calc_budget(days: int, level: str = "中等") -> int:
    """根据天数和消费水平估算预算（这行叫 docstring，是函数说明）"""
    per_day = {"经济": 300, "中等": 600, "豪华": 1500}
    return days * per_day.get(level, 600)

print(calc_budget(3))            # 1800（level 用默认值"中等"）
print(calc_budget(3, "豪华"))    # 4500
print(calc_budget(days=3, level="经济"))  # 900（关键字参数，可读性更好）
```

**类型注解**：`days: int` 表示参数应为整数，`-> int` 表示返回整数。Python 不强制检查，但它是文档、编辑器补全和 FastAPI 自动校验的基础。项目代码中每个函数都有注解，**看到冒号后面的类型不要慌，它只是"说明书"**。

## 5. 类（class）—— 三个项目的骨架

Agent、服务、管理器……项目里所有核心组件都是类。

```python
class NPC:
    """一个NPC角色"""

    def __init__(self, name: str, role: str):
        # __init__ 是构造函数，创建对象时自动执行
        # self 代表"这个对象自己"，每个方法的第一个参数都是它
        self.name = name
        self.role = role
        self.friendliness = 50   # 初始好感度

    def talk(self, message: str) -> str:
        return f"{self.name}({self.role})听到了：{message}"

    def add_friendliness(self, delta: float):
        self.friendliness += delta


npc = NPC("张三", "Python工程师")   # 创建实例，自动调用 __init__
print(npc.talk("你好"))             # 张三(Python工程师)听到了：你好
npc.add_friendliness(2.0)
print(npc.friendliness)             # 52.0
```

**继承**：子类拥有父类的一切，并可以扩展。HelloAgents 框架里 `SimpleAgent` 继承自 `Agent` 基类，就是这个机制。

```python
class Engineer(NPC):
    def talk(self, message: str) -> str:
        # 重写父类方法
        return f"{self.name}推了推眼镜：从技术上讲，{message}"

e = Engineer("张三", "Python工程师")
print(e.talk("这个需求做不了"))
```

## 6. 模块与导入

一个 `.py` 文件就是一个模块，一个带 `__init__.py` 的文件夹就是一个包。项目代码分散在几十个文件里，靠 `import` 互相引用：

```python
# 导入整个模块
import json

# 从模块导入指定内容（项目里最常见）
from pathlib import Path
from hello_agents import SimpleAgent, HelloAgentsLLM

# 导入本项目其他文件（app/services/llm_service.py）
from app.services.llm_service import get_llm
```

看项目代码时，**遇到不认识的名字，先看文件顶部的 import，就知道它来自哪里**。

## 7. 异常处理

网络请求、LLM 调用随时可能失败，项目里大量使用 try/except 兜底：

```python
import json

def parse_llm_response(text: str) -> dict:
    try:
        return json.loads(text)          # 尝试把字符串解析成字典
    except json.JSONDecodeError as e:    # 解析失败走这里
        print(f"JSON解析失败: {e}")
        return {}
```

> 出现位置：项目一解析 LLM 输出的行程 JSON、项目二解析规划结果，都靠这个模式——LLM 输出不总是合法 JSON，必须容错。

## 8. JSON —— Agent 与世界交换数据的语言

JSON 是一种文本格式，长得和 Python 字典几乎一样。前后端通信、LLM 结构化输出、API 响应全是 JSON。

```python
import json

data = {"city": "北京", "days": 3}

text = json.dumps(data, ensure_ascii=False)   # dict -> JSON字符串
print(text)                                    # {"city": "北京", "days": 3}

data2 = json.loads(text)                       # JSON字符串 -> dict
print(data2["city"])                           # 北京
```

**从 LLM 的啰嗦回复里抠出 JSON**（项目二的核心技巧）：LLM 经常在 JSON 前后加解释文字，需要先定位再解析：

```python
def extract_json(text: str):
    start = text.find("[")           # 找到第一个 [
    end = text.rfind("]") + 1        # 找到最后一个 ]
    if start == -1 or end == 0:
        return None
    return json.loads(text[start:end])   # 字符串切片后解析

reply = '好的，以下是任务列表：[{"title": "基本信息"}] 希望有帮助！'
print(extract_json(reply))   # [{'title': '基本信息'}]
```

## 9. 环境变量与 .env 文件

API 密钥绝不能写死在代码里（会泄露、会误传到 GitHub）。通行做法是放进 `.env` 文件，代码里读环境变量：

```
# .env 文件内容（key=value，每行一个）
LLM_API_KEY=sk-xxxxxxxx
LLM_BASE_URL=https://api.deepseek.com/v1
```

```python
import os
from dotenv import load_dotenv   # 来自 python-dotenv 包

load_dotenv()                    # 读取 .env 写入环境变量
api_key = os.getenv("LLM_API_KEY")
```

三个项目的 `.env.example` 就是模板：复制成 `.env` 再填入真实密钥。**`.env` 永远不要提交到 Git**（项目的 `.gitignore` 已把它排除）。

## 10. 异步编程 async/await（能看懂即可）

FastAPI 后端大量出现 `async def`。你现在只需要建立一个直觉：

- **同步函数**：`def f()` —— 调用后必须等它干完才能干别的
- **异步函数**：`async def f()` —— 遇到"等待"（网络请求、LLM 响应）时可以先去处理别的请求

```python
import asyncio

async def fetch_weather(city: str) -> str:
    await asyncio.sleep(1)        # 模拟1秒的网络等待，await = "等这件事，但别闲着"
    return f"{city}：晴"

async def main():
    # 并发执行两个请求，总耗时约1秒而不是2秒
    results = await asyncio.gather(fetch_weather("北京"), fetch_weather("上海"))
    print(results)

asyncio.run(main())
```

规则：`await` 只能出现在 `async def` 里；调用异步函数必须 `await` 它（或交给 `asyncio.run`）。看项目代码时遇到 `async`/`await`，理解为"这是个不阻塞服务器的函数"即可。

## 11. 装饰器（能认出来即可）

`@` 开头的一行叫装饰器，作用是"给下面的函数增加额外能力"。FastAPI 用它把函数注册成 API 接口：

```python
@app.post("/api/trip/plan")          # 意思：POST 请求 /api/trip/plan 时，执行下面这个函数
async def create_trip_plan(request):
    ...
```

你不需要会写装饰器，只需要知道：**`@app.get(...)`/`@app.post(...)` 下面的函数就是一个网络接口**。

## 12. 生成器与 yield（项目二会用到）

`yield` 让函数"产出一个值后暂停，下次再从暂停处继续"。项目二的流式推送（研究进度一条条发给浏览器）就靠它：

```python
def research_progress():
    yield "开始规划任务..."
    yield "正在搜索资料..."
    yield "生成报告完成"

for msg in research_progress():
    print(msg)        # 三条消息依次输出
```

---

## 自测清单

能独立完成以下练习，就可以进入下一篇：

1. 写一个函数 `count_attractions(plan: dict) -> int`，统计上文 `trip_plan` 结构中所有天的景点总数。
2. 写一个 `Agent` 类：构造时接收 `name` 和 `system_prompt`，有一个 `run(message)` 方法返回 `f"[{self.name}] 收到: {message}"`。再写一个子类 `WeatherAgent` 重写 `run`。
3. 给定字符串 `'结果如下：{"city": "北京", "temp": 25} 请查收'`，从中提取出字典并打印 `temp`。（提示：`find("{")` 和 `rfind("}")`）

## 推荐补充资源（可选）

- 官方入门教程中文版：https://docs.python.org/zh-cn/3/tutorial/
- 交互式练习：https://www.runoob.com/python3/python3-tutorial.html

下一篇：[02 · 环境搭建：uv 与开发工具链](./02-环境搭建-uv与开发工具链.md)
