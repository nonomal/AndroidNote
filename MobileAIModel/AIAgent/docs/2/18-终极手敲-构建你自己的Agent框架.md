# 18 · 终极手敲:构建你自己的 Agent 框架(原书第七章)

> 这是手敲系列的终点,对应原书**第七章《构建你的智能体框架》**。前面你敲的都是"应用"(用 LLM 做一个东西),这次你要造的是"框架"(让别人能用它做任何东西)——从零写出一个与 HelloAgents 同构的迷你框架 `myagents`,包含:多提供商 LLM 客户端、Message/Config 框架接口、Agent 抽象基类、工具注册表,以及**五种范式 Agent**(Simple / ReAct / Reflection / PlanAndSolve / FunctionCall),最后打包成一个真正可以 `pip install` 的 Python 库。
>
> 全部 15 个文件、约 600 行,每一行都在下面给出。预计 3~5 天。建议完成 11~14 篇后再学本篇。
>
> 学完后再去读 HelloAgents 官方源码和官方 `code/chapter7`,你会发现:官方第七章教的"继承框架基类做扩展",你已经能看懂它的每一层——因为**同构的东西你亲手造过一遍**。

## 0. 应用 vs 框架:本质区别是什么

你在第 11 篇写的 `MiniAgent` 是应用:工具写死在 `tools.py`、提示词写死在类里、只有一种行为模式。**框架的本质是"把变与不变分离"**:

- **不变的**(框架负责):消息怎么存、历史怎么截断、工具怎么注册和分发、LLM 怎么调、循环怎么兜底
- **变化的**(使用者负责):用什么模型、注册什么工具、选哪种范式、写什么提示词

衡量你是否造出了框架的唯一标准:**换一个完全不同的业务(客服/研究/游戏 NPC),一行框架代码都不用改,只写业务代码就能跑**。本篇最终验收就是这个。

## 1. 架构设计(先画图再动工)

```
myagents/                        ← Python包(框架本体)
├── __init__.py                  ← 对外API出口: from myagents import ...
├── config.py                    ← Config: 全局默认参数
├── message.py                   ← Message: 统一消息结构
├── llm.py                       ← LLM: 多提供商客户端(自动检测/流式/原生工具调用)
├── tools.py                     ← Tool + ToolRegistry: 工具系统
├── builtin_tools.py             ← 内置示例工具(计算器/时间)
└── agents/                      ← 五种范式,全部继承同一个基类
    ├── __init__.py
    ├── simple.py                ← SimpleAgent: 对话+文本协议工具
    ├── react.py                 ← ReActAgent: Thought→Action→Observation
    ├── reflection.py            ← ReflectionAgent: 初稿→评审→修改
    ├── plan_solve.py            ← PlanAndSolveAgent: 先计划后执行
    └── function_call.py         ← FunctionCallAgent: 原生工具调用
examples/demo.py                 ← 五种范式一次跑遍
pyproject.toml                   ← 打包配置(让框架可以pip安装)
.env                             ← 三件套
```

两条贯穿全篇的设计原则(面试必讲):

1. **依赖抽象,不依赖具体**:所有 Agent 只依赖"有 `invoke` 方法的 LLM"和"有 `execute` 方法的注册表",不关心背后是 DeepSeek 还是本地 Ollama、工具是计算器还是搜索——所以换模型/换工具不用改 Agent 代码
2. **基类收公共,子类只写差异**:记忆管理、配置、身份,写一次放基类;五种范式的差异**只在各自的 `run()` 方法里**——你会看到每个范式文件都出奇地短

## 2. 建项目与打包配置(30 分钟)

PyCharm New Project → `my-agent-framework`,虚拟环境同前(**必须选 Python 3.10 及以上**,02 篇装的 3.12 正好;若解释器是 3.9,下面安装会直接报版本错误)。按第 1 节的结构建好目录(`myagents`、`myagents/agents`、`examples` 三个文件夹)。

**关键一步**:在 `myagents/` 和 `myagents/agents/` 里各新建一个**空的** `__init__.py`(右键 → New → Python File,内容先留空,第 7、8 节再填)。`__init__.py` 是"这个文件夹是 Python 包"的身份证——**必须在安装前就创建**:如果先执行下面的 `pip install -e .` 再补 `__init__.py`,setuptools 会把包错误地登记成"命名空间包",之后 `from myagents import ...` 在项目目录之外会莫名失败,而且很难排查(重新执行一遍 `pip install -e .` 可修复)。

项目根目录新建 `pyproject.toml`——Python 的标准打包配置文件,有了它你的框架就能被 pip 安装:

```toml
[project]
name = "myagents"
version = "0.1.0"
description = "从零手敲的迷你Agent框架"
requires-python = ">=3.10"
dependencies = ["openai>=1.40.0", "python-dotenv>=1.0.0"]

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.setuptools]
packages = ["myagents", "myagents.agents"]
```

`.env` 三件套同前(第 11 篇)。然后在项目根目录执行**可编辑安装**:

```bash
pip install -e .
```

> `-e`(editable)= 把当前目录"链接"进环境而不是复制进去:改一行框架代码立刻生效,不用重装。这是所有框架/库开发者的日常工作方式——HelloAgents 本身就是这么开发的。

如果这一步报 editable 安装相关的错误,多半是 pip 版本太旧,先升级再装:`pip install -U pip`。

**✅ Checkpoint 0**:`pip list` 里出现 `myagents 0.1.0`。

## 3. 框架接口:message.py 与 config.py(30 分钟)

先写最底层的两块砖。`myagents/message.py`:

```python
"""统一的消息结构: 框架内所有对话记录的最小单元"""
from dataclasses import dataclass, field
from datetime import datetime
from typing import Literal

# Literal: 把role的取值限制在这四个字符串里,填错的IDE直接标红(第01篇类型注解的进阶用法)
Role = Literal["system", "user", "assistant", "tool"]


@dataclass
class Message:
    content: str
    role: Role
    timestamp: datetime = field(default_factory=datetime.now)

    def to_dict(self) -> dict:
        """转成OpenAI API要的格式(时间戳是框架内部字段,不发给模型)"""
        return {"role": self.role, "content": self.content}
```

`myagents/config.py`:

```python
"""框架全局配置: 默认参数集中在一处,而不是散落各文件写死"""
from dataclasses import dataclass


@dataclass
class Config:
    max_history: int = 20      # 记忆最多保留多少条消息
    temperature: float = 0.3
    timeout: int = 60          # LLM请求超时(秒)
    max_steps: int = 5         # 循环类Agent的步数保险丝
```

为什么值得为几个字段单独建文件?因为框架的使用者要能**一处改、处处生效**:`Config(max_history=100)` 传进任何 Agent 都立即生效——这就是"把变化的东西participate参数化"。

## 4. llm.py:多提供商 LLM 客户端(1.5 小时)

对应原书 7.2 节。比手敲应用里的 `llm.py` 强在三处:**自动检测提供商、流式接口、原生工具调用接口**。`myagents/llm.py`:

```python
"""统一的LLM客户端: 多提供商自动检测 + 普通/流式/原生工具调用三种invoke"""
import os

from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

# 提供商表: 名字 -> (base_url里的识别关键词, 默认base_url)
PROVIDERS = {
    "deepseek":   ("deepseek",   "https://api.deepseek.com/v1"),
    "zhipu":      ("bigmodel",   "https://open.bigmodel.cn/api/paas/v4/"),
    "modelscope": ("modelscope", "https://api-inference.modelscope.cn/v1/"),
    "ollama":     ("11434",      "http://localhost:11434/v1"),
    "vllm":       ("8001",       "http://localhost:8001/v1"),
}


class LLM:
    def __init__(self, model=None, api_key=None, base_url=None,
                 provider=None, temperature=0.3, timeout=60):
        if provider:  # 手动指定提供商 → 用它的默认地址
            if provider not in PROVIDERS:
                raise ValueError(f"未知provider: {provider},可选: {list(PROVIDERS)}")
            base_url = base_url or PROVIDERS[provider][1]
        base_url = base_url or os.getenv("LLM_BASE_URL")
        self.provider = provider or self._detect(base_url)
        self.model = model or os.getenv("LLM_MODEL_ID")
        api_key = api_key or os.getenv("LLM_API_KEY") or "local"  # 本地模型不校验key
        if not (base_url and self.model):
            raise ValueError("缺少配置: 请检查 .env 里的 LLM_MODEL_ID / LLM_BASE_URL")
        self.temperature = temperature
        self._client = OpenAI(api_key=api_key, base_url=base_url, timeout=timeout)
        print(f"✅ LLM就绪: provider={self.provider}, model={self.model}")

    @staticmethod
    def _detect(base_url: str) -> str:
        """自动检测: 从base_url里认出这是哪家(原书7.2.3的自动检测机制)"""
        for name, (keyword, _) in PROVIDERS.items():
            if keyword in (base_url or ""):
                return name
        return "custom"

    def invoke(self, messages: list[dict], temperature: float | None = None) -> str:
        """普通调用: 进一组messages,出一段文本"""
        resp = self._client.chat.completions.create(
            model=self.model, messages=messages,
            temperature=self.temperature if temperature is None else temperature,
        )
        return resp.choices[0].message.content or ""

    def stream_invoke(self, messages: list[dict], temperature: float | None = None):
        """流式调用: 一小段一小段yield出来(生成器,第04篇SSE的底层来源)"""
        stream = self._client.chat.completions.create(
            model=self.model, messages=messages, stream=True,
            temperature=self.temperature if temperature is None else temperature,
        )
        for chunk in stream:
            if chunk.choices and chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

    def invoke_with_tools(self, messages: list[dict], tools: list[dict]):
        """原生Function Calling: 返回完整message对象(可能带tool_calls字段,第10篇)"""
        resp = self._client.chat.completions.create(
            model=self.model, messages=messages, tools=tools,
            temperature=self.temperature,
        )
        return resp.choices[0].message
```

**✅ Checkpoint 1**:项目根目录新建临时文件 `t.py`:

```python
from myagents.llm import LLM
llm = LLM()
print(llm.invoke([{"role": "user", "content": "用一句话介绍你自己"}]))
```

运行后能看到 `provider=deepseek`(或你用的那家)被自动检测出来,并输出一句自我介绍。测完删掉 `t.py`。

## 5. tools.py:工具系统(1 小时)

对应原书 7.5 节。核心是两层抽象:`Tool`(一个工具 = 名字 + 说明书 + 函数)和 `ToolRegistry`(注册表 = 工具的电话簿 + 统一执行入口)。`myagents/tools.py`:

```python
"""工具系统: Tool基类 + ToolRegistry注册表"""


class Tool:
    """一个工具 = 名字 + 给模型看的说明书 + 真正干活的函数"""

    def __init__(self, name: str, description: str, func):
        self.name = name
        self.description = description
        self.func = func

    def run(self, tool_input: str) -> str:
        """统一入口: 出错不抛异常,把错误变成文本回给模型(第16篇坑6)"""
        try:
            return str(self.func(tool_input))
        except Exception as e:
            return f"工具执行出错: {e}"


class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool):
        self._tools[tool.name] = tool

    def unregister(self, name: str):
        self._tools.pop(name, None)

    def list_tools(self) -> list[str]:
        return list(self._tools)

    def describe(self) -> str:
        """生成给模型看的工具清单(拼进提示词)"""
        if not self._tools:
            return "暂无可用工具"
        return "\n".join(f"- {t.name}: {t.description}" for t in self._tools.values())

    def execute(self, name: str, tool_input: str) -> str:
        tool = self._tools.get(name)
        if tool is None:  # 模型幻觉出不存在的工具时,把可用清单回喂给它(第16篇坑4)
            return f"错误: 没有名为'{name}'的工具。可用工具: {self.list_tools()}"
        return tool.run(tool_input)
```

再写两个内置工具当"框架自带电池"。`myagents/builtin_tools.py`:

```python
"""内置示例工具: 计算器(ast安全求值,第11篇写过)和当前时间"""
import ast
import operator
from datetime import datetime

from .tools import Tool

_OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul,
        ast.Div: operator.truediv, ast.Pow: operator.pow, ast.USub: operator.neg}


def _safe_eval(node):
    if isinstance(node, ast.Expression):
        return _safe_eval(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.BinOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.left), _safe_eval(node.right))
    if isinstance(node, ast.UnaryOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.operand))
    raise ValueError("只支持数字四则运算")


def _calc(expression: str) -> str:
    return f"{expression} = {_safe_eval(ast.parse(expression.strip(), mode='eval'))}"


def _now(_: str = "") -> str:
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


calculator_tool = Tool("calculator", "计算数学表达式。参数: 算式字符串,如 (100-25)*0.8", _calc)
time_tool = Tool("time", "查询当前日期时间。参数留空即可", _now)
```

**✅ Checkpoint 2**:临时脚本验证注册表(测完删):

```python
from myagents.tools import ToolRegistry
from myagents.builtin_tools import calculator_tool, time_tool
r = ToolRegistry()
r.register(calculator_tool)
r.register(time_tool)
print(r.describe())
print(r.execute("calculator", "(100-25)*0.8"))   # → 60.0
print(r.execute("不存在的工具", ""))               # → 错误提示带可用清单
```

## 6. agent.py:抽象基类(1 小时,框架的脊椎)

对应原书 7.3.3。**ABC(抽象基类)**= Python 里的"合同":规定子类必须实现哪些方法,没实现就实例化报错。`myagents/agent.py`:

```python
"""Agent抽象基类: 所有范式共享的骨架(身份/LLM/记忆),差异全在run()里"""
from abc import ABC, abstractmethod
from typing import Optional

from .config import Config
from .message import Message


class Agent(ABC):
    def __init__(self, name: str, llm, system_prompt: Optional[str] = None,
                 config: Optional[Config] = None):
        self.name = name
        self.llm = llm
        self.system_prompt = system_prompt
        self.config = config or Config()
        self._history: list[Message] = []

    @abstractmethod
    def run(self, input_text: str, **kwargs) -> str:
        """子类必须实现: 每种范式的全部差异都集中在这个方法里"""

    # ---------- 公共记忆管理: 基类写一次,五种Agent全部继承 ----------
    def add_message(self, message: Message):
        self._history.append(message)
        if len(self._history) > self.config.max_history:   # 防止上下文无限膨胀(第16篇坑7)
            self._history = self._history[-self.config.max_history:]

    def history_dicts(self) -> list[dict]:
        return [m.to_dict() for m in self._history]

    def clear_history(self):
        self._history.clear()
```

注意 `llm` 参数没有写类型强制——**任何有 `invoke` 方法的对象都能当 LLM 用**(鸭子类型)。这是可测试性的关键:测试时塞一个假 LLM 进去,不花一分钱就能测完整个框架(第 17 篇的测试方法,本篇最后就这么验收)。

## 7. 五种范式 Agent(3 小时,本篇灵魂)

把第 2 节建的空 `myagents/agents/__init__.py` 填入一行包说明:

```python
"""范式Agent子包: 五种范式,同一个基类,差异只在run()"""
```

### 7.1 SimpleAgent:对话 + 文本协议工具

`myagents/agents/simple.py`(就是你第 11 篇 MiniAgent 的"框架化重构"):

```python
"""SimpleAgent: 基础对话 + 可选的文本协议工具调用"""
import re

from ..agent import Agent
from ..message import Message

TOOL_CALL_RE = re.compile(r"\[TOOL_CALL:([^:\]]+):([^\]]*)\]")


class SimpleAgent(Agent):
    def __init__(self, name, llm, system_prompt=None, config=None, tool_registry=None):
        super().__init__(name, llm, system_prompt, config)
        self.tools = tool_registry

    def _enhanced_prompt(self) -> str:
        """没有注册工具就是纯对话;注册了就自动把工具说明书拼进系统提示词"""
        base = self.system_prompt or "你是一个有用的AI助手。"
        if not self.tools or not self.tools.list_tools():
            return base
        return base + (
            "\n\n## 可用工具\n" + self.tools.describe() +
            "\n\n需要工具时,输出格式(一次可以多个): [TOOL_CALL:工具名:参数]\n"
            "例如: [TOOL_CALL:calculator:(100-25)*0.8]\n"
            "拿到工具结果后再给出最终回答;不需要工具就直接回答。"
        )

    def run(self, input_text: str, max_tool_iterations: int = 3, **kwargs) -> str:
        messages = [{"role": "system", "content": self._enhanced_prompt()}]
        messages += self.history_dicts()
        messages.append({"role": "user", "content": input_text})

        reply = ""
        for _ in range(max_tool_iterations):
            reply = self.llm.invoke(messages, **kwargs)
            calls = TOOL_CALL_RE.findall(reply)
            if not calls or not self.tools:
                break
            print(f"🔧 检测到{len(calls)}个工具调用")
            results = [f"[{n.strip()}] {self.tools.execute(n.strip(), a.strip())}"
                       for n, a in calls]
            messages.append({"role": "assistant", "content": reply})
            messages.append({"role": "user",
                             "content": "工具执行结果:\n" + "\n".join(results) +
                                        "\n请基于结果给出最终回答。"})
        else:  # 次数用尽还在调工具 → 强制收尾(死循环保险丝)
            reply = self.llm.invoke(messages, **kwargs)

        self.add_message(Message(input_text, "user"))
        self.add_message(Message(reply, "assistant"))
        return reply

    def stream_run(self, input_text: str, **kwargs):
        """流式版(不含工具,保持简单): 一边yield一边攒全文,结束后存记忆"""
        messages = [{"role": "system", "content": self.system_prompt or "你是一个有用的AI助手。"}]
        messages += self.history_dicts()
        messages.append({"role": "user", "content": input_text})
        full = ""
        for chunk in self.llm.stream_invoke(messages, **kwargs):
            full += chunk
            yield chunk
        self.add_message(Message(input_text, "user"))
        self.add_message(Message(full, "assistant"))
```

> `for...else` 是 Python 的冷门语法:循环**没有被 break 打断**时才执行 else——这里恰好表达"次数用尽仍未拿到最终答案"的兜底。

### 7.2 ReActAgent:边想边做

`myagents/agents/react.py`(第 11 篇你已实现过循环,这里的新东西是 **Thought/Action/Observation 的显式格式**——让推理过程可见、可调试):

```python
"""ReActAgent: Thought→Action→Observation循环(原书7.4.2)"""
import re

from ..agent import Agent
from ..message import Message

REACT_PROMPT = """你是一个会推理、会用工具的助手。

## 可用工具
{tools}

## 输出格式(每次只走一步,必须同时包含两行)
Thought: 你的思考
Action: 工具名[参数] 或 Finish[最终答案]

例如:
Thought: 需要先算出折扣价
Action: calculator[(100-25)*0.8]

## 问题
{question}

## 已执行的历史
{history}

现在输出你的下一步:"""

ACTION_RE = re.compile(r"Action:\s*(\w+)\[(.*)\]", re.S)


class ReActAgent(Agent):
    def __init__(self, name, llm, tool_registry, system_prompt=None,
                 config=None, max_steps=5):
        super().__init__(name, llm, system_prompt, config)
        self.tools = tool_registry
        self.max_steps = max_steps

    def run(self, input_text: str, **kwargs) -> str:
        history: list[str] = []
        for step in range(1, self.max_steps + 1):
            prompt = REACT_PROMPT.format(
                tools=self.tools.describe(),
                question=input_text,
                history="\n".join(history) or "(空)",
            )
            reply = self.llm.invoke([{"role": "user", "content": prompt}], **kwargs)

            match = ACTION_RE.search(reply)
            if not match:  # 模型没按格式来 → 把纠正指令写进历史,下一轮它会看到
                history.append("(上一步输出缺少Action行,请严格按格式输出)")
                continue

            action, arg = match.group(1), match.group(2).strip()
            print(f"🧠 第{step}步 Action: {action}[{arg[:40]}]")

            if action == "Finish":
                self.add_message(Message(input_text, "user"))
                self.add_message(Message(arg, "assistant"))
                return arg

            observation = self.tools.execute(action, arg)
            history.append(f"Action: {action}[{arg}]")
            history.append(f"Observation: {observation}")

        return "抱歉,我无法在限定步数内完成这个任务。"
```

### 7.3 ReflectionAgent:自己审自己

`myagents/agents/reflection.py`(原书 7.4.3;第 12 篇的"预算控制"练习就是它的特例):

```python
"""Reflection范式: 初稿→自我评审→修改,循环直到评审通过(原书7.4.3)"""
from ..agent import Agent
from ..message import Message

CRITIC_PROMPT = """你是一位严格的评审员。审查下面的回答是否准确、完整、清晰。
如果已经足够好,只输出四个字母: PASS
否则列出具体的改进意见(只给意见,不要重写答案)。

【任务】{task}
【当前回答】
{draft}"""

REVISE_PROMPT = """请根据评审意见修改回答,输出修改后的完整回答(不要解释改了什么)。

【任务】{task}
【你之前的回答】
{draft}
【评审意见】
{critique}"""


class ReflectionAgent(Agent):
    def __init__(self, name, llm, system_prompt=None, config=None, max_rounds=2):
        super().__init__(name, llm, system_prompt, config)
        self.max_rounds = max_rounds

    def run(self, input_text: str, **kwargs) -> str:
        system = self.system_prompt or "你是一个严谨的助手。"
        draft = self.llm.invoke([
            {"role": "system", "content": system},
            {"role": "user", "content": input_text},
        ], **kwargs)

        for i in range(1, self.max_rounds + 1):
            critique = self.llm.invoke([{
                "role": "user",
                "content": CRITIC_PROMPT.format(task=input_text, draft=draft),
            }], temperature=0)  # 评审要冷静,温度归零
            if critique.strip().upper().startswith("PASS"):
                print(f"🪞 第{i}轮评审: 通过")
                break
            print(f"🪞 第{i}轮评审: 有意见,修改中")
            draft = self.llm.invoke([{
                "role": "user",
                "content": REVISE_PROMPT.format(task=input_text, draft=draft, critique=critique),
            }], **kwargs)

        self.add_message(Message(input_text, "user"))
        self.add_message(Message(draft, "assistant"))
        return draft
```

### 7.4 PlanAndSolveAgent:先谋后动

`myagents/agents/plan_solve.py`(原书 7.4.4;第 13 篇整个项目就是它的放大版):

```python
"""Plan-and-Solve范式: 先产出JSON计划,逐步执行,最后汇总(原书7.4.4)"""
import json

from ..agent import Agent
from ..message import Message

PLAN_PROMPT = """把下面的任务拆解成2~5个可顺序执行的步骤。
只输出JSON: {{"steps": ["步骤1", "步骤2"]}}

任务: {task}"""

STEP_PROMPT = """你正在执行一个多步骤任务。
【总任务】{task}
【已完成步骤的结果】
{context}
【当前步骤】{step}
请只完成当前步骤,输出这一步的结果。"""

FINAL_PROMPT = """【总任务】{task}
【各步骤执行结果】
{context}
请整合以上结果,给出最终完整回答。"""


def _extract_json(text: str) -> dict:
    start, end = text.find("{"), text.rfind("}")
    if start == -1 or end == -1:
        raise ValueError(f"输出中没有JSON: {text[:100]}")
    return json.loads(text[start:end + 1])


class PlanAndSolveAgent(Agent):
    def run(self, input_text: str, **kwargs) -> str:
        plan = _extract_json(self.llm.invoke([{
            "role": "user", "content": PLAN_PROMPT.format(task=input_text),
        }], temperature=0))
        steps = plan.get("steps", [])
        if not (1 <= len(steps) <= 8):  # 防御: 计划拆得离谱直接报错(第13篇同款)
            raise ValueError(f"计划异常: {steps}")
        print(f"📋 计划({len(steps)}步): {steps}")

        context = ""
        for i, step in enumerate(steps, 1):
            result = self.llm.invoke([{
                "role": "user",
                "content": STEP_PROMPT.format(task=input_text,
                                              context=context or "(无)", step=step),
            }], **kwargs)
            context += f"\n步骤{i}({step}): {result}"
            print(f"  ✓ 步骤{i}/{len(steps)}完成")

        final = self.llm.invoke([{
            "role": "user", "content": FINAL_PROMPT.format(task=input_text, context=context),
        }], **kwargs)
        self.add_message(Message(input_text, "user"))
        self.add_message(Message(final, "assistant"))
        return final
```

> 注意 PLAN_PROMPT 里的 `{{"steps": ...}}`:这段字符串要经过 `.format()`,而 format 把 `{}` 当占位符——**双写花括号 `{{ }}` 表示字面意义的花括号**。忘了双写会报 KeyError,这是提示词模板最常见的坑。

### 7.5 FunctionCallAgent:原生工具调用

`myagents/agents/function_call.py`(原书 7.4.5;第 10 篇讲过"文本协议 vs 原生 Function Calling"的概念,现在两种你都亲手实现了):

```python
"""FunctionCall范式: 用模型原生的tool_calls能力(对比SimpleAgent的文本协议)"""
import json

from ..agent import Agent
from ..message import Message


class FunctionCallAgent(Agent):
    def __init__(self, name, llm, system_prompt=None, config=None):
        super().__init__(name, llm, system_prompt, config)
        self._schemas: list[dict] = []   # 给模型看的函数说明(JSON Schema)
        self._funcs: dict = {}           # 真正执行的Python函数

    def add_function(self, name: str, description: str, parameters: dict, func):
        """parameters 是 JSON Schema(第10篇): 机器可读的参数说明书"""
        self._schemas.append({"type": "function", "function": {
            "name": name, "description": description, "parameters": parameters,
        }})
        self._funcs[name] = func

    def run(self, input_text: str, max_steps: int = 5, **kwargs) -> str:
        messages = [{"role": "system", "content": self.system_prompt or "你是一个有用的助手。"}]
        messages += self.history_dicts()
        messages.append({"role": "user", "content": input_text})

        for _ in range(max_steps):
            msg = self.llm.invoke_with_tools(messages, self._schemas)

            if not msg.tool_calls:  # 没有工具调用 = 最终回答
                reply = msg.content or ""
                self.add_message(Message(input_text, "user"))
                self.add_message(Message(reply, "assistant"))
                return reply

            # 有工具调用: 原样把assistant消息放回去,再逐个执行并以tool角色回填
            messages.append({"role": "assistant", "content": msg.content,
                             "tool_calls": [tc.model_dump() for tc in msg.tool_calls]})
            for tc in msg.tool_calls:
                args = json.loads(tc.function.arguments or "{}")
                func = self._funcs.get(tc.function.name)
                result = str(func(**args)) if func else f"未知函数 {tc.function.name}"
                print(f"🔧 原生调用 {tc.function.name}({args}) → {result[:50]}")
                messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})

        return "抱歉,超过最大步数仍未完成。"
```

对比一下你就懂了为什么原生方式更可靠:**参数是模型按 JSON Schema 生成的结构化数据,不需要正则去猜**;代价是依赖模型/提供商支持 `tools` 参数(本地小模型未必支持,所以文本协议依然有存在价值——这正是 HelloAgents 默认用文本协议的原因)。

## 8. 出口与总演示(1 小时)

把第 2 节建的空 `myagents/__init__.py` 填上内容——它是框架的"门面",使用者只需要 `from myagents import ...`:

```python
"""myagents: 你亲手写的Agent框架"""
from .config import Config
from .message import Message
from .llm import LLM
from .tools import Tool, ToolRegistry
from .builtin_tools import calculator_tool, time_tool
from .agents.simple import SimpleAgent
from .agents.react import ReActAgent
from .agents.reflection import ReflectionAgent
from .agents.plan_solve import PlanAndSolveAgent
from .agents.function_call import FunctionCallAgent

__version__ = "0.1.0"
```

`examples/demo.py`——五种范式一次跑遍:

```python
"""五种范式一次跑遍(需要真实API Key)。在项目根目录运行: python examples/demo.py"""
from myagents import (LLM, SimpleAgent, ReActAgent, ReflectionAgent,
                      PlanAndSolveAgent, FunctionCallAgent, ToolRegistry,
                      calculator_tool, time_tool)

llm = LLM()
registry = ToolRegistry()
registry.register(calculator_tool)
registry.register(time_tool)

print("\n===== 1. SimpleAgent(文本协议工具) =====")
simple = SimpleAgent("小助", llm, tool_registry=registry)
print(simple.run("(100-25)*0.8 等于多少?"))
print(simple.run("我上一个问题问的是什么?"))       # 验证基类的记忆管理

print("\n===== 2. ReActAgent =====")
react = ReActAgent("推理者", llm, registry)
print(react.run("现在几点了?另外 123*456 是多少?"))

print("\n===== 3. ReflectionAgent =====")
reflect = ReflectionAgent("评审型作者", llm)
print(reflect.run("用两句话向小学生解释什么是黑洞"))

print("\n===== 4. PlanAndSolveAgent =====")
planner = PlanAndSolveAgent("规划者", llm)
print(planner.run("为30人的周末公司团建设计一个方案,包含地点类型、日程和人均预算"))

print("\n===== 5. FunctionCallAgent(原生工具调用) =====")
fc = FunctionCallAgent("原生派", llm)
fc.add_function(
    "get_weather", "查询城市天气",
    {"type": "object",
     "properties": {"city": {"type": "string", "description": "城市名"}},
     "required": ["city"]},
    lambda city: f"{city}今天晴,25度",
)
print(fc.run("北京天气怎么样?"))
```

**✅ 最终验收**:

- [ ] `python examples/demo.py` 五个部分全部正常输出
- [ ] SimpleAgent 第二问能答出第一问的内容(基类记忆生效)
- [ ] ReActAgent 打印出多步 `🧠 Action:`,并且时间和乘积都正确
- [ ] ReflectionAgent 打印出 `🪞 评审` 过程
- [ ] PlanAndSolveAgent 打印出 `📋 计划` 和逐步 `✓`
- [ ] FunctionCallAgent 打印 `🔧 原生调用 get_weather({'city': ...})`(如果你的提供商不支持 tools 参数会报错——这本身就验证了第 7.5 节说的兼容性差异,换 DeepSeek 官方 API 再试)
- [ ] **框架终极测试**:新开一个文件夹,写一个 10 行的新业务(比如注册一个"单位换算"工具 + SimpleAgent),`from myagents import ...` 直接可用,一行框架代码都不用改

## 9. 对照官方:你造的和 HelloAgents 的差距在哪

现在去读官方仓库,你会发现结构一一对应:

| 你的 myagents | HelloAgents 官方 | 官方多做了什么 |
| --- | --- | --- |
| `LLM` + PROVIDERS 表 | `HelloAgentsLLM` | 更多提供商、重试、token统计 |
| `Message` / `Config` | `Message` / `Config` | 更多字段与序列化 |
| `Agent` 基类 | `Agent` 基类 | 生命周期钩子、回调 |
| `ToolRegistry` | `ToolRegistry` | 参数schema、异步工具、MCP接入(第03篇) |
| 五个范式 Agent | 同名五个 | 更完善的解析容错和流式支持 |

原书第七章配套代码(`code/chapter7`)教的是**继承官方基类做扩展**(`class MySimpleAgent(SimpleAgent)`),建议现在去把它通读一遍:`my_llm.py` 如何在子类里接管 modelscope 分支、`my_react_agent.py` 如何换掉提示词模板——你已经具备逐行看懂它们的能力,这就是本篇的意义。

**进阶练习**:

1. 给 `LLM.invoke` 加超时重试(第 10 篇 `tenacity`),让五种 Agent 免费获得稳定性——体会"框架改一处,所有应用受益"
2. 给 `ToolRegistry` 加一个 `@registry.tool(description=...)` 装饰器注册方式(第 01 篇装饰器),对齐主流框架的开发体验
3. 用第 17 篇的方法给五种范式各写一个 pytest(假 LLM 按剧本回话),凑一个 `tests/` 目录——从此你敢重构框架
4. 把你第 11 篇的 Mini Agent 后端改成 `from myagents import SimpleAgent` 实现,体会"应用瘦身"

## 10. 面试怎么讲(这是简历上最硬的一条)

"我从零实现过一个 Agent 框架:抽象基类统一记忆和配置管理,五种范式(Simple/ReAct/Reflection/PlanAndSolve/FunctionCall)只重写 run 方法;LLM 层做了多提供商自动检测,同一套 Agent 代码无缝切换云端 API 和本地 vLLM;工具系统用注册表模式,文本协议和原生 Function Calling 两种调用方式都实现过,清楚各自的适用边界;框架用 pyproject 打包,可 pip 安装,配了基于假 LLM 的单元测试。"——这段话的每个词你都有代码背书,面试官往任何方向追问你都接得住。

下一篇:[19 · 实战手敲五:企业级文档问答系统](./19-实战手敲五-企业级文档问答系统.md) | 返回目录:[README](./README.md)
