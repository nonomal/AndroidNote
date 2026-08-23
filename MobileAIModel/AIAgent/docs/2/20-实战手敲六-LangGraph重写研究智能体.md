# 20 · 实战手敲六:用 LangGraph 重写研究智能体

> 手敲系列第六篇,也是**唯一一篇"重写"**:把 13 篇那个能跑的深度研究智能体,用 LangGraph 重写成一个能在生产里活下来的状态机。
>
> 全部 15 个文件、约 1800 行,每一行都在下面给出。预计 3~4 天。
>
> 核心新技能:**状态机建模(节点/边/reducer)**、**Postgres checkpointer 与断点续跑**、**`interrupt()` 人在环中**、**时间旅行与分叉重跑**、**给状态机写测试**。

## 0. 为什么值得把一个"已经能跑"的项目重写一遍

13 篇的 `run_research()` 是这样的:

```python
def run_research(question):
    tasks = plan(question)                      # 规划
    with ThreadPoolExecutor(max_workers=3) as ex:   # 并行研究
        futures = {ex.submit(research, t): i for i, t in enumerate(tasks)}
        findings = [None] * len(tasks)
        for f in as_completed(futures):
            findings[futures[f]] = f.result()
    report = write(question, findings)          # 撰写
    return report
```

这段代码**没有任何问题**——只要它一路顺利跑完。现在给它加四个真实需求:

| 需求 | 13 篇要怎么改 |
| --- | --- |
| 计划要人过一眼再花钱搜索 | 函数得拆成两半,中间的状态得自己存起来,还得设计一套"任务 id → 半成品状态"的表 |
| 撰写报告时进程被 OOM 杀了,重启别重搜一遍 | 每一步的中间结果都得自己序列化落库,还得自己判断"从哪一步接着跑" |
| 老板问"如果当时只研究第一个方向会怎样" | 只能整个重跑一遍,三次搜索三次 LLM 全部重付费 |
| 报告质量不行要自动重写,但别无限重写 | 自己写循环 + 自己数轮数 |

四件事全都能自己写。写完之后你会发现,你写的东西有个名字,叫**工作流引擎**。LangGraph 就是那个已经写好的工作流引擎,而且它是**为 LLM 应用**写的:状态是可序列化的字典、暂停恢复是一等公民、并行合并有明确语义。

所以这一篇的重点不是"学一个新框架的 API",而是**换一种思考方式**:

> 从"我的程序按顺序调用了哪些函数",换成"我的系统有哪些状态,以及从每个状态出发能去哪些状态"。

这个转变的收益,一句话概括:**你的 agent 从一个函数变成一个可以被暂停、检查、修改、重放的对象。**

## 1. 成品长什么样

四条命令,四种 13 篇里没有的能力。

**(1) 起一个研究,它自己停下来等人批**

```bash
$ python -m rg.cli start "2025 年具身智能的进展" --thread t1
thread_id = t1
⏸  等待人工审批(第 1 版计划)
   1. 2025 年具身智能的进展的主要参与者与代表性方案
   2. 2025 年具身智能的进展的关键技术路线
   3. 2025 年具身智能的进展的落地案例与商业化情况
   提示: 回复 {'action':'approve'} / {'action':'edit','tasks':[...]} / {'action':'reject','comment':'...'}
   下一个待执行节点: ('approve',)
```

注意这个进程**已经退出了**。计划躺在 Postgres 里。

**(2) 换一个进程(甚至换一台机器)接着跑**

```bash
$ python -m rg.cli resume t1 approve
✅ status=done
   发现数=3 评审轮数=2
   评审={'score': 9, 'issues': []}
```

**(3) 让撰写节点崩掉,然后续跑——三次搜索不重做**

```bash
$ RG_CRASH_AT=write python -m rg.cli resume t2 approve
RuntimeError: [注入故障] 节点 write 崩了

$ python -m rg.cli status t2
next       : ('write',)
findings   : 3 条        ← 研究成果已经落库了

$ python -m rg.cli continue t2
从 ('write',) 继续(已有 3 条发现, 不会重跑)
✅ status=done
```

**(4) 回到历史上的某一步,改掉当时的计划,只重跑后面**

```bash
$ python -m rg.cli fork t1 2 --set-tasks "只看人形机器人这一条线"
回到 step=2, 当时 next=('research', 'research', 'research')
已在该点写入新计划(1 个子任务), 分支 checkpoint 已建立
✅ status=done
   发现数=1
```

原来那条线一个字都没被改掉——一个 thread 里现在有两条分支,都能读。

这四件事对应 LangGraph 的四个卖点:**中断/恢复、持久化、并行合并、时间旅行**。下面每一件都会亲手跑一遍。

## 2. 心智模型:状态、节点、边

先把三个概念钉死,后面所有代码都是这三个概念的排列组合。

**状态(State)** 是一个 TypedDict。它是这张图的**全部记忆**,也是被序列化进 Postgres 的那个东西。

**节点(Node)** 是一个函数 `(state) -> dict`。关键纪律:

> **节点只返回"要改的字段",不返回整个 state。**

返回全量会把别人并行写进去的字段冲掉。这不是风格问题,是正确性问题。

**边(Edge)** 决定下一步去哪。三种:

- `add_edge("a", "b")` —— 固定跳转
- `add_conditional_edges("a", fn, [...])` —— `fn(state)` 返回目标节点名
- `fn` 返回 `[Send("b", payload), ...]` —— **运行时扇出**:发几份、每份给什么参数,跑到这一步才知道

还有一件事必须现在就理解,否则后面会踩:

> **同一个字段被同一步里的多个节点写入时,默认是"只允许一个写入者",第二个写入直接抛错。** 要允许多个写入者,必须给这个字段声明 reducer。

这就是第 8 节那个 30 行文件要证明的事。

## 3. 建项目与起 Postgres(20 分钟)

```bash
mkdir research_graph && cd research_graph
uv init --python 3.12
uv add "langgraph>=1.2" "langgraph-checkpoint-postgres>=3.1" \
       "psycopg[binary,pool]>=3.2" "fastapi>=0.115" "uvicorn[standard]>=0.32" \
       "openai>=1.40" ddgs "redis>=5.0" python-dotenv pytest grandalf
mkdir -p rg tests static
touch rg/__init__.py tests/__init__.py
```

`grandalf` 是画图用的,只在 `draw_ascii()` 时需要——没有它会报 `ImportError: Install grandalf to draw graphs`,而这句提示不会告诉你该装哪个包的哪个版本,所以现在就装上。

Postgres 用 Docker 起:

```bash
docker run -d --name rg-pg -p 5432:5432 \
  -e POSTGRES_USER=rg -e POSTGRES_PASSWORD=rg -e POSTGRES_DB=rg postgres:17-alpine
docker run -d --name rg-redis -p 6379:6379 redis:7-alpine
```

**✅ Checkpoint 1**:两个容器都在:

```
$ docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
rg-pg     postgres:17-alpine   Up 10 seconds
rg-redis  redis:7-alpine       Up 8 seconds
```

`pyproject.toml` 补上两段配置:

```toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]
# E501 对中文注释不适用: 中文按 1 字符计数但显示占 2 列
ignore = ["E501"]

# 项目没写 build-system(不打包成库), pytest 也不会把 rootdir 加进 sys.path,
# 少了这一段 `uv run pytest` 会在 import rg 时报 ModuleNotFoundError
[tool.pytest.ini_options]
pythonpath = ["."]
testpaths = ["tests"]
# 测试函数名用了中文, 默认的 python_functions = "test*" 已经能匹配, 这里显式钉住
python_functions = ["test_*"]
```

## 4. rg/config.py —— 缺配置就报错(10 分钟)

```python
"""集中式配置。所有开关都从环境变量读, 没有硬编码的默认密钥。

设计原则: 缺配置就报错, 不"悄悄用个默认值继续跑"。
线上事故里最常见的一类就是"以为读到了配置, 其实用的是兜底默认值"。
"""

from __future__ import annotations

import os

# 两种运行模式, 决定 llm.py / search.py 走真实调用还是本地替身:
#   fake    —— 脚本化假模型 + 本地搜索样例, 完全离线、可复现, 测试和教程用
#   real    —— 真实 LLM API + 真实联网搜索
# 默认 fake: 教程里任何一条命令跑起来都不该产生费用
MODE = os.getenv("RG_MODE", "fake")

LLM_BASE_URL = os.getenv("LLM_BASE_URL", "https://api.deepseek.com/v1")
LLM_MODEL = os.getenv("LLM_MODEL", "deepseek-chat")

# checkpointer 用的 Postgres。给了默认值是因为它不是密钥,
# 而且教程里的 compose 就是这个地址 —— 密钥类配置绝不给默认值
PG_DSN = os.getenv("RG_PG_DSN", "postgresql://rg:rg@127.0.0.1:5432/rg")

# 报告评审的通过线与最大重写轮数
REVIEW_PASS = int(os.getenv("RG_REVIEW_PASS", "7"))
MAX_ROUNDS = int(os.getenv("RG_MAX_ROUNDS", "2"))


def llm_api_key() -> str:
    key = os.getenv("LLM_API_KEY", "")
    if not key:
        raise RuntimeError("未配置 LLM_API_KEY(RG_MODE=real 时必需)")
    return key


def service_api_keys() -> set[str]:
    """HTTP 服务自己的访问密钥, 逗号分隔。空集合意味着服务必须拒绝启动。"""
    raw = os.getenv("RG_API_KEYS", "")
    return {k.strip() for k in raw.split(",") if k.strip()}
```

`MODE` 默认 `fake` 这个决定值得多说一句。这一篇要验证的是**图的控制流**——分支走对了吗、并行合上了吗、崩了能续吗。这些和模型聪不聪明毫无关系,而真模型带来的随机性会让"这次为什么走了另一条路"变成一个无法回答的问题。**默认离线可复现,是这一篇所有 Checkpoint 都能真跑的前提。**

## 5. rg/state.py —— 整个项目最该先想清楚的文件(30 分钟)

```python
"""图的状态定义 —— 这是整个项目最该先想清楚的文件。

LangGraph 的心智模型: **节点是纯函数, 只返回"要更新哪些字段"**, 框架负责合并。
所以状态的每个字段都要回答一个问题: 多个节点同时写它时怎么合并?

- 默认行为是"覆盖"(后写的赢)
- 用 Annotated[..., reducer] 指定别的合并方式

`findings` 必须用 reducer: 三个研究员节点**并行**跑完, 各自返回一条发现。
不加 reducer 的字段一步之内只允许一个写入者, 三个并行节点同时写它会直接抛
InvalidUpdateError。跑 `python -m rg.why_reducer` 能看到两种写法的实测对照 ——
这是所有人写 LangGraph 并行时第一次都会撞上的错误。
"""

from __future__ import annotations

import operator
from typing import Annotated, TypedDict


class Finding(TypedDict):
    index: int
    task: str
    summary: str
    sources: list[str]


class ResearchState(TypedDict, total=False):
    # 输入
    question: str

    # 规划阶段
    tasks: list[str]
    plan_round: int  # 重新规划了几次, 防止"改计划-再改计划"无限循环

    # 人工审批的结果, 保留下来是为了事后审计: 谁在什么时候批了什么
    approval: dict

    # 并行研究的产出。operator.add 让三个并行节点的返回值拼接而不是互相覆盖
    findings: Annotated[list[Finding], operator.add]

    # 撰写与评审
    report: str
    review: dict
    review_round: int

    # 终态标记, 给 HTTP 层看
    status: str
```

`total=False` 是刻意的:图刚启动时状态里只有 `question`,别的字段都还不存在。所以节点里读状态要用 `state.get("x", 默认值)`,不要 `state["x"]` ——除了那些你能确定一定已经被前面节点写过的字段。

`Finding` 里那个 `index` 是**必需的**,不是装饰。并行节点的完成顺序是不确定的,`findings` 里的顺序也就是不确定的;报告要按计划顺序写,只能靠 `index` 排回来。这一点在第 7 节的 `write` 节点里会再出现一次。

## 6. rg/llm.py 与 rg/search.py —— 一个能被断言的替身(40 分钟)

`FakeLLM` 不是为了省钱(虽然确实省),它是为了让"图走了哪条路"这件事**可断言**。

```python
"""LLM 客户端 + 一个脚本化的假模型。

为什么要有假模型: 这一篇的重点是**图的控制流**(分支、循环、中断、恢复、回放),
不是模型有多聪明。用假模型有三个好处:
  1. 教程里每个 Checkpoint 都能真跑, 输出可复现, 且不花钱
  2. 测试能断言"图走了哪条路", 不受模型随机性干扰
  3. 出错时你能确定问题在图里, 而不是在模型抽风

生产代码里也应该留这样一个替身 —— 它是 CI 能跑 agent 测试的前提。
"""

from __future__ import annotations

import json
import re
from typing import Any

from rg import config


class FakeLLM:
    """按 system prompt 里的角色标记返回脚本化结果。

    calls 记录每次调用, 测试里可以断言"规划师被调了几次""温度是不是 0"。

    一个刻意的设计: 评审的分数**只看报告内容**, 不看"这是第几次评审"。
    我第一版用了 self.review_round 计数器, 结果服务里同时跑两个 thread 时
    它们共享同一个 FakeLLM 实例, 第二个 thread 的第一次评审直接拿到高分 ——
    替身也是有状态的, 而**跨请求共享的状态就是 bug 的温床**。
    改成看内容之后, 每个 thread 的行为都独立且可复现。
    """

    def __init__(self) -> None:
        self.calls: list[dict] = []

    def chat(self, messages: list[dict], temperature: float = 0.0) -> str:
        system = messages[0]["content"]
        user = messages[-1]["content"]
        self.calls.append({"role_hint": system[:12], "temperature": temperature})

        if "研究规划师" in system:
            return json.dumps(
                {
                    "tasks": [
                        f"{_topic(user)}的主要参与者与代表性方案",
                        f"{_topic(user)}的关键技术路线",
                        f"{_topic(user)}的落地案例与商业化情况",
                    ]
                },
                ensure_ascii=False,
            )

        if "研究员" in system:
            task = _first_line(user)
            return (
                f"关于「{task}」, 现有材料显示: 该方向在 2025 年出现了三个明确趋势, "
                f"其中最主要的是规模化部署带来的成本下降(约 40%)。[1][2]"
            )

        if "报告撰写人" in system:
            n = user.count("### 子任务:")
            body = (
                f"# {_topic(user)}研究报告\n\n"
                f"## 摘要\n\n本报告基于 {n} 个子方向的检索材料, 给出现状与结论。\n\n"
                f"## 现状\n\n规模化部署使成本下降约 40%。[1][2]\n\n"
            )
            # 收到评审意见时才补上这一节 —— 让"重写确实改了东西"可被观察
            if "评审意见" in user:
                body += "## 数据来源与时间范围\n\n材料均来自 2025 年公开报告。[1][2][3]\n\n"
            return body + "## 结论\n\n该方向已进入工程落地阶段, 建议按场景分批推进。"

        if "报告评审" in system:
            if "数据来源与时间范围" in user:
                return json.dumps({"score": 9, "issues": []}, ensure_ascii=False)
            return json.dumps(
                {"score": 5, "issues": ["缺少数据来源的时间范围", "结论部分过短"]},
                ensure_ascii=False,
            )

        raise ValueError(f"FakeLLM 不认识这个角色: {system[:30]}")
```

**最后那个 `raise` 很重要。** 替身遇到没见过的角色时应该炸,不应该返回一句"好的"——否则你新加了一个节点、忘了给替身写脚本,测试还是绿的,而那个节点收到的是一句废话。

真实客户端和单例:

```python
class RealLLM:
    """真实 API。只在 RG_MODE=real 时构造 —— 构造即校验 key。"""

    def __init__(self) -> None:
        from openai import OpenAI

        self.client = OpenAI(api_key=config.llm_api_key(), base_url=config.LLM_BASE_URL)

    def chat(self, messages: list[dict], temperature: float = 0.0) -> str:
        resp = self.client.chat.completions.create(
            model=config.LLM_MODEL,
            messages=messages,
            temperature=temperature,
        )
        return resp.choices[0].message.content or ""


_INSTANCE: Any = None


def get_llm() -> Any:
    """单例。图里每个节点都要用, 不要每次新建客户端(会重建连接池)。"""
    global _INSTANCE
    if _INSTANCE is None:
        _INSTANCE = FakeLLM() if config.MODE == "fake" else RealLLM()
    return _INSTANCE


def reset_llm(instance: Any = None) -> None:
    """测试用: 换掉单例。生产代码不会调它, 但留这个口子比让测试去改私有变量干净。"""
    global _INSTANCE
    _INSTANCE = instance


def chat(messages: list[dict], temperature: float = 0.0) -> str:
    return get_llm().chat(messages, temperature=temperature)


def chat_json(messages: list[dict], temperature: float = 0.0) -> dict:
    """要求模型输出 JSON, 容忍它套上 ```json 围栏或前后带废话。

    这个函数在前面几篇里出现过多次 —— 因为**它是所有"结构化输出"的地基**,
    而模型不遵守格式是常态, 不是异常。
    """
    raw = chat(messages, temperature=temperature)
    text = raw.strip()
    fence = re.search(r"```(?:json)?\s*(.+?)\s*```", text, re.S)
    if fence:
        text = fence.group(1)
    else:
        # 没有围栏就抓第一个平衡的花括号块
        start = text.find("{")
        end = text.rfind("}")
        if start != -1 and end > start:
            text = text[start : end + 1]
    try:
        return json.loads(text)
    except json.JSONDecodeError as e:
        raise ValueError(f"模型没有返回合法 JSON: {raw[:200]}") from e


def _topic(user_text: str) -> str:
    """从 user 消息里抠出研究主题, 只为让假输出看起来像真的。"""
    m = re.search(r"研究问题[::]\s*(.+)", user_text)
    if m:
        return m.group(1).strip().splitlines()[0]
    return user_text.strip().splitlines()[0][:20]


def _first_line(user_text: str) -> str:
    m = re.search(r"子任务[::]\s*(.+)", user_text)
    return m.group(1).strip() if m else user_text.strip().splitlines()[0]
```

`search.py` 和 13 篇几乎一样,只多了 fake 分支和一条纪律:

```python
def web_search(query: str, max_results: int = 5) -> list[dict]:
    if config.MODE == "fake":
        return FAKE_RESULTS[:max_results]
    try:
        from ddgs import DDGS

        with DDGS() as d:
            raw = list(d.text(query, max_results=max_results))
        return [
            {"title": r.get("title", ""), "url": r.get("href", ""), "snippet": r.get("body", "")}
            for r in raw
        ]
    except Exception as e:
        # 打日志但不抛: 外部依赖挂掉不该让整个研究失败
        print(f"[search] 搜索失败, 降级处理: {type(e).__name__}: {e}")
        return []


def format_results(results: list[dict]) -> str:
    if not results:
        return "(本次搜索无结果。请基于已有知识回答, 并在开头注明: 信息可能不是最新。)"
    return "\n\n".join(
        f"[{i}] {r['title']}\n{r['snippet']}\n来源: {r['url']}" for i, r in enumerate(results, 1)
    )
```

搜索失败返回空列表而不是抛异常,是因为**搜索层不该替业务决定"搜不到要不要继续"**。但空列表不能悄悄变成"我什么都不知道",所以 `format_results` 给了一句明确的降级提示,让模型知道自己在没有材料的情况下作答。

**✅ Checkpoint 2**:两个模块各自跑一下,确认替身是可复现的。

```
$ python -m rg.search
MODE=fake, 拿到 3 条
[1] 2025 年度技术趋势报告
报告指出规模化部署使单位成本下降约 40%, 主要来自推理侧优化。
来源: https://example.com/trends-2025
...

$ python -m rg.llm
MODE=fake
规划: {
  "tasks": [
    "2025 年具身智能的进展的主要参与者与代表性方案",
    "2025 年具身智能的进展的关键技术路线",
    "2025 年具身智能的进展的落地案例与商业化情况"
  ]
}
评审(初版报告): {"score": 5, "issues": ["缺少数据来源的时间范围", "结论部分过短"]}
评审(改过的报告): {"score": 9, "issues": []}
→ 评审只看内容, 不看轮数; 所以并发跑多个 thread 时互不干扰
```

## 7. rg/graph.py —— 这一篇的心脏(2.5 小时)

文件头先把这一篇的立意写清楚:

```python
"""图的定义: 节点 + 边 + 条件跳转。

和 docs/2/13 的 `run_research()` 对比着看 —— 那边是一个函数从上到下调三个 agent,
这边是把每一步变成节点、把每个"接下来去哪"变成边。换来的是四件在 13 篇里
写不出来(或者要自己造一遍)的能力:

  1. **断点续跑**: 每个节点跑完自动存 checkpoint, 进程挂了从下一个节点继续
  2. **人在环中**: 节点里一行 interrupt() 就能暂停整张图等人批准
  3. **回放**: 任意一次执行的每一步状态都留着, 可以从中间某步改一改重跑
  4. **并行的正确合并**: reducer 保证三个研究员的结果不会互相覆盖

节点写法上有两条纪律:
  - 节点只返回"要改的字段", 不返回整个 state(返回全量会把并行结果冲掉)
  - 节点必须可重跑: 恢复执行时同一个节点可能被执行两次, 不能有累加副作用
"""

from __future__ import annotations

import os
import sys
from typing import Literal, TypedDict

from langgraph.graph import END, START, StateGraph
from langgraph.types import Command, Send, interrupt

from rg import config
from rg.llm import chat, chat_json
from rg.search import format_results, web_search
from rg.state import Finding, ResearchState
```

四个角色的 prompt。和 13 篇的区别只有一处:每个 prompt 里都埋了一个中文角色名(`研究规划师` / `研究员` / `报告撰写人` / `报告评审`),`FakeLLM` 靠它分派。

```python
PLANNER_PROMPT = """你是研究规划师。把用户的研究问题拆解成 3~5 个可以独立搜索的子任务。
只输出 JSON: {"tasks": ["子任务1", "子任务2", ...]}
要求: 每个子任务具体、可搜索; 合起来能覆盖原问题; 不要重复。"""

RESEARCHER_PROMPT = """你是研究员。根据提供的搜索结果回答子任务, 要求:
1. 只依据搜索结果, 不编造; 信息不足要明说
2. 保留关键数据、时间、事实
3. 结尾用 [1][2] 的形式标注用到的来源编号"""

REPORTER_PROMPT = """你是报告撰写人。把各子任务的研究材料整合成一篇结构清晰的 Markdown 研究报告:
# 报告标题
## 摘要(3 句话以内)
## 若干章节(对应子任务, 内容相近的可合并)
## 结论
只依据提供的材料写作, 保持客观, 保留材料中的来源标注。"""

REVIEWER_PROMPT = """你是报告评审。按"材料支撑是否充分、结论是否客观、结构是否完整"给报告打 1~10 分。
只输出 JSON: {"score": 整数, "issues": ["问题1", "问题2"]}
分数低于 7 时必须列出具体问题。"""


class TaskPayload(TypedDict):
    """研究节点的输入。它不是 ResearchState —— Send 过来的是单个子任务,
    节点没必要(也不应该)看到全局状态。
    """

    index: int
    task: str


def _maybe_crash(node: str) -> None:
    """故障注入: `RG_CRASH_AT=write` 会让 write 节点抛异常。

    这不是花活 —— 它是**验证 checkpointer 真的有用**的唯一诚实办法。
    没有它, "进程挂了能续跑"只是一句你抄来的宣传语。
    """
    if os.getenv("RG_CRASH_AT") == node:
        raise RuntimeError(f"[注入故障] 节点 {node} 崩了")
```

`_maybe_crash` 这十行是我强烈建议你**留在生产代码里**的东西(它只在环境变量匹配时才生效,平时是死代码)。理由很简单:"我们有 checkpointer,挂了能续跑"是一句需要证据的断言,而唯一的证据就是**真的挂一次**。没有故障注入,你永远不知道自己的续跑逻辑是对的还是只是没被考过。

### 7.1 节点

```python
def plan(state: ResearchState) -> dict:
    """规划: 拆子任务。重新规划时把上一版计划和驳回意见带上。"""
    round_no = state.get("plan_round", 0) + 1
    user = f"研究问题: {state['question']}"
    if state.get("approval", {}).get("action") == "reject":
        user += (
            f"\n\n上一版计划被驳回: {state['tasks']}\n"
            f"驳回意见: {state['approval'].get('comment', '(无)')}\n请重新拆解。"
        )
    data = chat_json(
        [{"role": "system", "content": PLANNER_PROMPT}, {"role": "user", "content": user}]
    )
    tasks = data.get("tasks", [])
    # 宁可失败也不要默默跑偏: 模型偶尔拆出 0 个或 20 个任务
    if not (1 <= len(tasks) <= 8):
        raise ValueError(f"规划结果异常, 子任务数={len(tasks)}: {tasks}")
    return {"tasks": tasks, "plan_round": round_no, "status": "planned"}
```

那个 `1 <= len(tasks) <= 8` 的校验是**成本闸门**,不是洁癖。下一步会按 `len(tasks)` 扇出并行节点,每个节点一次搜索加一次 LLM。模型偶发地拆出 20 个子任务时,你要的是当场失败,而不是一张 20 倍账单。

接下来是这一篇最值得逐字读的节点:

```python
def approve(state: ResearchState) -> dict:
    """人在环中: 把计划摊给人看, 暂停, 等人回话。

    interrupt() 的语义要看清楚:
      - 第一次执行到这里, 它**抛出**中断, 图停下, invoke 返回值里带 __interrupt__
      - 用 Command(resume=X) 再次 invoke 同一个 thread_id 时,
        这个节点**从头重跑**, 而 interrupt() 直接返回 X

    "从头重跑"这件事必须记住: interrupt() 之前的代码会执行两次。
    所以别在它前面发邮件、扣款、写数据库 —— 副作用要放在 interrupt() 之后。
    """
    decision = interrupt(
        {
            "question": state["question"],
            "tasks": state["tasks"],
            "plan_round": state.get("plan_round", 1),
            "hint": "回复 {'action':'approve'} / {'action':'edit','tasks':[...]} / {'action':'reject','comment':'...'}",
        }
    )
    action = (decision or {}).get("action", "approve")
    if action == "edit":
        tasks = decision.get("tasks") or state["tasks"]
        return {"approval": decision, "tasks": tasks, "status": "approved"}
    return {"approval": decision or {"action": "approve"}, "status": action}
```

**"interrupt() 之前的代码会执行两次"这句话是这一篇最容易造成生产事故的一条知识。** 它不是 bug,是恢复机制的必然结果:LangGraph 恢复执行的粒度是**节点**,它没法从函数中间某一行接着跑,只能把整个节点重跑一遍,然后让 `interrupt()` 直接返回你给的值。

于是这样写就完蛋了:

```python
def approve(state):
    send_email_to_boss(state["tasks"])   # ❌ 老板会收到两封
    charge_user(9.9)                     # ❌ 扣两次钱
    decision = interrupt({...})
    ...
```

副作用必须放在 `interrupt()` **后面**。第 12 节的测试里有一条专门守这件事:它断言恢复之后规划师的调用次数**没有**增加。

三个研究/撰写/评审节点:

```python
def research(payload: TaskPayload) -> dict:
    """单个子任务的研究。被 Send 并行拉起 N 份, 各自返回一条 finding。

    返回 {"findings": [一条]} 而不是整个列表 —— reducer 负责拼接。
    """
    _maybe_crash("research")
    results = web_search(payload["task"])
    summary = chat(
        [
            {"role": "system", "content": RESEARCHER_PROMPT},
            {
                "role": "user",
                "content": f"子任务: {payload['task']}\n\n搜索结果:\n{format_results(results)}",
            },
        ]
    )
    finding: Finding = {
        "index": payload["index"],
        "task": payload["task"],
        "summary": summary,
        "sources": [r["url"] for r in results],
    }
    return {"findings": [finding]}


def write(state: ResearchState) -> dict:
    """撰写报告。重写时把评审意见带上 —— 这是 Reflection 循环真正起作用的地方。"""
    _maybe_crash("write")
    # 并行完成顺序是乱的, 报告要按计划顺序写 —— 按 index 排回来
    ordered = sorted(state["findings"], key=lambda f: f["index"])
    material = "\n\n".join(f"### 子任务: {f['task']}\n{f['summary']}" for f in ordered)
    user = f"研究问题: {state['question']}\n\n研究材料:\n{material}"
    if state.get("review", {}).get("issues"):
        user += "\n\n上一版的评审意见, 请逐条改进:\n" + "\n".join(
            f"- {i}" for i in state["review"]["issues"]
        )
    report = chat(
        [{"role": "system", "content": REPORTER_PROMPT}, {"role": "user", "content": user}],
        temperature=0.5,
    )
    return {"report": report, "status": "written"}


def review(state: ResearchState) -> dict:
    data = chat_json(
        [
            {"role": "system", "content": REVIEWER_PROMPT},
            {"role": "user", "content": state["report"]},
        ]
    )
    return {
        "review": data,
        "review_round": state.get("review_round", 0) + 1,
        "status": "reviewed",
    }


def finalize(state: ResearchState) -> dict:
    return {"status": "done"}


def rejected(state: ResearchState) -> dict:
    """人驳回且已经改过 MAX_ROUNDS 版计划 —— 停下来, 别再烧钱。"""
    return {"status": "rejected"}
```

`research` 的签名是 `payload: TaskPayload`,不是 `state: ResearchState`。这是 `Send` 的特性:**被 Send 拉起的节点收到的是 Send 的第二个参数,不是全局状态**。这个设计很好——研究一个子任务不需要看到别的子任务,少一个耦合点。

`finalize` 和 `rejected` 只改一个 `status` 字段,看着像废节点。它们不是:图需要**有名字的终态**。没有它们,你只能靠"`next` 是空的"判断结束,而分不清"顺利跑完"和"被人否决后停下"。给终态起名字,HTTP 层和监控就都有东西可看了。

### 7.2 边:两个必须有上限的地方

```python
def after_approval(state: ResearchState) -> list[Send] | Literal["plan", "rejected"]:
    """审批之后去哪:
    - 批准/改过 → 扇出成 N 个并行研究节点
    - 驳回 → 回去重新规划; 但重规划次数有上限, 否则人和模型可以互相拉扯到破产
    """
    action = state.get("approval", {}).get("action", "approve")
    if action == "reject":
        if state.get("plan_round", 1) >= config.MAX_ROUNDS:
            return "rejected"
        return "plan"
    return [Send("research", {"index": i, "task": t}) for i, t in enumerate(state["tasks"])]


def after_review(state: ResearchState) -> Literal["write", "finalize"]:
    """评审之后去哪: 分数够了就收工, 不够就退回重写 —— 但有轮数上限。

    **上限这件事没得商量。** 没有它, 一个刻薄的评审模型能让你无限重写,
    每一轮都是真金白银。所有"自我改进"循环都必须有硬性出口。
    """
    score = state.get("review", {}).get("score", 0)
    if score >= config.REVIEW_PASS or state.get("review_round", 0) >= config.MAX_ROUNDS:
        return "finalize"
    return "write"
```

这两个函数里各有一个上限,原因不同但同样致命:

- `after_review` 的上限防的是**模型**:评审模型可以永远不满意。
- `after_approval` 的上限防的是**人**:一个心情不好的审批人可以一直点驳回,每次驳回都触发一次重新规划。

写任何带回路的图时,请先回答"这个回路的出口在哪",再写节点。

### 7.3 组装

```python
def build_graph(checkpointer=None):
    g = StateGraph(ResearchState)
    g.add_node("plan", plan)
    g.add_node("approve", approve)
    g.add_node("research", research)
    g.add_node("write", write)
    g.add_node("review", review)
    g.add_node("finalize", finalize)
    g.add_node("rejected", rejected)

    g.add_edge(START, "plan")
    g.add_edge("plan", "approve")
    # 条件边的第三个参数是"这个函数可能返回哪些目标", 只用于画图和校验;
    # Send 的目标不在这里列, 因为 Send 是运行时才决定发几份的
    g.add_conditional_edges("approve", after_approval, ["plan", "research", "rejected"])
    g.add_edge("research", "write")  # N 个 research 全部完成后才进 write(自动汇合)
    g.add_edge("write", "review")
    g.add_conditional_edges("review", after_review, ["write", "finalize"])
    g.add_edge("finalize", END)
    g.add_edge("rejected", END)

    return g.compile(checkpointer=checkpointer)
```

`g.add_edge("research", "write")` 这一行是这个文件里信息量最高的一行:**N 个并行的 research 全部完成之后,write 才会执行一次。** 13 篇里那段 `ThreadPoolExecutor` + `as_completed` + `future_to_index` 的十行代码,在这里是一条边。汇合(join)是框架给的,你不需要写,也不需要担心写漏。

`build_graph` 接受 `checkpointer=None` 而不是自己造一个,是为了让**测试用 InMemorySaver、服务用 PostgresSaver、CLI 用另一个连接池**——同一张图,三种持久化,一行都不改。

最后是可执行的自检:

```python
if __name__ == "__main__":
    from langgraph.checkpoint.memory import InMemorySaver

    app = build_graph(InMemorySaver())
    if "--graph" in sys.argv:
        print(app.get_graph().draw_ascii())
        raise SystemExit(0)

    cfg = {"configurable": {"thread_id": "demo-1"}}
    out = app.invoke({"question": "2025 年具身智能的进展"}, cfg)
    print("== 第一次 invoke 停在了中断上 ==")
    itr = out["__interrupt__"][0]
    print("待审批的计划:")
    for i, t in enumerate(itr.value["tasks"], 1):
        print(f"  {i}. {t}")
    print(f"下一个要执行的节点: {app.get_state(cfg).next}")

    print("\n== 批准, 继续跑完 ==")
    final = app.invoke(Command(resume={"action": "approve"}), cfg)
    print(
        f"status={final['status']} 发现数={len(final['findings'])} 评审轮数={final['review_round']}"
    )
    print(f"评审: {final['review']}")
    print("\n报告:\n" + final["report"])
```

**✅ Checkpoint 3**:先把图画出来,确认拓扑和你脑子里的一致。

```
$ python -m rg.graph --graph
          +-----------+
          | __start__ |
          +-----------+
                *
                *
                *
            +------+
            | plan |
            +------+
                *
                *
                *
          +---------+
          | approve |
          +---------+
           .         ..
         ..            .
        .               ..
+----------+              .
| research |              .
+----------+              .
      *                   .
      *                   .
      *                   .
  +-------+               .
  | write |               .
  +-------+               .
      .                   .
      .                   .
      .                   .
 +--------+               .
 | review |               .
 +--------+               .
      .                   .
      .                   .
      .                   .
+----------+        +----------+
| finalize |        | rejected |
+----------+        +----------+
           *         *
            **     **
              *   *
          +---------+
          | __end__ |
          +---------+
```

看这张图时注意两件事:`research` 只画了**一个**框(扇出几份是运行时才知道的),`review → write` 那条回边在 ASCII 图里看不出方向(`grandalf` 画的是无向布局)——**图里的循环靠读代码确认,不靠看图**。

**✅ Checkpoint 4**:跑完整条链路(用内存 saver,不需要 Postgres)。

```
$ python -m rg.graph
== 第一次 invoke 停在了中断上 ==
待审批的计划:
  1. 2025 年具身智能的进展的主要参与者与代表性方案
  2. 2025 年具身智能的进展的关键技术路线
  3. 2025 年具身智能的进展的落地案例与商业化情况
下一个要执行的节点: ('approve',)

== 批准, 继续跑完 ==
status=done 发现数=3 评审轮数=2
评审: {'score': 9, 'issues': []}

报告:
# 2025 年具身智能的进展研究报告

## 摘要

本报告基于 3 个子方向的检索材料, 给出现状与结论。

## 现状

规模化部署使成本下降约 40%。[1][2]

## 数据来源与时间范围

材料均来自 2025 年公开报告。[1][2][3]

## 结论

该方向已进入工程落地阶段, 建议按场景分批推进。
```

三个数字请对照着看:**发现数=3**(三个并行节点的结果都在,reducer 生效了)、**评审轮数=2**(第一版被打了 5 分,重写了一次)、报告里多出来的 **`## 数据来源与时间范围`** 一节(重写确实按意见改了内容,不是原地转一圈)。

## 8. rg/why_reducer.py —— 30 行证明一件事(20 分钟)

上一节 `state.py` 里那个 `Annotated[list[Finding], operator.add]`,是最容易被当成"抄一下就行"的一行。这个文件的存在就是为了让你**亲眼看见删掉它会怎样**。

```python
"""一个 30 行的对照实验: 并行节点的状态字段不加 reducer 会发生什么。

这个文件不属于业务代码, 但它值得留在仓库里 ——
它是"为什么 state.py 里那个 Annotated 不能删"的证据。

**实测结论(langgraph 1.2)**: 不加 reducer 不是静默丢数据, 而是直接抛
InvalidUpdateError: "Can receive only one value per step"。
这是个好设计 —— 框架宁可让你当场失败, 也不给你一个悄悄少了两条数据的结果。
早期 0.x 版本在部分路径上是覆盖语义, 所以网上还能搜到"静默丢数据"的说法;
自己跑一遍比信任任何二手说法都可靠。
"""

from __future__ import annotations

import operator
from typing import Annotated, TypedDict

from langgraph.errors import InvalidUpdateError
from langgraph.graph import END, START, StateGraph
from langgraph.types import Send


class Bad(TypedDict, total=False):
    items: list[str]  # 默认合并语义: 一步之内只允许一个写入者


class Good(TypedDict, total=False):
    items: Annotated[list[str], operator.add]  # 合并语义 = 拼接


def worker(payload: dict) -> dict:
    return {"items": [payload["name"]]}


def fanout(state) -> list[Send]:
    return [Send("worker", {"name": n}) for n in ("a", "b", "c")]


def build(schema):
    g = StateGraph(schema)
    g.add_node("start", lambda s: {})
    g.add_node("worker", worker)
    g.add_edge(START, "start")
    g.add_conditional_edges("start", fanout, ["worker"])
    g.add_edge("worker", END)
    return g.compile()


if __name__ == "__main__":
    for name, schema in (("不加 reducer", Bad), ("加 operator.add", Good)):
        try:
            out = build(schema).invoke({})
            print(f"{name:<16} 3 个并行节点各写 1 条 → items = {out.get('items')}")
        except InvalidUpdateError as e:
            head = str(e).splitlines()[0]
            print(f"{name:<16} 3 个并行节点各写 1 条 → {type(e).__name__}: {head}")
```

**✅ Checkpoint 5**:

```
$ python -m rg.why_reducer
不加 reducer       3 个并行节点各写 1 条 → InvalidUpdateError: At key 'items': Can receive only one value per step. Use an Annotated key to handle multiple values.
加 operator.add   3 个并行节点各写 1 条 → items = ['a', 'b', 'c']
```

这里有一个**需要更正流传说法**的点,值得单独讲。

我写这个文件之前,凭印象在文档里写的是"不加 reducer 会静默覆盖,你会得到 1 条而不是 3 条"。跑完发现不对:langgraph 1.2 是**直接抛异常**的,而且异常信息里就写着解决办法(`Use an Annotated key`)。网上那些"静默丢数据"的说法多半来自 0.x 时代,或者是别人也在传抄。

这件事的教训比这个知识点本身更重要:**任何"框架会怎样"的断言,自己跑一遍再往文档里写。** 一个只花 30 行、20 分钟的对照实验,能替你挡掉一段错误的记忆,还能在同事问"这个 Annotated 能不能删"的时候当场给出答案。

## 9. rg/checkpointer.py —— 从"演示能跑"到"能上线"(30 分钟)

```python
"""Postgres checkpointer —— 让"进程挂了从中断处继续"这句话成真。

checkpointer 是 LangGraph 里最值钱的一个零件, 也是最容易被教程一句
`InMemorySaver()` 糊过去的一个。它们的区别不是性能, 是**语义**:

  InMemorySaver  —— 进程退出, 所有 thread 消失。演示够用, 上线等于没有
  PostgresSaver  —— 每个节点跑完把状态写进表, 换台机器、隔一天都能接着跑

用连接池而不是单连接: HTTP 服务是多请求并发的, 单连接会串行化,
而且一次网络抖动就把所有 thread 一起搞挂。
"""

from __future__ import annotations

from contextlib import contextmanager

from langgraph.checkpoint.postgres import PostgresSaver
from psycopg_pool import ConnectionPool

from rg import config

# autocommit=True 是 LangGraph 官方要求: checkpointer 自己管事务边界,
# 外面再套一层事务会让 setup() 的 CREATE TABLE 卡住
_POOL_KW = {
    "autocommit": True,
    "prepare_threshold": 0,  # 连接池 + 预备语句在 pgbouncer 后面会互相打架, 关掉
    "row_factory": None,
}


@contextmanager
def checkpointer(dsn: str | None = None, *, setup: bool = False):
    """开一个带连接池的 PostgresSaver。

    setup=True 会建表(幂等)。生产里建议只在部署脚本里跑一次,
    而不是每次进程启动都跑 —— 多实例同时 CREATE TABLE 会互相锁。
    """
    dsn = dsn or config.PG_DSN
    kw = {k: v for k, v in _POOL_KW.items() if v is not None}
    with ConnectionPool(conninfo=dsn, min_size=1, max_size=5, kwargs=kw) as pool:
        saver = PostgresSaver(pool)
        if setup:
            saver.setup()
        yield saver


if __name__ == "__main__":
    with checkpointer(setup=True) as cp:
        print(f"checkpointer 就绪: {config.PG_DSN}")
        threads = {t.config["configurable"]["thread_id"] for t in cp.list(None, limit=200)}
        print(f"库里已有 {len(threads)} 个 thread: {sorted(threads) or '(空)'}")
```

三个参数值得记住,它们都是**踩过才知道**的:

- `autocommit=True` —— 官方要求。checkpointer 自己管事务边界,外面再包一层事务,`setup()` 里的 `CREATE TABLE` 会卡在锁上。
- `prepare_threshold=0` —— 关掉预备语句。连接池 + 预备语句在 pgbouncer 这类连接复用器后面会互相打架(同一个物理连接上出现别人的预备语句名)。本机直连不会出问题,但这个默认值会在你上生产的那一天咬你。
- `min_size=1, max_size=5` —— 池而不是单连接。HTTP 服务是并发的。

**✅ Checkpoint 6**:建表并确认表结构。

```
$ python -m rg.checkpointer
checkpointer 就绪: postgresql://rg:rg@127.0.0.1:5432/rg
库里已有 0 个 thread: (空)

$ docker exec rg-pg psql -U rg -d rg -c '\dt'
               List of relations
 Schema |         Name          | Type  | Owner
--------+-----------------------+-------+-------
 public | checkpoint_blobs      | table | rg
 public | checkpoint_migrations | table | rg
 public | checkpoint_writes     | table | rg
 public | checkpoints           | table | rg
(4 rows)
```

四张表,各管一件事:`checkpoints` 存每一步的状态快照,`checkpoint_blobs` 存大字段(报告正文这种),`checkpoint_writes` 存"这一步各个节点分别写了什么"(并行汇合靠它),`checkpoint_migrations` 是版本号。**你不需要碰这些表**,但知道它们存在,你就知道"状态到底在哪"——这个问题在排查线上问题时会被问到。

## 10. rg/cli.py —— 用"多个进程"证明持久化(1.5 小时)

这个文件的设计有一条硬性纪律,请先读文件头:

```python
"""命令行入口: 把 checkpointer 的四种能力做成能单独验证的子命令。

  start    <question>        起一个研究, 停在审批中断上
  resume   <thread> <action> 批准 / 改计划 / 驳回, 继续跑
  status   <thread>          当前停在哪、状态里有什么
  history  <thread>          每一步的快照(回放的原料)
  fork     <thread> <step>   从历史某一步分叉出一条新线重跑

**每个子命令都是一个独立进程。** 这一点是整篇的关键:
进程之间什么都不共享, 能接着跑完全靠 Postgres 里的 checkpoint。
如果这些命令换成同一个进程里的几个函数调用, 就什么都没证明。
"""
```

**"如果这些命令换成同一个进程里的几个函数调用,就什么都没证明"** —— 这句话适用于所有关于持久化的验证。同进程里跑,内存里的对象还活着,你测的是内存,不是数据库。

两个打印辅助函数:

```python
def _print_interrupt(out: dict) -> None:
    itr = out.get("__interrupt__")
    if not itr:
        return
    v = itr[0].value
    print(f"⏸  等待人工审批(第 {v['plan_round']} 版计划)")
    for i, t in enumerate(v["tasks"], 1):
        print(f"   {i}. {t}")
    print(f"   提示: {v['hint']}")


def _print_final(out: dict) -> None:
    print(f"✅ status={out.get('status')}")
    if out.get("status") == "rejected":
        print("   计划被驳回且已达重规划上限, 已停止(没有继续消耗模型调用)")
        return
    print(f"   发现数={len(out.get('findings', []))} 评审轮数={out.get('review_round')}")
    print(f"   评审={out.get('review')}")
    print("\n" + (out.get("report") or "(无报告)"))
```

`start` / `resume`:

```python
def cmd_start(args) -> int:
    with checkpointer() as cp:
        app = build_graph(cp)
        cfg = {"configurable": {"thread_id": args.thread}}
        out = app.invoke({"question": args.question}, cfg)
        print(f"thread_id = {args.thread}")
        _print_interrupt(out)
        snap = app.get_state(cfg)
        print(f"   下一个待执行节点: {snap.next}")
    return 0


def cmd_resume(args) -> int:
    payload: dict = {"action": args.action}
    if args.action == "edit":
        payload["tasks"] = args.tasks
    if args.action == "reject":
        payload["comment"] = args.comment or ""

    with checkpointer() as cp:
        app = build_graph(cp)
        cfg = {"configurable": {"thread_id": args.thread}}
        snap = app.get_state(cfg)
        if not snap.next:
            print(f"thread {args.thread} 已经跑完或不存在, 没有可恢复的中断")
            return 1
        out = app.invoke(Command(resume=payload), cfg)
        # 驳回后会回到 plan 再次停在 approve 上, 所以这里可能又是一个中断
        if out.get("__interrupt__"):
            _print_interrupt(out)
        else:
            _print_final(out)
    return 0
```

`cmd_resume` 结尾那个 `if out.get("__interrupt__")` 不是防御性代码。**一次 `invoke` 可以停在下一个中断上**——驳回之后图会回到 `plan`、重新规划、再次停在 `approve`。任何调用 `invoke` 的地方都要准备好"我可能又拿到一个中断"。

接下来是整篇**最容易混**的一处:

```python
def cmd_continue(args) -> int:
    """接着跑 —— 不带任何人工输入。

    和 resume 的区别是整篇里最容易混的一处:
      Command(resume=X)  回答一个 interrupt(把 X 交给暂停的那个节点)
      invoke(None, cfg)  纯粹"从上次停的地方往下跑", 用于崩溃之后的续跑

    拿 invoke(None) 去回答 interrupt, 图会**原地再停一次**;
    拿 Command(resume=...) 去续跑一个崩掉的节点, 那个 resume 值没人接, 直接丢掉。
    """
    with checkpointer() as cp:
        app = build_graph(cp)
        cfg = {"configurable": {"thread_id": args.thread}}
        snap = app.get_state(cfg)
        if not snap.next:
            print(f"thread {args.thread} 已经跑完, 没有待执行节点")
            return 1
        print(f"从 {snap.next} 继续(已有 {len(snap.values.get('findings', []))} 条发现, 不会重跑)")
        out = app.invoke(None, cfg)
        if out.get("__interrupt__"):
            _print_interrupt(out)
        else:
            _print_final(out)
    return 0
```

把这两行贴在显示器边上:

| | 用途 | 用错的后果 |
| --- | --- | --- |
| `invoke(Command(resume=X), cfg)` | 回答一个 `interrupt()` | 对一个崩掉的节点用它:`X` 没人接收,被丢掉,等于 `invoke(None)` |
| `invoke(None, cfg)` | 从上次停的节点往下跑(崩溃续跑) | 对一个等审批的 thread 用它:`approve` 节点重跑并**再次 interrupt**,原地不动 |

第 13 节的 HTTP 层会把这个区别做成两个不同的接口,并且**替调用方挡住用错的那个**——因为用错时图不会报错,只会"看起来什么都没发生",这种失败最难查。

`status` 和 `history`:

```python
def cmd_status(args) -> int:
    with checkpointer() as cp:
        app = build_graph(cp)
        cfg = {"configurable": {"thread_id": args.thread}}
        snap = app.get_state(cfg)
        if snap.created_at is None:
            print(f"thread {args.thread} 不存在")
            return 1
        v = snap.values
        print(f"thread     : {args.thread}")
        print(f"next       : {snap.next or '(已结束)'}")
        print(f"status     : {v.get('status')}")
        print(f"question   : {v.get('question')}")
        print(f"tasks      : {len(v.get('tasks', []))} 个")
        print(f"findings   : {len(v.get('findings', []))} 条")
        print(f"plan_round : {v.get('plan_round')}  review_round: {v.get('review_round')}")
        print(f"report     : {len(v.get('report') or '')} 字")
    return 0
```

`snap.created_at is None` 是判断"thread 不存在"的办法。这个判断不太直观(为什么不是 `snap is None`?)——`get_state` 对不存在的 thread 也会返回一个 `StateSnapshot`,只是里面全是空的。

```python
def cmd_history(args) -> int:
    with checkpointer() as cp:
        app = build_graph(cp)
        cfg = {"configurable": {"thread_id": args.thread}}
        # get_state_history 是**倒序**的(最新在前), 回放时要注意这一点
        snaps = list(app.get_state_history(cfg))
        if not snaps:
            print(f"thread {args.thread} 没有历史")
            return 1
        # snapshot 的 metadata 里只有 step/source, **没有"是哪个节点写的"**。
        # 想知道每一步执行了什么, 得看上一个快照的 next —— 那就是这一步跑的节点。
        # 按时间正序排一遍再配对
        chrono = list(reversed(snaps))
        print(f"共 {len(snaps)} 个快照:\n")
        print(f"{'step':>5}  {'本步执行的节点':<26} {'下一步':<26} findings  status")
        prev_step = None
        for i, s in enumerate(chrono):
            step = s.metadata.get("step", "?")
            # step 号回退, 说明这里开始是 fork 出来的另一条分支。
            # 同一个 thread 里可以有多条分支, get_state_history 会把它们**拼在一起**返回
            forked = prev_step is not None and isinstance(step, int) and step <= prev_step
            if forked:
                print(f"  --- 以上为原分支; 以下从 step {step - 1} 分叉 ---")
            prev_step = step if isinstance(step, int) else prev_step
            if i == 0:
                ran = f"({s.metadata.get('source')})"
            elif forked:
                ran = "(fork: update_state)"
            else:
                ran = ",".join(chrono[i - 1].next)
            nxt = ",".join(s.next) or "(END)"
            print(
                f"{step:>5}  {ran:<26} {nxt:<26} "
                f"{len(s.values.get('findings', [])):>8}  {s.values.get('status')}"
            )
    return 0
```

这个函数我写错过一次,值得完整讲——**它是一条二手知识害人的典型案例**。

我最初的写法是 `s.metadata["writes"].keys()`,以为 metadata 里记着"这一步哪些节点写了状态"。跑出来每一行都是 `(loop)`,因为 `metadata` 里根本没有 `writes` 这个 key。langgraph 1.2 的 `StateSnapshot.metadata` 只有三个字段:`step`、`source`、`parents`。

正确的做法要绕一下:**上一个快照的 `next`,就是这一步执行的节点**。所以得把倒序的历史 `reversed()` 成正序,然后拿 `chrono[i-1].next` 当"本步执行的节点"。这就是上面那段配对逻辑。

`fork`:

```python
def cmd_fork(args) -> int:
    """时间旅行: 回到某个快照, 改一改状态, 从那里重新往下跑。

    这是排查"为什么这次答得不好"最有力的工具: 不用重头再来,
    直接把当时的中间状态改掉, 只重跑后面的部分。
    """
    with checkpointer() as cp:
        app = build_graph(cp)
        cfg = {"configurable": {"thread_id": args.thread}}
        target = None
        for s in app.get_state_history(cfg):
            if s.metadata.get("step") == args.step:
                target = s
                break
        if target is None:
            print(f"找不到 step={args.step} 的快照")
            return 1

        print(f"回到 step={args.step}, 当时 next={target.next or '(END)'}")
        if args.set_tasks:
            # update_state 会在这个历史点上**新建一个分支**,
            # 原来的那条线一个字都不会被改掉
            forked = app.update_state(target.config, {"tasks": args.set_tasks}, as_node="approve")
            print(f"已在该点写入新计划({len(args.set_tasks)} 个子任务), 分支 checkpoint 已建立")
        else:
            forked = target.config

        out = app.invoke(None, forked)
        if out.get("__interrupt__"):
            _print_interrupt(out)
        else:
            _print_final(out)
    return 0
```

`update_state(target.config, values, as_node="approve")` 三个参数各有讲究:

- 第一个参数是**历史快照的 config**(里面带 `checkpoint_id`),不是 thread 的 config。这就是"回到过去"的意思。
- `as_node="approve"` 是"假装这个写入是 approve 节点做的"。它决定了接下来走哪条边——写完之后图会从 `approve` 的出边继续,也就是 `after_approval`,于是按新的 `tasks` 扇出。
- 返回值是一个**新的 config**,指向新建的分支 checkpoint。往下跑要用它,不是用 `cfg`。

`main()` 就是常规的 argparse,略;完整代码在 `rg/cli.py`。

### 10.1 四种能力,一条一条验

下面五个 Checkpoint 是这一篇的核心。**每条命令都是一个新进程。**

**✅ Checkpoint 7**:起一个研究,进程退出,换个进程还能看到它停在哪。

```
$ python -m rg.cli start "2025 年具身智能的进展" --thread t1
thread_id = t1
⏸  等待人工审批(第 1 版计划)
   1. 2025 年具身智能的进展的主要参与者与代表性方案
   2. 2025 年具身智能的进展的关键技术路线
   3. 2025 年具身智能的进展的落地案例与商业化情况
   提示: 回复 {'action':'approve'} / {'action':'edit','tasks':[...]} / {'action':'reject','comment':'...'}
   下一个待执行节点: ('approve',)

$ python -m rg.cli status t1
thread     : t1
next       : ('approve',)
status     : planned
question   : 2025 年具身智能的进展
tasks      : 3 个
findings   : 0 条
plan_round : 1  review_round: None
report     : 0 字
```

`findings: 0 条` 是这个 Checkpoint 的重点:**图确实停住了,后面的节点一个都没跑。** 不是"跑完了但没告诉你",是真的没花那三次搜索的钱。

**✅ Checkpoint 8**:另一个进程批准,一路跑完,然后看完整的执行历史。

```
$ python -m rg.cli resume t1 approve
✅ status=done
   发现数=3 评审轮数=2
   评审={'score': 9, 'issues': []}

# 2025 年具身智能的进展研究报告
...(报告正文, 同 Checkpoint 4)

$ python -m rg.cli history t1
共 10 个快照:

 step  本步执行的节点                    下一步                        findings  status
   -1  (input)                    __start__                         0  None
    0  __start__                  plan                              0  None
    1  plan                       approve                           0  planned
    2  approve                    research,research,research        0  approve
    3  research,research,research write                             3  approve
    4  write                      review                            3  written
    5  review                     write                             3  reviewed
    6  write                      review                            3  written
    7  review                     finalize                          3  reviewed
    8  finalize                   (END)                             3  done
```

这张表把整张图的行为摊平成了十行,值得逐行看:

- **step 2 → 3**:`approve` 的下一步是 `research,research,research` —— 一个 `Send` 列表在这里变成了三个真实的并行任务。
- **step 3**:三个 research 一起完成,`findings` 从 0 跳到 3,下一步是**一个** `write` —— 汇合发生了。
- **step 4~7**:`write → review → write → review`。评审第一次打 5 分,退回重写;第二次 9 分,进 `finalize`。**Reflection 循环在历史里是看得见的。**
- 最后一行 `next` 是 `(END)`,`status=done`。

这十行就是"可观测性"。13 篇的版本跑完只有一个报告字符串,中间发生了什么全靠 print 猜。

**✅ Checkpoint 9**:让 `write` 节点崩掉,然后续跑——三次搜索不重做。

```
$ python -m rg.cli start "边缘推理的成本结构" --thread t2
   下一个待执行节点: ('approve',)

$ RG_CRASH_AT=write python -m rg.cli resume t2 approve
  File "/private/tmp/research_graph/rg/graph.py", line 68, in _maybe_crash
    raise RuntimeError(f"[注入故障] 节点 {node} 崩了")
RuntimeError: [注入故障] 节点 write 崩了
During task with name 'write' and id '4429ca42-8542-c7ed-3ab9-9faea6c55975'

$ python -m rg.cli status t2
thread     : t2
next       : ('write',)
status     : approve
question   : 边缘推理的成本结构
tasks      : 3 个
findings   : 3 条          ← 关键: 研究成果已经落库
plan_round : 1  review_round: None
report     : 0 字

$ python -m rg.cli continue t2
从 ('write',) 继续(已有 3 条发现, 不会重跑)
✅ status=done
   发现数=3 评审轮数=2
   评审={'score': 9, 'issues': []}
```

**这个 Checkpoint 是整篇的价值所在。** 三行值得反复看:

1. `next: ('write',)` —— 图知道自己停在哪个节点上,而且停在**崩掉的那一个**,不是它后面那个。
2. `findings: 3 条` —— 三次搜索、三次 LLM 调用的成果**已经在数据库里**。
3. `继续(已有 3 条发现, 不会重跑)` —— 续跑从 `write` 开始,研究阶段一次都没重做。

把这件事换算成钱:一个真实的研究任务可能扇出 8 个子任务、每个子任务一次搜索 + 一次长上下文 LLM 调用。撰写节点因为报告太长撞了 token 上限而失败——没有 checkpointer,你得从第一次搜索开始全部重来;有了它,你改个参数从 `write` 接着跑。

**✅ Checkpoint 10**:驳回两次,撞上重规划上限自动停。

```
$ python -m rg.cli start "低空经济的商业模式" --thread t3
   下一个待执行节点: ('approve',)

$ python -m rg.cli resume t3 reject --comment "太笼统, 请按技术/政策/市场三条线拆"
⏸  等待人工审批(第 2 版计划)
   1. 低空经济的商业模式的主要参与者与代表性方案
   2. 低空经济的商业模式的关键技术路线
   3. 低空经济的商业模式的落地案例与商业化情况
   提示: 回复 {'action':'approve'} / {'action':'edit','tasks':[...]} / {'action':'reject','comment':'...'}

$ python -m rg.cli resume t3 reject --comment "还是不行"
✅ status=rejected
   计划被驳回且已达重规划上限, 已停止(没有继续消耗模型调用)
```

第二次驳回没有进入研究阶段,`status` 是有名字的终态 `rejected`。(这里的第二版计划和第一版长得一样,是因为 `FakeLLM` 不看驳回意见——真模型会按 `comment` 重拆。要验证"意见确实传下去了",看 `plan` 节点拼的那段 user 消息就行。)

**✅ Checkpoint 11**:人工改计划,扇出数量随之改变。

```
$ python -m rg.cli start "固态电池的量产瓶颈" --thread t4
   下一个待执行节点: ('approve',)

$ python -m rg.cli resume t4 edit --tasks "固态电解质的量产良率" "车企的装车时间表"
✅ status=done
   发现数=2 评审轮数=2
   评审={'score': 9, 'issues': []}

$ python -m rg.cli history t4 | head -9
共 10 个快照:

 step  本步执行的节点                    下一步                        findings  status
   -1  (input)                    __start__                         0  None
    0  __start__                  plan                              0  None
    1  plan                       approve                           0  planned
    2  approve                    research,research                 0  approved
    3  research,research          write                             2  approved
    4  write                      review                            2  written
```

`research,research` —— 两个,不是三个。**扇出数量是运行时按 `len(tasks)` 决定的**,而 `tasks` 刚被人改过。这就是 `Send` 相对于"预先画好 N 个并行分支"的价值。

**✅ Checkpoint 12**:时间旅行——回到 step 2,改掉当时的计划,只重跑后面。

```
$ python -m rg.cli fork t1 2 --set-tasks "只看人形机器人这一条线"
回到 step=2, 当时 next=('research', 'research', 'research')
已在该点写入新计划(1 个子任务), 分支 checkpoint 已建立
✅ status=done
   发现数=1 评审轮数=2
   评审={'score': 9, 'issues': []}

$ python -m rg.cli history t1
共 17 个快照:

 step  本步执行的节点                    下一步                        findings  status
   -1  (input)                    __start__                         0  None
    0  __start__                  plan                              0  None
    1  plan                       approve                           0  planned
    2  approve                    research,research,research        0  approve
    3  research,research,research write                             3  approve
    4  write                      review                            3  written
    5  review                     write                             3  reviewed
    6  write                      review                            3  written
    7  review                     finalize                          3  reviewed
    8  finalize                   (END)                             3  done
  --- 以上为原分支; 以下从 step 2 分叉 ---
    3  (fork: update_state)       research                          0  approve
    4  research                   write                             1  approve
    5  write                      review                            1  written
    6  review                     write                             1  reviewed
    7  write                      review                            1  written
    8  review                     finalize                          1  reviewed
    9  finalize                   (END)                             1  done
```

十个快照变成十七个,**同一个 thread 里现在有两条分支**,原来那条一个字都没改。注意分叉后的 step 3 只扇出**一个** `research`(新计划只有一个子任务),而且 `plan` 节点**没有重跑**——我们是从 step 2 之后接着跑的,规划阶段的钱不用再付一次。

两条分支都能读回来:

```python
from rg.checkpointer import checkpointer
from rg.graph import build_graph

with checkpointer() as cp:
    app = build_graph(cp)
    cfg = {"configurable": {"thread_id": "t1"}}
    ends = [s for s in app.get_state_history(cfg) if not s.next]   # 两个终态
    for s in ends:
        cid = s.config["configurable"]["checkpoint_id"]
        print(cid, len(s.values["findings"]))
    # 想读哪条分支, 就带上那条分支的 checkpoint_id
    old = ends[-1].config["configurable"]["checkpoint_id"]
    snap = app.get_state({"configurable": {"thread_id": "t1", "checkpoint_id": old}})
    print(snap.values["status"], len(snap.values["findings"]))
```

实测:

```
t1 里有 2 个终态快照(两条分支各一个):
  checkpoint_id=1f19aae0-4a1e-66be-8009-bacd04c4c60b  findings=1  报告 161 字
  checkpoint_id=1f19aade-93bb-6e7c-8008-afbfbcf7eed0  findings=3  报告 161 字

按 checkpoint_id 直接读原分支: status=done findings=3
原分支的子任务: ['2025 年具身智能的进展的主要参与者与代表性方案', ...]
```

**`thread_id` 定位一条线,`thread_id + checkpoint_id` 定位一条线上的一个点。** 这就是时间旅行的全部机制,没有魔法。

这个能力在真实工作里怎么用?两个场景:

- **排查**:用户投诉"这份报告结论不对"。你不用重跑,直接翻历史看每一步的中间状态,找到是哪一步开始跑偏的。
- **对比**:想知道"如果当时子任务拆得更细会不会更好",fork 一条分支重跑后面,两个报告并排看。代价只有分叉点之后那部分的调用费。

## 11. rg/ratelimit.py —— 一份可以直接搬过来的限流器(20 分钟)

19 篇写过一份一样的 Lua 令牌桶,这里直接复用。差别只有一处,而这处差别值得说:

```python
"""Redis 令牌桶限流。

和 docs/2/19 的 cache.py 里那份是同一套 Lua, 这里单独拎出来是因为
这一篇的服务只需要限流、不需要答案缓存 —— agent 的每次执行都是有状态的,
缓存"同样的问题返回同样的答案"在这里是错的。

Lua 的理由和上一篇一样: 令牌桶是"读余额 → 按时间补 → 扣减 → 写回"四步,
分开发命令在并发下会交错, 两个请求读到同一个余额, 于是漏放。
"""
```

**19 篇有答案缓存,这一篇没有,而且不该有。** 文档问答是无状态的:同一个问题、同一份知识库,答案就该一样,缓存是纯收益。研究 agent 不是:每次执行都有自己的 thread、自己的审批记录、自己的分支历史。给它加"同样的问题返回上次的报告"这种缓存,等于把一个有状态的长任务伪装成一次查询。**照抄上一个项目的配件之前,先问一句"这个项目的语义还成立吗"。**

Lua 脚本和 19 篇完全一样:

```python
_LUA = """
local key = KEYS[1]
local cap = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local st = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(st[1])
local ts = tonumber(st[2])
if tokens == nil then tokens = cap; ts = now end
tokens = math.min(cap, tokens + (now - ts) * rate)
local ok = 0
if tokens >= 1 then tokens = tokens - 1; ok = 1 end
redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', key, 3600)
return {ok, tostring(tokens)}
"""


class RateLimiter:
    def __init__(self, capacity: int = 5, refill_per_sec: float = 0.5) -> None:
        url = os.getenv("RG_REDIS_URL", "redis://127.0.0.1:6379/1")
        self.r = redis.from_url(url, decode_responses=True)
        self.cap = capacity
        self.rate = refill_per_sec
        self.script = self.r.register_script(_LUA)

    def allow(self, who: str) -> tuple[bool, float]:
        now = self.r.time()[0]  # 用 Redis 的时间, 不用各自机器的本地时间
        ok, left = self.script(keys=[f"rg:rl:{who}"], args=[self.cap, self.rate, now])
        return bool(ok), float(left)
```

`self.r.time()[0]` 而不是 `time.time()`:多个 API 实例的本机时钟会有几百毫秒到几秒的偏差,而令牌桶的补充量是 `(now - ts) * rate`。用各自的本地时钟,时钟快的那台实例会算出更多令牌。**分布式限流必须有一个统一的时钟源,而 Redis 自己就是那个源。**

**✅ Checkpoint 13**:

```
$ python -m rg.ratelimit
容量 3, 每秒补 0.5 个, 连打 5 次:
  第 1 次: 通过  余额=2.00
  第 2 次: 通过  余额=1.00
  第 3 次: 通过  余额=0.00
  第 4 次: 拒绝  余额=0.00
  第 5 次: 拒绝  余额=0.00
等 2.1 秒(应补回约 1 个令牌)...
  再打一次: 通过  余额=0.00
```

自检脚本里 `user = f"probe-{uuid.uuid4().hex[:6]}"` 是刻意的:桶状态在 Redis 里留着,固定 key 会让第二次跑这个脚本时"第 1 次就被拒绝",看着像代码坏了。**任何按标识计数的东西,验证时都要用一次性的标识。** 第 12 节的测试会因为忘了这条纪律吃一次苦头。

## 12. tests/ —— 给状态机写测试(2 小时)

这一节是这一篇和前面所有手敲项目最不一样的地方。**给 agent 写测试,测的不是"模型答得对不对",而是"图走了哪条路"。**

`tests/test_graph.py` 文件头就是这份答案:

```python
"""测试。重点全部在**控制流**上, 不在模型输出上。

一个 agent 项目值得测什么, 这份文件就是答案:
  - 图的路径对不对(批准 → 研究; 驳回 → 重规划; 驳回到上限 → 停)
  - 并行结果有没有丢(reducer)
  - 中断能不能恢复、恢复后状态对不对
  - 崩溃之后续跑, 已完成的节点**不会重跑**
  - 循环有没有硬性出口

LLM 用 FakeLLM(不是 mock 库), 因为我们要断言的是"图走了哪条路",
需要一个行为完全确定的模型。InMemorySaver 够用的地方就用它 ——
只有验证"跨进程续跑"时才必须上 Postgres, 那部分单独一个测试并可跳过。
"""

@pytest.fixture(autouse=True)
def fake_llm():
    """每个测试一个全新的 FakeLLM —— 共享实例会让测试之间互相污染。"""
    inst = llm.FakeLLM()
    llm.reset_llm(inst)
    yield inst
    llm.reset_llm(None)


@pytest.fixture
def app():
    return build_graph(InMemorySaver())


def cfg(name: str = "") -> dict:
    return {"configurable": {"thread_id": name or uuid.uuid4().hex[:8]}}
```

### 12.1 中断与恢复

```python
def test_停在审批中断上而不是一路跑完(app):
    c = cfg()
    out = app.invoke({"question": "测试问题一"}, c)
    assert "__interrupt__" in out, "图应该在 approve 节点暂停"
    assert out["__interrupt__"][0].value["plan_round"] == 1
    assert app.get_state(c).next == ("approve",)
    # 关键: 暂停时后面的节点一个都没跑
    assert out.get("findings") is None or out["findings"] == []
    assert not out.get("report")


def test_approve_节点在恢复时会从头重跑(app, fake_llm):
    """interrupt() 之前的代码执行两次 —— 这不是 bug, 是设计。

    所以副作用(发通知、扣款)绝不能写在 interrupt() 前面。
    这里用规划师的调用次数来证明: plan 只跑了一次(它在 approve 之前的节点里),
    而 approve 自己被执行了两遍。
    """
    c = cfg()
    app.invoke({"question": "测试问题三"}, c)
    planner_calls = [x for x in fake_llm.calls if "研究规划师" in x["role_hint"]]
    assert len(planner_calls) == 1
    app.invoke(Command(resume={"action": "approve"}), c)
    # 恢复之后规划师**没有**被再调一次 —— 已完成的节点不重跑
    planner_calls = [x for x in fake_llm.calls if "研究规划师" in x["role_hint"]]
    assert len(planner_calls) == 1
```

第二个测试是**用调用计数断言执行语义**的范例。"已完成的节点不会重跑"这句话没法直接断言,但"规划师被调用的次数没变"可以。`FakeLLM.calls` 那个列表就是为这类断言存在的——**给替身加一个调用记录,你就能测到框架的行为,而不只是自己代码的返回值。**

### 12.2 分支

```python
def test_驳回会回到规划并再次暂停(app):
    c = cfg()
    app.invoke({"question": "测试问题四"}, c)
    out = app.invoke(Command(resume={"action": "reject", "comment": "太笼统"}), c)
    assert "__interrupt__" in out, "驳回后应重新规划并再次等待审批"
    assert out["__interrupt__"][0].value["plan_round"] == 2


def test_驳回到上限就停下不再烧钱(app, fake_llm):
    c = cfg()
    app.invoke({"question": "测试问题五"}, c)
    app.invoke(Command(resume={"action": "reject", "comment": "第一次不行"}), c)
    out = app.invoke(Command(resume={"action": "reject", "comment": "还是不行"}), c)
    assert out["status"] == "rejected"
    assert "__interrupt__" not in out
    # 没有进入研究阶段 —— 一次研究员调用都没有
    assert not [x for x in fake_llm.calls if "研究员" in x["role_hint"]]


def test_人工改计划后按新计划扇出(app):
    c = cfg()
    app.invoke({"question": "测试问题六"}, c)
    out = app.invoke(Command(resume={"action": "edit", "tasks": ["只研究这一个方向"]}), c)
    assert out["status"] == "done"
    assert len(out["findings"]) == 1, "改成 1 个子任务后应该只扇出 1 个研究节点"
    assert out["findings"][0]["task"] == "只研究这一个方向"
```

`assert not [x for x in fake_llm.calls if "研究员" in x["role_hint"]]` 这一行是**成本断言**:不是"状态对不对",而是"有没有花不该花的钱"。带成本的分支都应该有这样一条。

### 12.3 并行与循环

```python
def test_三个并行研究节点的结果不会互相覆盖(app):
    c = cfg()
    app.invoke({"question": "测试问题七"}, c)
    out = app.invoke(Command(resume={"action": "approve"}), c)
    idx = sorted(f["index"] for f in out["findings"])
    assert idx == [0, 1, 2], "reducer 失效时这里会少于 3 条"


def test_评审不通过会重写一次(app, fake_llm):
    c = cfg()
    app.invoke({"question": "测试问题九"}, c)
    out = app.invoke(Command(resume={"action": "approve"}), c)
    writer_calls = [x for x in fake_llm.calls if "报告撰写人" in x["role_hint"]]
    assert len(writer_calls) == 2, "第一版评审 5 分应触发一次重写"
    assert out["review_round"] == 2
    assert "数据来源与时间范围" in out["report"], "重写应该真的按意见改了内容"


def test_重写循环有硬性出口(app, monkeypatch):
    """把评审改成永远不满意, 验证循环不会无限转。"""

    class NeverHappy(llm.FakeLLM):
        def chat(self, messages, temperature=0.0):
            if "报告评审" in messages[0]["content"]:
                import json as _json

                return _json.dumps({"score": 1, "issues": ["永远不行"]}, ensure_ascii=False)
            return super().chat(messages, temperature=temperature)

    llm.reset_llm(NeverHappy())
    c = cfg()
    app.invoke({"question": "测试问题十"}, c)
    out = app.invoke(Command(resume={"action": "approve"}), c)
    assert out["status"] == "done"
    assert out["review_round"] == config.MAX_ROUNDS, "必须在 MAX_ROUNDS 处退出"
```

`NeverHappy` 这个子类是**测试极端行为**的标准手法:继承替身,只改你要考的那一个角色。想验证"循环有出口",最直接的办法就是造一个永远不满意的评审——如果出口逻辑写错了,这个测试会**挂住不返回**,而不是断言失败,所以它同时也是一个死循环探测器。

`test_评审不通过会重写一次` 里的第三条断言(`"数据来源与时间范围" in out["report"]`)容易被省掉,但它很关键:前两条只证明了"又调了一次撰写",第三条才证明**重写真的按意见改了内容**,而不是原地跑了一遍一样的 prompt。

还有两条守着"输入异常"和"参数纪律":

```python
def test_规划数量异常直接失败而不是默默跑偏(app):
    class TooMany(llm.FakeLLM):
        def chat(self, messages, temperature=0.0):
            if "研究规划师" in messages[0]["content"]:
                import json as _json

                return _json.dumps({"tasks": [f"t{i}" for i in range(20)]}, ensure_ascii=False)
            return super().chat(messages, temperature=temperature)

    llm.reset_llm(TooMany())
    with pytest.raises(ValueError, match="规划结果异常"):
        app.invoke({"question": "测试问题十一"}, cfg())


def test_温度全是零(app, fake_llm):
    """除了报告撰写(0.5), 其它角色必须是 0 —— 结构化输出经不起随机性。"""
    c = cfg()
    app.invoke({"question": "测试问题十二"}, c)
    app.invoke(Command(resume={"action": "approve"}), c)
    for call in fake_llm.calls:
        expected = 0.5 if "报告撰写人" in call["role_hint"] else 0.0
        assert call["temperature"] == expected, f"{call} 温度不对"
```

`test_温度全是零` 看着琐碎,它防的是一类**很难发现的回归**:某天有人为了让报告"更生动"把全局温度调到 0.8,于后规划和评审的 JSON 开始偶发解析失败——而这种失败是概率性的,本地跑十次可能都是绿的。**把配置纪律写成断言,比写在 wiki 里管用。**

### 12.4 只有 Postgres 才能验的两件事

```python
@pytest.mark.skipif(os.getenv("RG_SKIP_PG") == "1", reason="显式跳过需要 Postgres 的测试")
def test_崩溃后续跑不会重跑已完成的节点(fake_llm, monkeypatch):
    """这个测试是整份文件里最重要的一个 —— 它是 checkpointer 存在的理由。

    用真 Postgres 而不是 InMemorySaver: 内存 saver 测不出"换个连接还能读到"。
    """
    from rg.checkpointer import checkpointer

    thread = f"test-{uuid.uuid4().hex[:8]}"
    with checkpointer(setup=True) as cp:
        app = build_graph(cp)
        c = {"configurable": {"thread_id": thread}}
        app.invoke({"question": "崩溃续跑测试"}, c)

        # 让 write 节点崩掉
        monkeypatch.setenv("RG_CRASH_AT", "write")
        with pytest.raises(RuntimeError, match="注入故障"):
            app.invoke(Command(resume={"action": "approve"}), c)

        snap = app.get_state(c)
        assert snap.next == ("write",), "应该停在崩掉的那个节点上, 等着重试"
        assert len(snap.values["findings"]) == 3, "研究阶段的成果必须已经落库"
        researcher_calls = len([x for x in fake_llm.calls if "研究员" in x["role_hint"]])
        assert researcher_calls == 3

        # 修好之后续跑
        monkeypatch.delenv("RG_CRASH_AT")
        out = app.invoke(None, c)
        assert out["status"] == "done"
        # 关键断言: 研究员没有被再调一次 —— 三次昂贵的搜索没有白花
        assert len([x for x in fake_llm.calls if "研究员" in x["role_hint"]]) == 3
        cp.delete_thread(thread)  # 测试要自己收拾垃圾, 否则 checkpoints 表越跑越大


@pytest.mark.skipif(os.getenv("RG_SKIP_PG") == "1", reason="显式跳过需要 Postgres 的测试")
def test_跨连接读回状态(fake_llm):
    """两次独立的连接池, 模拟"服务重启"。"""
    from rg.checkpointer import checkpointer

    thread = f"test-{uuid.uuid4().hex[:8]}"
    with checkpointer(setup=True) as cp:
        build_graph(cp).invoke({"question": "跨进程测试"}, {"configurable": {"thread_id": thread}})

    with checkpointer() as cp2:  # 全新的池
        snap = build_graph(cp2).get_state({"configurable": {"thread_id": thread}})
        assert snap.next == ("approve",)
        assert len(snap.values["tasks"]) == 3
        cp2.delete_thread(thread)
```

三条纪律藏在这两个测试里:

1. **`cp.delete_thread(thread)` 不是可选的。** 我第一版忘了写,跑几十遍之后 `checkpoints` 表里躺着一堆 `test-xxxxxxxx`。测试污染共享资源,下一个人会在排查别的问题时被这些垃圾误导。
2. **`test_跨连接读回状态` 开了两个独立的连接池。** 同一个池里读回来什么都没证明——可能是池的缓存。两个池才能证明"状态真的在 Postgres 里"。
3. **`skipif` 而不是 `try/except`。** 没有 Postgres 时**明确跳过并说明原因**,不要静默通过。`RG_SKIP_PG=1` 是给"我只想快速跑一下逻辑测试"的人留的口子,而 CI 里绝不设这个变量。

### 12.5 tests/test_api.py 与一次自己坑自己的排查

HTTP 层的测试用 `TestClient`,它会走完整的 lifespan,所以 checkpointer 是真连 Postgres 的:

```python
@pytest.fixture
def key() -> str:
    """每个测试一个独立的 API Key。

    这不是洁癖 —— 限流器是**按 Key 计数**的, 容量只有 5。
    所有测试共用一个 Key 时, 跑到第六个测试就开始收 429,
    而失败信息是 `assert 429 == 422`, 看上去像是参数校验坏了。
    我第一版就是这么写的, 排查了几分钟才反应过来是自己的限流在拦自己。
    **测试之间共享任何按标识计数的资源, 迟早会串味。**
    """
    return f"test-key-{uuid.uuid4().hex[:8]}"


@pytest.fixture(autouse=True)
def env(monkeypatch, key):
    monkeypatch.setenv("RG_API_KEYS", key)
    monkeypatch.setenv("RG_MODE", "fake")
    llm.reset_llm(llm.FakeLLM())
    yield
    llm.reset_llm(None)


@pytest.fixture
def client(env):
    from rg.main import app

    with TestClient(app) as c:
        yield c


@pytest.fixture
def hdr(key: str) -> dict:
    return {"X-API-Key": key}
```

这个 fixture 的注释记录了一次真实的排查,值得展开讲,因为**它的失败现象和真实原因毫无关系**。

我第一版在模块顶部写了 `KEY = "test-key-abc"`,所有测试共用。跑出来:

```
7 failed, 19 passed in 0.81s
```

七条失败里五条是 `assert 429 == 422` / `assert 429 == 409` / `assert 429 == 200`,另外两条是 `KeyError: 'status'` 和 `KeyError: 'steps'`——因为 429 的响应体里只有 `detail`,没有业务字段。

看着像什么?像参数校验坏了、像状态机判断错了、像返回体结构变了。**真正的原因是限流器容量 5,而所有测试共用一把 key,从第六个请求开始全部被自己的限流拦下。**

排查这类问题的关键是**先看有没有共享的计数资源**,而不是逐条去读那五个失败的断言。凡是"按某个标识计数"的东西(限流器、配额、去重表、连接池),测试之间共享它就一定会串味,而串味之后的失败信息通常指向完全无关的地方。

剩下的测试覆盖了 HTTP 层该管的三件事:

```python
def test_没配密钥就拒绝启动(monkeypatch):
    """fail closed: 这是整个服务最重要的一条安全属性, 必须有测试守着。"""
    monkeypatch.setenv("RG_API_KEYS", "")
    from rg.main import app

    with pytest.raises(RuntimeError, match="拒绝启动"), TestClient(app):
        pass


def test_鉴权(client, hdr):
    body = {"question": "鉴权测试问题", "thread_id": new_thread()}
    assert client.post("/api/research", json=body).status_code == 401
    assert client.post("/api/research", json=body, headers={"X-API-Key": "wrong"}).status_code == 401
    assert client.post("/api/research", json=body, headers=hdr).status_code == 200


def test_等审批时调_continue_会被挡住(client, hdr):
    """/resume 和 /continue 语义不同, 服务必须替调用方挡住用错的那个。"""
    t = new_thread()
    client.post("/api/research", json={"question": "语义区分测试", "thread_id": t}, headers=hdr)
    r = client.post(f"/api/research/{t}/continue", headers=hdr)
    assert r.status_code == 409
    assert "请用 /resume" in r.json()["detail"]


def test_thread_id_格式被校验(client, hdr):
    """thread_id 是客户端可控的字符串, 会被当成主键写进库 —— 必须白名单校验。"""
    r = client.post(
        "/api/research",
        json={"question": "注入测试问题", "thread_id": "a'; DROP TABLE checkpoints;--"},
        headers=hdr,
    )
    assert r.status_code == 422
```

最后一个 autouse fixture 负责收垃圾:

```python
@pytest.fixture(autouse=True)
def _cleanup():
    """测试自己收拾垃圾: 所有 test-http-* 的 thread 跑完统一删掉。"""
    yield
    if os.getenv("RG_SKIP_PG") == "1":
        return
    from rg.checkpointer import checkpointer

    with checkpointer() as cp:
        for t in {
            c.config["configurable"]["thread_id"]
            for c in cp.list(None, limit=500)
            if c.config["configurable"]["thread_id"].startswith("test-http-")
        }:
            cp.delete_thread(t)
```

**✅ Checkpoint 14**:全套测试跑绿。

```
$ uv run pytest tests -q
...........................                                              [100%]
27 passed in 0.87s

$ uv run pytest tests -q --durations=8
============================= slowest 8 durations ==============================
0.08s call     tests/test_api.py::test_没配密钥就拒绝启动
0.03s teardown tests/test_api.py::test_没配密钥就拒绝启动
0.03s call     tests/test_graph.py::test_崩溃后续跑不会重跑已完成的节点
0.02s call     tests/test_graph.py::test_跨连接读回状态
0.02s call     tests/test_api.py::test_history_能看到每一步
0.02s call     tests/test_api.py::test_跑完之后再_resume_会被挡住
0.02s call     tests/test_api.py::test_stream_逐节点推送
0.02s call     tests/test_api.py::test_审批后跑完
27 passed in 0.79s
```

**27 个测试,0.87 秒,而且里面包含两个真连 Postgres 的。** 这个速度是 `FakeLLM` 换来的:一个打真 API 的等价测试集要跑几分钟、花掉几块钱,还会因为模型抽风随机变红。**"agent 项目测不了"这个说法不成立,前提是你从第一天就留了替身。**

顺带看一下最慢的那条:`test_没配密钥就拒绝启动` 花了 0.08 秒,是全场最慢——因为它要真的启动一次 lifespan 然后失败。最重要的两个 Postgres 测试各 0.02~0.03 秒。

## 13. rg/main.py —— HTTP 层:一个"停得住"的服务(2 小时)

上一节先从客户端一侧看到了这些接口怎么被调用,现在看服务端怎么写。**这一层的设计和普通 CRUD 服务有一处根本不同,值得先说清楚:**

```python
"""FastAPI 服务: 把图的四种能力开成 HTTP 接口。

接口设计上有一个和普通 CRUD 服务不同的地方值得留意:
**这里没有"一个请求跑完一次研究"的接口。** 一次研究会在审批处暂停,
可能几分钟也可能几小时才有人来批。所以对外暴露的是:

  POST /api/research            起一个 thread, 立刻返回待审批的计划
  POST /api/research/{t}/resume 带上审批结果, 继续(可能又停在下一个中断)
  POST /api/research/{t}/continue 崩溃后续跑, 不带人工输入
  GET  /api/research/{t}        当前状态
  GET  /api/research/{t}/history 每一步的快照
  GET  /api/research/{t}/stream?action=approve  SSE: 边跑边推每个节点的产出
  GET  /ui                      静态审批台页面

**thread_id 就是这个长任务的句柄**, 它存在 Postgres 里, 和进程无关 ——
服务重启、换机器、隔一天再来批, 都能接着跑。这是 checkpointer 给的,
不是我们自己写的。
"""
```

"没有一个请求跑完一次研究的接口"这件事,是**长任务和请求-响应模型的根本矛盾**。一次研究要停在人审上,而 HTTP 请求活不了那么久。业界解决它有三种办法:轮询、回调、或者把任务句柄交给客户端。这里选第三种,因为 `thread_id` 天然就是那个句柄——它已经是 checkpointer 的主键了,我们不需要再造一套任务表。

### 13.1 lifespan:连接池的生命周期跟着进程走

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    if not config.service_api_keys():
        raise RuntimeError("未配置 RG_API_KEYS, 拒绝启动(不允许无鉴权运行)")
    # 连接池的生命周期跟着进程走, 不要每个请求新建 —— 建池比跑一次图还慢
    cm = checkpointer(setup=True)
    saver = cm.__enter__()
    STATE["cm"] = cm
    STATE["app"] = build_graph(saver)
    print(f"[boot] 图已编译, checkpointer={config.PG_DSN}, MODE={config.MODE}")
    yield
    cm.__exit__(None, None, None)
    STATE.clear()
```

两条:

- **`raise RuntimeError` 而不是 `print("警告: 未配置密钥")`。** fail closed 的意思是**配错了就起不来**,而不是"起来了但没锁门"。一个能在没有鉴权的情况下正常启动的服务,总有一天会被人在生产环境这么启动一次。
- **手动 `__enter__()` / `__exit__()` 看着别扭,但这里没有更好的写法。** `checkpointer()` 是同步上下文管理器,而 lifespan 是异步的,不能直接 `async with`。把它存在 `STATE` 里、退出时显式关掉,是 langgraph 官方例子里的做法。

`saver` 只在 `build_graph(saver)` 用了一次,`cm` 才是要留着关的那个。

### 13.2 鉴权与限流:两层 Depends

```python
def auth(x_api_key: Annotated[str | None, Header()] = None) -> str:
    if not x_api_key or x_api_key not in config.service_api_keys():
        raise HTTPException(401, "缺少或无效的 X-API-Key")
    return x_api_key


def rate_limit(user: Annotated[str, Depends(auth)]) -> str:
    ok, _ = limiter.allow(user)
    if not ok:
        raise HTTPException(429, "请求过于频繁, 请稍后重试")
    return user
```

`rate_limit` 依赖 `auth`,于是**限流一定发生在鉴权之后**,而且计数用的是通过鉴权的那个 key。反过来写(先限流再鉴权)会让任何人都能用无效 key 打爆你的限流器。

哪些接口用哪个:**读接口用 `auth`,写接口和 SSE 用 `rate_limit`。** `GET /{t}` 和 `/history` 只查库,让人随便刷;`POST` 和 SSE 会真的调模型,必须限。

### 13.3 一个统一的返回体

```python
def _view(thread_id: str, out: dict | None = None) -> dict:
    """统一的返回体: 无论是刚起、刚恢复还是查询, 客户端看到的字段都一样。"""
    snap = _snapshot(thread_id)
    v = snap.values
    itr = (out or {}).get("__interrupt__")
    return {
        "thread_id": thread_id,
        "status": v.get("status"),
        "next": list(snap.next),
        "waiting_for_approval": bool(itr) or "approve" in snap.next,
        "pending": itr[0].value if itr else None,
        "tasks": v.get("tasks", []),
        "findings": [
            {"index": f["index"], "task": f["task"], "sources": f["sources"]}
            for f in sorted(v.get("findings", []), key=lambda f: f["index"])
        ],
        "review": v.get("review"),
        "report": v.get("report"),
    }
```

三个设计点:

1. **`waiting_for_approval` 是给客户端算好的。** 客户端不该被要求理解 `next` 元组里有没有 `"approve"` 这种框架细节。`bool(itr) or "approve" in snap.next` 同时覆盖了"这次调用刚中断"和"上次就停在那儿"两种情况。
2. **`findings` 里不返回 `summary`。** 每条 summary 是几百字,三条就把响应撑到几 KB,而客户端要的是进度。完整内容在 `report` 里。**返回体的字段是接口契约的一部分,不是 state 的镜像。**
3. **`sorted(..., key=lambda f: f["index"])`。** 并行完成顺序是乱的,而客户端会按数组顺序渲染。排序放在这一层,前端就不用操心。

`_snapshot()` 顺手把 404 处理了:

```python
def _snapshot(thread_id: str):
    snap = STATE["app"].get_state(_cfg(thread_id))
    if snap.created_at is None:
        raise HTTPException(404, f"thread {thread_id} 不存在")
    return snap
```

**`snap.created_at is None` 是"这个 thread 不存在"的判断方式**,不是 `get_state` 抛异常。langgraph 对一个没见过的 thread_id 会返回一个空快照而不是报错——这是个容易踩的坑,不显式判断的话,查一个不存在的 thread 会返回 `{"status": null, "next": []}` 而不是 404。

### 13.4 三个写接口

```python
@app.post("/api/research")
def start(req: StartRequest, user: Annotated[str, Depends(rate_limit)]) -> dict:
    snap = STATE["app"].get_state(_cfg(req.thread_id))
    if snap.created_at is not None:
        # thread_id 由客户端指定, 所以必须防重: 复用一个已有 id 会把两次研究搅在一起
        raise HTTPException(409, f"thread {req.thread_id} 已存在, 请换一个 id")
    out = STATE["app"].invoke({"question": req.question}, _cfg(req.thread_id))
    return _view(req.thread_id, out)
```

```python
@app.post("/api/research/{thread_id}/resume")
def resume(thread_id: str, req: ResumeRequest, user: Annotated[str, Depends(rate_limit)]) -> dict:
    snap = _snapshot(thread_id)
    if "approve" not in snap.next:
        raise HTTPException(409, f"thread {thread_id} 当前不在等待审批(next={list(snap.next)})")
    payload = {"action": req.action}
    if req.action == "edit":
        if not req.tasks:
            raise HTTPException(422, "action=edit 必须给出 tasks")
        payload["tasks"] = req.tasks
    if req.action == "reject":
        payload["comment"] = req.comment or ""
    out = STATE["app"].invoke(Command(resume=payload), _cfg(thread_id))
    return _view(thread_id, out)


@app.post("/api/research/{thread_id}/continue")
def cont(thread_id: str, user: Annotated[str, Depends(rate_limit)]) -> dict:
    """崩溃后续跑。注意它和 resume 不能互换 —— 见 rg/cli.py 里的说明。"""
    snap = _snapshot(thread_id)
    if not snap.next:
        raise HTTPException(409, f"thread {thread_id} 已结束, 无待执行节点")
    if "approve" in snap.next:
        raise HTTPException(409, "该 thread 在等人审批, 请用 /resume 而不是 /continue")
    out = STATE["app"].invoke(None, _cfg(thread_id))
    return _view(thread_id, out)
```

**这两个接口里的 409 是这一节最该抄走的东西。** 第 10 节讲过 `Command(resume=...)` 和 `invoke(None)` 用错时会**静默地走错路**:对一个等审批的 thread 调 `invoke(None)`,approve 节点重跑、再次中断,调用方拿到一个"看起来正常"的响应,却什么都没推进。

CLI 里我们靠注释提醒自己;**在 HTTP 层要靠 409 挡住别人。** 接口是给别的团队用的,他们不会读你的注释。凡是"用错了不会报错、只是不生效"的语义差异,都值得在边界上加一次显式检查。

输入校验全靠 Pydantic:

```python
class StartRequest(BaseModel):
    question: str = Field(min_length=4, max_length=300)
    thread_id: str = Field(min_length=1, max_length=64, pattern=r"^[A-Za-z0-9_-]+$")
```

`pattern` 那条不是洁癖:**`thread_id` 会被当成主键写进 Postgres,而它完全由客户端提供。** psycopg 的参数化已经挡住了注入,但白名单是第二道门,而且顺手排除了带空格、斜杠、中文的 id——那些 id 能存进去,但会让日志、URL 和运维脚本都变得难处理。

### 13.5 SSE:一个被失败测试改掉的接口设计

```python
@app.get("/api/research/{thread_id}/stream")
def stream(
    thread_id: str,
    user: Annotated[str, Depends(rate_limit)],
    action: Literal["approve", "reject"] | None = None,
) -> StreamingResponse:
    """SSE: 从当前位置往下跑, 每个节点跑完推一条。

    stream_mode="updates" 推的是"哪个节点返回了什么", 正好对应前端要的进度条。
    另外两种模式:values 推每步之后的完整状态(体积大), debug 推底层任务事件。
    """
    snap = _snapshot(thread_id)
    if not snap.next:
        raise HTTPException(409, f"thread {thread_id} 已结束")
    waiting = "approve" in snap.next
    if waiting and action is None:
        raise HTTPException(409, "该 thread 在等人审批, 请带上 ?action=approve|reject")
    payload = Command(resume={"action": action, "comment": ""}) if waiting else None
    q: queue.Queue = queue.Queue()

    def worker() -> None:
        try:
            for chunk in STATE["app"].stream(payload, _cfg(thread_id), stream_mode="updates"):
                for node, upd in chunk.items():
                    if node == "__interrupt__":
                        q.put({"type": "interrupt", "value": upd[0].value})
                        continue
                    q.put({"type": "node", "node": node, "keys": sorted((upd or {}).keys())})
        except Exception as e:
            q.put({"type": "error", "detail": f"{type(e).__name__}: {e}"})
        q.put(None)

    threading.Thread(target=worker, daemon=True).start()

    def events():
        while True:
            ev = q.get()
            if ev is None:
                break
            yield f"data: {json.dumps(ev, ensure_ascii=False)}\n\n"

    return StreamingResponse(events(), media_type="text/event-stream")
```

**那个 `?action=` 参数是被一个失败的测试逼出来的,这段经历比接口本身有价值。**

我的第一版没有 `action`,接口里只有一句 `app.stream(None, cfg, stream_mode="updates")`——理由听起来很顺:流式接口就是"从当前位置往下跑",不该管人工输入。然后我写了这个测试:

```python
def test_stream_逐节点推送(client, hdr):
    t = new_thread()
    client.post("/api/research", json={"question": "流式测试", "thread_id": t}, headers=hdr)
    with client.stream("GET", f"/api/research/{t}/stream?action=approve", headers=hdr) as r:
        lines = [ln for ln in r.iter_lines() if ln.startswith("data: ")]
    assert any('"node": "research"' in ln or '"node":"research"' in ln for ln in lines)
```

红了:`lines` 里只有一条 `interrupt` 事件,一个 `research` 节点都没有。

原因就是第 10 节那张表里的区别:**thread 停在 approve 的中断上,`invoke(None)` 会让 approve 从头重跑,再次 `interrupt()`。** 图纹丝不动,而流"正常结束"了。

这里有两条路:把测试改成"断言收到一条 interrupt"(承认这个接口对等审批的 thread 没用),或者改接口。我改了接口——因为**研究阶段是整条链路上唯一值得看进度条的一段,而它恰好在审批之后。** 一个"只能在审批之后用、但没法把审批结果带进来"的流式接口,等于没有流式接口。

于是接口最终的注释是这样的:

```
为什么要有 ?action= 这个查询参数: 研究阶段是**唯一值得看进度条的一段**,
而它在审批之后。如果这个接口只会 `invoke(None)`, 那么对一个停在审批上的
thread 调它, approve 节点会从头重跑、再次 interrupt —— 你只能收到一条
interrupt 事件, 一个研究节点都推不出来。这不是 bug, 是把"恢复"和"续跑"
当成同一件事的后果(见 rg/cli.py)。所以: 等审批时必须带 action, 由这里
翻译成 Command(resume=...); 崩溃续跑时不带, 走 invoke(None)。
SSE 只能用 GET, 所以人工输入只能从查询串进来 —— 需要 edit 改计划时
请用 POST /resume, 那条路径不适合做流式。
```

顺带加了第二个测试,把"不带 action 会怎样"钉住:

```python
def test_stream_等审批时不带_action_会被挡住(client, hdr):
    """不带 action 就 invoke(None): approve 会重跑并再次 interrupt, 一个研究节点都推不出来。"""
    ...
    assert r.status_code == 409
    assert "action" in r.json()["detail"]
```

**一个失败的测试推动了一次接口重新设计,这才是给 agent 写测试的真正回报。** 如果我当时把断言改软,这个接口会带着"对等审批的 thread 实际不可用"的缺陷发布,而且没人会发现——它不报错,只是不干活。

`edit` 走不了 SSE 是这个方案的代价:SSE 只能 GET,而一整份改过的计划塞不进查询串。这个取舍我接受了,并且写在注释里——**把方案的边界写清楚,比假装它没有边界好。**

技术上还有两处值得说:

- **`queue.Queue` + 后台线程,而不是直接在生成器里 `for chunk in app.stream(...)`。** 图是同步阻塞的,直接在生成器里迭代会把它跑在 FastAPI 的线程池里,一个慢 thread 能占着不放。放进后台线程、主体只从队列取,生成器就随时可以被客户端断开而不卡住。
- **`except Exception` 把错误变成一条 `error` 事件。** SSE 的响应头早就发出去了,这时候抛异常客户端只会看到连接莫名断掉。把异常变成流里的一条事件,前端才能显示"✗ 节点崩了"。

### 13.6 实测这一层

**✅ Checkpoint 15**:启动服务并验鉴权。

```
$ RG_API_KEYS=rg-key-1,rg-key-2 uv run uvicorn rg.main:app --port 8020
[boot] 图已编译, checkpointer=postgresql://rg:rg@127.0.0.1:5432/rg, MODE=fake
INFO:     Uvicorn running on http://127.0.0.1:8020 (Press CTRL+C to quit)

$ curl -s localhost:8020/health
{"status":"ok","mode":"fake"}

$ # 不带 key / 错 key / 对 key
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8020/api/research \
    -H 'Content-Type: application/json' -d '{"question":"鉴权探针问题","thread_id":"h-auth-1"}'
401
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8020/api/research \
    -H 'X-API-Key: nope' -H 'Content-Type: application/json' \
    -d '{"question":"鉴权探针问题","thread_id":"h-auth-1"}'
401
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8020/api/research \
    -H 'X-API-Key: rg-key-1' -H 'Content-Type: application/json' \
    -d '{"question":"鉴权探针问题","thread_id":"h-auth-1787020834"}'
200
```

> 顺带记一个自己坑自己的小插曲:我一开始想用一个 shell 循环把三种 header 一起打完,写成 `${h:+-H "$h"}`,结果打印出 `401/400/400`——空 header 那次被 shell 拆坏了,curl 发出了一个畸形请求头。换成三条显式的 curl 之后才是 `401/401/200`。**验证脚本本身出的错,最容易被当成被测代码的错。**

**✅ Checkpoint 16**:四种 409/422。

```
$ # 1. 重复的 thread_id
{"detail":"thread h-auth-1787020834 已存在, 请换一个 id"}   # 409

$ # 2. 等审批时调 /continue
{"detail":"该 thread 在等人审批, 请用 /resume 而不是 /continue"}   # 409

$ # 3. 已经跑完了再调 /resume
{"detail":"thread h-auth-1787020834 当前不在等待审批(next=[])"}   # 409

$ # 4. 非法 thread_id
{"detail":[{"type":"string_pattern_mismatch",
            "loc":["body","thread_id"],
            "msg":"String should match pattern '^[A-Za-z0-9_-]+$'",
            "input":"a'; DROP TABLE checkpoints;--"}]}          # 422
```

第 2 条和第 3 条是这一层真正的价值:**用错了会被明确拒绝,而不是拿到一个看起来正常的响应。**

**✅ Checkpoint 17**:一条完整的 HTTP 链路。

```
$ curl -s -X POST localhost:8020/api/research -H 'X-API-Key: rg-key-2' \
    -H 'Content-Type: application/json' \
    -d '{"question":"2025 年边缘侧推理进展","thread_id":"h-flow-1787020897"}' | jq
{
  "status": "planned",
  "next": ["approve"],
  "waiting_for_approval": true,
  "pending": {
    "question": "2025 年边缘侧推理进展",
    "tasks": [
      "子任务A: 2025 年边缘侧推理进展 的技术进展",
      "子任务B: 2025 年边缘侧推理进展 的代表性产品",
      "子任务C: 2025 年边缘侧推理进展 的公开数据"
    ],
    "plan_round": 1,
    "hint": "回复 {'action':'approve'} / ..."
  }
}

$ curl -s -X POST localhost:8020/api/research/h-flow-1787020897/resume \
    -H 'X-API-Key: rg-key-2' -H 'Content-Type: application/json' \
    -d '{"action":"approve"}' | jq '{status,next,waiting_for_approval}'
{ "status": "done", "next": [], "waiting_for_approval": false }

findings: 3 条
  [0] 子任务A: ... 来源 3 条
  [1] 子任务B: ... 来源 3 条
  [2] 子任务C: ... 来源 3 条
review: {"score": 9, "issues": []}
report: 165 字
```

两次 POST 就跑完了一次带人审的研究,而且**第二次 POST 完全可以来自另一台机器、另一个进程、隔一天再发。**

`GET /history` 的输出和第 10 节 CLI 那份一模一样(10 步),因为它们读的是同一张表:

```
$ curl -s localhost:8020/api/research/h-flow-1787020897/history -H 'X-API-Key: rg-key-2' | jq -r ...
step  ran                        next                             findings
   -1 __start__                  ('plan',)                               0
    0 plan                       ('approve',)                            0
    1 approve                    ('research','research','research')      0
    2 research,research,research ('write',)                              3
    3 write                      ('review',)                             3
    4 review                     ('write',)                              3
    5 write                      ('review',)                             3
    6 review                     ('finalize',)                           3
    7 finalize                   ()                                      3
count = 10
```

**✅ Checkpoint 18**:SSE 逐节点推送。

```
$ # 不带 action
$ curl -s localhost:8020/api/research/h-sse-1/stream -H 'X-API-Key: rg-key-2'
{"detail":"该 thread 在等人审批, 请带上 ?action=approve|reject"}   # 409

$ # 带上 action
$ curl -sN "localhost:8020/api/research/h-sse-1/stream?action=approve" -H 'X-API-Key: rg-key-2'
data: {"type": "node", "node": "approve", "keys": ["approval", "status"]}
data: {"type": "node", "node": "research", "keys": ["findings"]}
data: {"type": "node", "node": "research", "keys": ["findings"]}
data: {"type": "node", "node": "research", "keys": ["findings"]}
data: {"type": "node", "node": "write", "keys": ["report", "status"]}
data: {"type": "node", "node": "review", "keys": ["review", "review_round", "status"]}
data: {"type": "node", "node": "write", "keys": ["report", "status"]}
data: {"type": "node", "node": "review", "keys": ["review", "review_round", "status"]}
data: {"type": "node", "node": "finalize", "keys": ["status"]}
```

九条事件,把整张图的走法完整暴露出来了:approve → research×3 → write → review → **write → review**(评审 5 分,重写一轮)→ finalize。`keys` 字段顺带证明了"节点只返回要改的字段"这条纪律——`research` 只返回 `findings`,不返回整个 state。

> 排查时踩的一个坑:某次 resume 的响应解析报 `KeyError: 'status'`,我以为返回体结构变了。实际响应体是 `{"detail":"请求过于频繁, 请稍后重试"}`——`rg-key-1` 的 5 个令牌在前面的探针里烧完了。换成 `rg-key-2` 并在各段之间 `sleep 12` 才干净。**和第 12 节测试里那个 429 是同一个坑,连现象(KeyError 而不是"限流")都一样。**

## 14. static/index.html —— 一个五十行的审批台(40 分钟)

审批这件事最终得有人来点。给运营同学一份 curl 命令是不现实的,所以配一个页面:

```python
# 审批台页面。挂在 /ui 而不是 /, 这样根路径留给以后的重定向或版本页。
# html=True 让 /ui 直接返回 index.html, 不用自己写一个读文件的路由。
# 注意: 这个页面本身不需要鉴权(它是一坨静态 HTML, 没有任何数据),
# 真正的门在它调用的那些 /api 上 —— key 是用户自己填进输入框的。
_STATIC = Path(__file__).resolve().parent.parent / "static"
if _STATIC.is_dir():
    app.mount("/ui", StaticFiles(directory=_STATIC, html=True), name="ui")
```

`if _STATIC.is_dir()` 这个判断是给容器留的:镜像里如果没 COPY static 目录,服务照样起得来,只是没有 UI。**可选组件缺失不该让主服务起不来。**

页面本体是一坨没有任何构建步骤的 HTML,关键就三个函数。第一个起研究:

```javascript
async function start() {
  $('log').textContent = ''; $('report').textContent = '';
  const r = await fetch('/api/research', {
    method: 'POST', headers: hdr(),
    body: JSON.stringify({question: $('q').value, thread_id: $('tid').value}),
  });
  const d = await r.json();
  if (!r.ok) return log('✗ ' + JSON.stringify(d));
  $('tasks').value = d.pending.tasks.join('\n');
  log(`已生成第 ${d.pending.plan_round} 轮计划, 共 ${d.pending.tasks.length} 个子任务, 等你审批`);
}
```

`d.pending.tasks.join('\n')` 把待审批的计划填进一个 `textarea` —— **审批界面的"编辑"能力就是这么来的:让人直接改那几行文字,而不是给每个任务做一个输入框加删除按钮。** 一次研究的计划只有 3~5 行,纯文本足够,而且改起来比点按钮快。

第二个是流式跑图,也是整个页面唯一有技术含量的地方:

```javascript
// 批准/驳回走 SSE(能看进度条), edit 走 POST /resume ——
// SSE 只能 GET, 塞不下一整份改过的计划。
//
// 这里没用 EventSource: 它不支持自定义请求头, 而我们的鉴权在 X-API-Key 上。
// 用 fetch + ReadableStream 手动切 `data: ` 行, 代码只多五行, 但不用为了
// 一个前端 API 的限制去把鉴权改成查询参数(那会让 key 落进日志和浏览器历史)。
async function go(action) {
  const r = await fetch(`/api/research/${$('tid').value}/stream?action=${action}`, {headers: hdr()});
  if (!r.ok) return log('✗ ' + JSON.stringify(await r.json()));
  const reader = r.body.pipeThrough(new TextDecoderStream()).getReader();
  let buf = '';
  for (;;) {
    const {value, done} = await reader.read();
    if (done) break;
    buf += value;
    const parts = buf.split('\n\n');
    buf = parts.pop();
    for (const p of parts) {
      if (!p.startsWith('data: ')) continue;
      const ev = JSON.parse(p.slice(6));
      if (ev.type === 'node') log(`▸ ${ev.node} 完成 → ${ev.keys.join(', ')}`);
      else if (ev.type === 'interrupt') log('⏸ 又停在人审上: ' + JSON.stringify(ev.value.tasks));
      else log('✗ ' + ev.detail);
    }
  }
  refresh();
}
```

**`EventSource` 不能带自定义请求头,这是个绕不过去的浏览器 API 限制。** 常见的"解决办法"是把 key 挪到查询参数里(`?key=xxx`),然后 `new EventSource(url)` 就能用了。别这么干:**查询串会进 access log、进浏览器历史、进 Referer 头**,一个能起研究、能烧模型额度的 key 就这么散落到三个地方。

正确的做法是自己解析 SSE,而它比想象中简单:SSE 的格式就是"以 `\n\n` 分隔的块,每块里 `data: ` 开头的行是负载"。`buf += value; const parts = buf.split('\n\n'); buf = parts.pop();` 这三行是**流式解析的标准骨架**——最后一段可能是半条,留在 buffer 里等下一片数据。多写五行,换掉一个真实的安全问题,这个交易很划算。

`buf = parts.pop()` 容易写错成 `parts.pop()` 之后忘了赋值回去,那样在网络分片刚好切在一条事件中间时会丢事件——而这种 bug 在本地开发时几乎不会出现(本地一次就读完了),只在真实网络下偶发。

第三个函数处理 edit/reject,走普通 POST:

```javascript
async function resume(payload) {
  log('提交 ' + payload.action + ' ...');
  const r = await fetch(`/api/research/${$('tid').value}/resume`, {
    method: 'POST', headers: hdr(), body: JSON.stringify(payload),
  });
  const d = await r.json();
  if (!r.ok) return log('✗ ' + JSON.stringify(d));
  if (d.waiting_for_approval) {
    $('tasks').value = d.pending.tasks.join('\n');
    log(`重新规划完成(第 ${d.pending.plan_round} 轮), 请再审一次`);
  } else render(d);
}
```

`if (d.waiting_for_approval)` 那个分支是驳回的正常路径:**驳回之后图会重新规划,然后又停在审批上,于是要把新计划填回 textarea 让人再审一次。** 第 13.3 节把 `waiting_for_approval` 算好了返回,前端这里就是一个 if——这就是"返回体不做 state 的镜像"换来的好处。

还有一行不起眼但省事:

```javascript
$('tid').value = 'ui-' + Math.random().toString(36).slice(2, 8);
```

页面加载就生成一个随机 `thread_id`,因为 `POST /api/research` 对重复 id 返回 409。**凡是"客户端提供主键 + 服务端防重"的设计,客户端都该默认生成一个,而不是让人手填。**

**✅ Checkpoint 19**:页面能打开。

```
$ curl -s -o /dev/null -w '%{http_code} %{content_type} %{size_download}\n' localhost:8020/ui
307  0
$ curl -sL -o /dev/null -w '%{http_code} %{content_type} %{size_download}\n' localhost:8020/ui
200 text/html; charset=utf-8 4754
$ curl -s -o /dev/null -w '%{http_code}\n' localhost:8020/ui/
200
```

`/ui` 返回 **307 而不是 200**,一开始我以为挂载错了。不是:`StaticFiles(html=True)` 会把 `/ui` 重定向到 `/ui/` 再返回 index.html,这是它的正常行为。加 `-L` 跟随重定向就看到 200 和 4754 字节。**看到 3xx 先想"这是不是框架的正常跳转",别急着改代码。**

浏览器里打开 `http://127.0.0.1:8020/ui`,填上 key、点"1. 起研究",计划出现在 textarea 里;点"批准并开始",黑框里一行行滚出来:

```
已生成第 1 轮计划, 共 3 个子任务, 等你审批
▸ approve 完成 → approval, status
▸ research 完成 → findings
▸ research 完成 → findings
▸ research 完成 → findings
▸ write 完成 → report, status
▸ review 完成 → review, review_round, status
▸ write 完成 → report, status
▸ review 完成 → review, review_round, status
▸ finalize 完成 → status
状态=done 发现=3 评分=9
```

## 15. 交付:Dockerfile、compose、CI(1 小时)

### 15.1 Dockerfile

```dockerfile
# 用官方 uv 镜像: 它把 uv 装好了, 省掉一层 curl 安装
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

WORKDIR /app

# 依赖单独一层: 只改代码时这一层命中缓存, 重建从 40s 降到 2s
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project

COPY rg ./rg
COPY tests ./tests
COPY static ./static

# 默认假模型: 镜像跑起来就能用, 不需要任何 key。
# 真跑要在 compose/部署时传 RG_MODE=real + LLM_API_KEY。
ENV RG_MODE=fake \
    PYTHONUNBUFFERED=1 \
    PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

# --workers 1 是刻意的: 图的 checkpointer 连接池跟着进程走,
# 多 worker 会各自建池(能跑, 但连接数翻倍且日志难看)。
# 要横向扩就上多个容器, 让 Postgres 做唯一的共享状态。
CMD ["uvicorn", "rg.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

四个决定:

- **`COPY pyproject.toml uv.lock` 单独一层。** Docker 的层缓存按指令+文件内容算,把依赖安装放在 `COPY rg ./rg` 之前,改代码时这一层直接命中。这是 Dockerfile 最基本也最容易漏的优化。
- **`ENV RG_MODE=fake`。** 镜像的默认行为应该是"拉下来就能跑",而不是"必须先配一堆 key"。要真跑,在部署时覆盖。
- **`COPY tests ./tests`。** 有人会说测试不该进生产镜像。这里进了,因为**能在部署环境里 `docker compose exec api pytest tests -q` 跑一遍烟测,价值大于那几十 KB。**
- **`--workers 1`,横向扩靠多容器。** 这是这一篇整个架构的收口:**所有共享状态都在 Postgres 里,进程本身是无状态的。** 于是"扩容"就是多起几个容器,不需要任何粘性会话——一个 thread 可以在容器 A 上起、在容器 B 上审批、在容器 C 上续跑。

那句 `--workers 1` 的注释值得再说一遍:多 worker 不是**不能**跑,而是每个 worker 各自建一个连接池,`max_size=5` 就变成了 `worker 数 × 5`。Postgres 的 `max_connections` 默认 100,几个多 worker 的容器就能把它吃满,而症状是"偶发的连不上数据库"。

### 15.2 docker-compose.yml

```yaml
# 一条命令把三样东西拉起来: Postgres(checkpointer) + Redis(限流) + 服务本体。
#
# 注意 depends_on 用的是 service_healthy 而不是 service_started ——
# Postgres 容器"起来了"和"能接受连接了"差好几秒, 后者才是我们需要的条件。
# 少了 condition, 服务会在启动时 setup() 失败并直接退出。
services:
  pg:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: rg
      POSTGRES_PASSWORD: rg
      POSTGRES_DB: rg
    ports:
      - "5432:5432"
    volumes:
      - rg_pg:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rg -d rg"]
      interval: 3s
      timeout: 3s
      retries: 20

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 20

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      # 容器之间用服务名互相寻址, 不是 127.0.0.1
      RG_PG_DSN: postgresql://rg:rg@pg:5432/rg
      RG_REDIS_URL: redis://redis:6379/1
      RG_MODE: ${RG_MODE:-fake}
      # 没有默认值: 忘了配就启动失败, 这正是我们要的行为
      RG_API_KEYS: ${RG_API_KEYS:?请先在 .env 或环境里设置 RG_API_KEYS}
      LLM_API_KEY: ${LLM_API_KEY:-}
      LLM_BASE_URL: ${LLM_BASE_URL:-https://api.deepseek.com/v1}
      LLM_MODEL: ${LLM_MODEL:-deepseek-chat}
    depends_on:
      pg:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "python -c \"import httpx;httpx.get('http://127.0.0.1:8000/health').raise_for_status()\""]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s

volumes:
  rg_pg:
```

三处细节:

**`condition: service_healthy` 不是可选的。** 默认的 `depends_on: [pg]` 只保证"pg 容器已创建",而 Postgres 从进程启动到能接受连接要好几秒。少了这个条件,`api` 会在 lifespan 里 `setup()` 失败然后直接退出——现象是"compose up 之后 api 容器反复重启",而 pg 的日志一切正常。

**`${RG_API_KEYS:?...}` 里的 `:?` 是 compose 层的 fail closed。** 它和 lifespan 里那句 `raise RuntimeError` 是同一条纪律的两层:compose 这层在**容器还没起**的时候就报错,报错信息里还能写中文提示。

**`${RG_MODE:-fake}` 用 `:-`(有默认值),`RG_API_KEYS` 用 `:?`(无默认值,必填)。** 这两个符号的区别就是"这个配置忘了配会不会出安全问题"的判断结果。

**✅ Checkpoint 20**:验证 compose 配置,包括它该失败的时候真的失败。

```
$ RG_API_KEYS=demo docker compose config -q && echo "compose 配置合法"
compose 配置合法

$ RG_API_KEYS= docker compose config -q
error while interpolating services.api.environment.RG_API_KEYS: required variable
RG_API_KEYS is missing a value: 请先在 .env 或环境里设置 RG_API_KEYS
```

**第二条命令才是这个 Checkpoint 的重点。** 只验证"配置合法"证明不了 fail closed 生效——必须验证**忘配的时候真的起不来**。这是这一篇反复出现的一条:安全属性要用"它该失败时失败了"来验证,不是用"它正常时正常"。

### 15.3 CI

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    # CI 里起真 Postgres 和 Redis, 而不是把测试降级成"只跑内存版"。
    # 崩溃续跑那两个测试是整个项目的核心卖点, 在 CI 里跳过等于没测。
    services:
      pg:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: rg
          POSTGRES_PASSWORD: rg
          POSTGRES_DB: rg
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U rg -d rg"
          --health-interval 3s
          --health-timeout 3s
          --health-retries 20
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 3s
          --health-timeout 3s
          --health-retries 20

    env:
      RG_MODE: fake              # CI 绝不打真 API: 慢、贵、还会因为模型抽风而随机红
      RG_PG_DSN: postgresql://rg:rg@127.0.0.1:5432/rg
      RG_REDIS_URL: redis://127.0.0.1:6379/1
      RG_API_KEYS: ci-key

    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - run: uv sync --frozen

      - name: Lint
        run: |
          uvx ruff check .
          uvx ruff format --check .

      - name: Test
        run: uv run pytest tests -q
```

两条:

**CI 里不设 `RG_SKIP_PG`。** 第 12.4 节那两个测试是这个项目的核心卖点,而它们只在有 Postgres 时才跑。如果 CI 图省事不起服务容器,那两条会静默跳过,`27 passed` 变成 `25 passed, 2 skipped`——而没人会注意到那个 `2 skipped`。**跳过的测试等于没有的测试,而且比没有更危险,因为它让你以为覆盖了。**

**`RG_MODE: fake`。** CI 绝不打真 API:慢、花钱、而且会因为模型的随机性让构建随机变红。一个会随机红的 CI,一个月之后就没人看了。

**✅ Checkpoint 21**:本地把 CI 的三步跑一遍。

```
$ uvx ruff check .
All checks passed!

$ uvx ruff format --check .
15 files already formatted

$ uv run pytest tests -q
27 passed in 0.87s
```

`ruff` 这两步我都吃过一次:
- `ruff check` 报 `F401 'json' imported but unused` —— `rg/cli.py:17`,重构时留下的。`uvx ruff check --fix .` 直接修掉。
- `ruff format --check` 说 4 个文件要重排(比如 `format_results` 里的生成器被折成一行、`@pytest.mark.skipif` 的装饰器被收成一行)。跑 `uvx ruff format .` 之后再 `check` + `pytest`,都干净。

**格式化放进 CI 的价值不在"代码好看",在于把"格式"这个话题从 code review 里彻底删掉。**

### 15.4 项目最终长这样

```
research_graph/
├── rg/
│   ├── config.py         51 行   环境变量与常量
│   ├── graph.py         238 行   ← 心脏: 节点 + 边 + 组装
│   ├── state.py          39 行   状态定义与 reducer
│   ├── cli.py           263 行   六个子命令: start/status/resume/continue/history/fork
│   ├── main.py          241 行   FastAPI: 7 个接口
│   ├── llm.py           180 行   真模型 + FakeLLM
│   ├── search.py         62 行   搜索工具 + 假搜索
│   ├── checkpointer.py   65 行   Postgres 连接池
│   ├── ratelimit.py      48 行   Redis 令牌桶
│   └── why_reducer.py    56 行   一个证明 reducer 必要性的小实验
├── tests/
│   ├── test_graph.py    190 行   13 个测试: 控制流、并行、循环、崩溃续跑
│   └── test_api.py      245 行   14 个测试: 鉴权、409 语义、SSE
├── static/index.html            审批台
├── Dockerfile / docker-compose.yml / .github/workflows/ci.yml
└── pyproject.toml / uv.lock

总计约 1678 行, 27 个测试, 0.87 秒跑完。
```

对着第 0 节那段 `run_research()` 看一眼:**原来那 30 行函数干的事,现在散在 `graph.py` 的 238 行里,换来了断点续跑、人在环中、时间旅行、和 27 个能在 0.87 秒内跑完的测试。** 这个交换划不划算,取决于这个 agent 要不要上线。只在自己电脑上跑一次的东西,30 行那版更好;要交给别人用、要在半夜崩了之后能接着跑的,就得是这一版。

## 16. 踩坑与专家提示

这一篇踩的坑和上一篇的性质不一样。上一篇多数是"配置写错、字段记错"这类;这一篇多数是**对框架的语义理解错了**,而它们的共同特征是:**不报错,只是不对**。

### 坑 1:我把"框架会静默覆盖"当成事实写进了文档

我原本打算在第 8 节这样写:"不给 `findings` 加 reducer,三个并行节点的结果会互相覆盖,最后只剩一条"。这是我从别处读来的说法,听着也合理。

然后我跑了 `rg/why_reducer.py`,langgraph 1.2 抛的是:

```
langgraph.errors.InvalidUpdateError: At key 'findings': Can receive only one value per step.
Use an Annotated key to handle multiple values.
```

**它不静默覆盖,它直接报错,而且错误信息里就写了怎么修。** 这比静默覆盖好得多——一个会报错的设计缺陷,你在第一次跑的时候就知道了。

**教训**:任何"框架会怎样"的断言,自己跑一遍再往文档里写。二手结论的保鲜期比你想的短,尤其在版本迭代快的框架上。

### 坑 2:`metadata["writes"]` 不存在

写 `cmd_history` 时我想打印"这一步跑了哪个节点",顺手写了 `s.metadata["writes"].keys()`。跑出来一片 `(loop)`,因为 `.get("writes")` 返回 `None` 走了兜底分支。

langgraph 1.2 的 `StateSnapshot.metadata` 只有三个字段:`step`、`source`、`parents`。**"哪个节点跑了"这个信息根本不在快照自己身上**——它在**上一个快照的 `.next` 里**:

```python
"ran": list(chrono[i - 1].next) if i > 0 else [s.metadata.get("source")],
```

**教训**:遇到"某个字段是 None"的时候,先 `print(dict(snap.metadata))` 看看到底有什么,不要凭印象猜 schema。猜错的成本是一个静默走兜底分支的 bug。

### 坑 3:`invoke(None)` 和 `Command(resume=...)` 用错时不报错

这是这一篇最值钱的一个坑,前面出现了三次(第 10 节的表、第 13.4 节的 409、第 13.5 节的 SSE 重设计),这里收口:

| | `invoke(None, cfg)` | `invoke(Command(resume=X), cfg)` |
|---|---|---|
| 语义 | 从 `next` 指的节点继续跑 | 回答一个正在等待的 `interrupt()` |
| 用在 | 崩溃/超时之后续跑 | 人审完了、要把结果送进去 |
| 用错的后果 | 对等审批的 thread:approve 重跑 → 再次 interrupt → **图纹丝不动** | 对没在等待的 thread:resume 值被丢弃 |
| 会报错吗 | **不会** | **不会** |

"不会报错"这一行是关键。**用错了你会拿到一个 HTTP 200、一个格式正确的响应体、和一张一步都没往前走的图。** 我是靠一个断言 `research` 节点出现过的测试才发现的。

**教训**:框架里凡是"两个 API 长得像、用错不报错"的地方,都要在自己的边界上加一次显式检查。这一篇的做法是 HTTP 层的两个 409,和 CLI 里两个子命令的分工。

### 坑 4:`interrupt()` 前面的代码会执行两次

`approve` 节点在恢复时**从头重跑**,`interrupt()` 之前的语句因此执行两遍。写成这样就出事了:

```python
def approve(state):
    send_email_to_boss(state["tasks"])     # ← 老板会收到两封
    decision = interrupt({...})
    return {...}
```

**副作用必须放在 `interrupt()` 之后。** 这不是 bug,是 checkpointer 的工作方式决定的:节点是最小的重放单位,而不是语句。

**教训**:凡是"可恢复"的执行模型,节点/任务就必须是**幂等**的。这条在 Celery、Temporal、Airflow 里是同一条,不是 langgraph 特有的。

### 坑 5:`FakeLLM` 用计数器决定返回什么,一并发就串味

我第一版 `FakeLLM` 里有个 `self.review_round` 计数器,用来让"第一次评审给 5 分、第二次给 9 分"。测试里跑多个 thread 时,这个计数器是**共享的**,于是第二个 thread 的第一次评审直接拿到 9 分,重写循环压根没触发,`test_评审不通过会重写一次` 随机变红。

改成**按内容判断**:看 prompt 里的报告有没有包含上一轮加进去的章节标题。

**教训**:测试替身的行为要由**输入**决定,不能由**调用次数**决定。计数器型替身在任何并发或多用例场景下都会串味,而且失败是随机的。

### 坑 6:所有测试共用一把 API Key,被自己的限流打死

现象是 `7 failed`,里面有 `assert 429 == 422`、`assert 429 == 409`,还有两个 `KeyError: 'status'`。看着像参数校验坏了、像状态机判断错了、像返回体结构变了。

真实原因:限流器容量 5,所有测试共用一把 key,第六个请求开始全被自己的限流拦下。

同一个坑在手动验证时又踩了一次:`rg-key-1` 的令牌在前面的 curl 探针里烧完,后面一条 resume 解析报 `KeyError: 'status'`,而响应体其实是 `{"detail":"请求过于频繁, 请稍后重试"}`。

**教训**:测试之间共享任何**按标识计数**的资源(限流器、配额、去重表)迟早串味,而串味后的失败信息通常指向完全无关的地方。排查这类"一堆莫名失败"时,**先找共享的计数资源**,再逐条读断言。

### 坑 7:测试不收垃圾,把共享数据库搞脏

Postgres 测试跑了几十遍之后,`checkpoints` 表里躺着一堆 `test-xxxxxxxx`。它不会让测试失败,但会在你下次 `cli.py list` 排查真问题时干扰视线。

修复:`cp.delete_thread(thread)`,以及一个 autouse fixture 统一清 `test-http-*`。

**教训**:测试用了共享的持久化资源,就有责任把它恢复原样。

### 坑 8:`/ui` 返回 307,而我以为挂载错了

`StaticFiles(html=True)` 会把 `/ui` 重定向到 `/ui/`。`curl -s` 不跟随重定向,于是看到 `307` 和 `0 bytes`。

**教训**:看到 3xx 先想"这是不是框架的正常跳转"。`-L` 一下再判断。

### 坑 9:验证脚本自己出的错

- shell 里写 `${h:+-H "$h"}` 想一次打完三种 header,结果空 header 那次被 shell 拆坏,得到 `401/400/400` 而不是 `401/401/200`。
- `python3 -c` 一行流里用了带反斜杠的 f-string,`SyntaxError: f-string expression part cannot include a backslash`。

**教训**:验证脚本出的错最容易被当成被测代码的错。一行流写不下就换 heredoc,不要为了"一条命令搞定"去和 shell 引号搏斗。

### 专家提示:三个"不做会后悔"的设计

1. **`_maybe_crash()` 故障注入。** 六行代码,是**验证 checkpointer 真的有用**的唯一诚实办法。没有它,"进程挂了能续跑"只是一句你抄来的宣传语。而且它在测试里直接复用,`test_崩溃后续跑不会重跑已完成的节点` 就靠它。

2. **每个循环都有硬性出口,而且出口值在 config 里。** `MAX_ROUNDS` 同时管两件事:评审重写的上限、人工驳回后重规划的上限。**没有它,一个刻薄的评审模型能让你无限重写,每一轮都是真金白银。** 这条在所有"自我改进"结构里都成立——Reflection、ReAct 的循环、多 agent 辩论,一个都不能少。

3. **`FakeLLM` 从第一天就写,并且记录每次调用。** `calls` 那个列表让你能断言"研究员被调了 3 次"、"温度全是 0"、"续跑之后研究员没有被再调一次"。**这些是控制流和成本的断言,而它们才是 agent 项目真正该测的东西。** 27 个测试 0.87 秒跑完,这个速度完全来自它。

### 专家提示:什么时候不要用 LangGraph

诚实一点,这套东西是有成本的:

- **一次性脚本、跑完就完的任务**:`docs/2/13` 那 30 行 `run_research()` 更好。状态机的收益全在"中断、恢复、审批、回放"上,不需要这些的时候它纯粹是额外的概念负担。
- **纯线性的流水线**:上一篇的 RAG 服务就是,一条直线走完,`if/else` 完全够。用状态机去表达一条直线,只是把函数调用改写成了字典。
- **对延迟敏感到毫秒级的场景**:每个节点跑完都要写一次 checkpoint,这是几毫秒的数据库往返。对一次要跑几十秒的研究任务无所谓,对一个要求 P99 50ms 的接口就不行。
- **团队里没人愿意学它**:这是最现实的一条。状态机的心智模型和普通业务代码差别不小,一个只有你懂的框架在团队里是负债。

**判断标准很简单:这个流程需要"停下来"吗?** 需要停(等人、等外部事件、崩了要续),就上状态机;从头跑到尾,就别上。

## 17. 面试视角

### 一句话版本

"我把一个深度研究 agent 从函数调用链重写成了 LangGraph 状态机:规划 → 人工审批 → 三个子任务并行研究 → 撰写 → 评审,评审不过退回重写,带轮数上限;checkpointer 用 Postgres,所以进程崩了能从崩掉的那个节点续跑、已完成的节点不重跑,人审可以隔一天在另一台机器上做;并行结果靠 `Annotated[list, operator.add]` reducer 合并,不加会直接抛 `InvalidUpdateError`;还支持时间旅行——从历史任意一步改状态分叉出一条新分支重跑,两条分支各自独立可读;对外是 7 个 HTTP 接口加一个 SSE 进度流和静态审批台,27 个测试 0.87 秒跑完,其中两个真连 Postgres 验跨连接续跑。"

### 高频追问与答法

**Q:为什么不用 `if/else` 手写这个流程?**

只要不需要"停下来",`if/else` 确实更好,`docs/2/13` 那 30 行就是。状态机的收益全在四件事上:**崩溃后从中断处续跑、执行到一半暂停等人、每一步的状态都留着可回放、并行结果的正确合并。** 这四件事自己实现一遍,你写出来的东西有个名字,叫工作流引擎——而且你的版本没有测试。

反过来说边界也要清楚:纯线性流水线、对延迟毫秒级敏感的接口、或者团队里没人愿意学它的时候,别上。

**Q:checkpointer 到底存了什么?怎么证明它有用?**

`setup()` 建四张表:`checkpoints`(每一步的状态快照)、`checkpoint_blobs`(大字段)、`checkpoint_writes`(节点的写入记录)、`checkpoint_migrations`。每个节点跑完自动写一次。

证明方式是**故障注入**:`RG_CRASH_AT=write` 让 write 节点抛异常,然后另起一个进程 `cli.py continue`。观察到的是 `next: ('write',)`、`findings: 3 条`、续跑之后研究员的调用次数**还是 3**——三次昂贵的搜索没有白花。**没有故障注入,"能续跑"就只是一句宣传语。**

**Q:`interrupt()` 恢复的时候,节点是从中断的那一行往下走吗?**

不是,**节点从头重跑**,只不过 `interrupt()` 这次直接返回你给的值。所以 `interrupt()` 之前的代码会执行两次,副作用(发通知、扣款、写库)必须放在它之后。

这是 checkpointer 的工作方式决定的:**节点是最小的重放单位,不是语句。** 同一条纪律在 Celery、Temporal、Airflow 里都成立。

**Q:`Command(resume=X)` 和 `invoke(None)` 有什么区别?**

前者是"回答一个正在等待的中断",后者是"从 `next` 指的节点继续跑"。关键在于**用错不会报错**:对一个等审批的 thread 调 `invoke(None)`,approve 节点重跑、再次中断,你拿到一个 200 响应和一张纹丝不动的图。

我是被一个测试逼着发现的:SSE 接口第一版只有 `invoke(None)`,`test_stream_逐节点推送` 断言收到过 `research` 节点,红了——流里只有一条 interrupt。修法是给接口加 `?action=approve|reject`,等审批时必须带,不带返回 409。**框架里"两个 API 长得像、用错不报错"的地方,都要在自己的边界上加显式检查。**

**Q:三个研究节点并行,结果怎么合并?**

`Send("research", payload)` 在运行时扇出 N 份,`add_edge("research", "write")` 提供自动汇合(N 个全完成才进 write)。合并靠 reducer:

```python
findings: Annotated[list[Finding], operator.add]
```

不加会抛 `InvalidUpdateError: At key 'findings': Can receive only one value per step`。**注意它不是静默覆盖——我原本以为是,跑了一遍才知道。** 报错比静默覆盖好,你第一次跑就会知道。

另外每个节点只返回 `{"findings": [一条]}`,不返回整个列表;并行完成顺序是乱的,所以 write 里要 `sorted(..., key=lambda f: f["index"])` 排回计划顺序。

**Q:时间旅行具体怎么用?**

`get_state_history()` 拿到所有快照(**倒序**),挑一个,`update_state(target.config, {...}, as_node="approve")` —— 它返回一个新 config,**分叉出一条新分支**,不改原来那条。之后拿 `thread_id + checkpoint_id` 就能分别读两条分支的最终状态。

两个真实用途:**排查**(线上一次跑歪了,回到歪之前那步,改掉输入重跑,看是不是同一个问题)和**对比**(同一个计划换两组子任务,跑完比报告,而不是起两个 thread 从头跑)。

**Q:`StateSnapshot.metadata` 里能看到哪个节点跑了吗?**

不能。1.2 的 metadata 只有 `step`、`source`、`parents`。"哪个节点跑了"在**上一个快照的 `.next`** 里,所以打印历史要把相邻两个快照配对着看。我一开始写 `metadata["writes"]`,拿到 `None` 走了兜底分支,打印出一片 `(loop)`。

**Q:agent 项目怎么测?不是不确定的吗?**

**测的不是模型输出,是图走了哪条路。** 27 个测试里的断言长这样:批准后 `findings` 是 3 条、驳回后 `plan_round` 变 2、驳回到上限 `status == "rejected"` 且研究员调用次数是 0(成本断言)、评审不过撰写人被调 2 次且新报告里出现了按意见加的章节、`review_round == MAX_ROUNDS`(循环有出口)。

前提是从第一天就有 `FakeLLM`,而且它**按内容而不是按调用次数**决定返回什么——计数器型替身在多 thread 下会串味,失败还是随机的。收益是 27 个测试 0.87 秒跑完,其中包含两个真连 Postgres 的。

**Q:为什么两个测试要用真 Postgres,不能全用 InMemorySaver?**

`InMemorySaver` 测不出"换一个连接还能读到"。`test_跨连接读回状态` 刻意开了两个独立的连接池,第二个池能读回 `next == ("approve",)`,才证明状态真在库里。这两条在 CI 里也必须真跑——**跳过的测试等于没有的测试,而且更危险,因为 `2 skipped` 没人会注意到。**

**Q:这个服务怎么扩容?**

`--workers 1`,横向加容器。**所有共享状态都在 Postgres 里,进程本身无状态**,所以一个 thread 可以在容器 A 上起、容器 B 上审批、容器 C 上续跑,不需要粘性会话。

多 worker 不是不能跑,是每个 worker 各建一个连接池(`max_size=5`),几个容器就能把 Postgres 默认的 `max_connections=100` 吃满,而症状是"偶发连不上数据库"。

**Q:限流为什么不复用上一篇的答案缓存?**

**因为这里不该有答案缓存。** 文档问答是无状态的,同一个问题该给同一个答案;研究 agent 每次执行都有自己的 thread、审批记录、分支历史,缓存"同样的问题返回上次的报告"等于把一个有状态的长任务伪装成一次查询。限流可以照搬,缓存不行。**照抄上一个项目的配件之前,先问一句这个项目的语义还成立吗。**

**Q:前端为什么不用 EventSource?**

它不能带自定义请求头,而鉴权在 `X-API-Key` 上。常见的"解决办法"是把 key 挪进查询串,那会让它落进 access log、浏览器历史和 Referer 头。改用 `fetch` + `TextDecoderStream` 自己按 `\n\n` 切块,多五行代码,换掉一个真实的安全问题。

**Q:继续做下去,下一步优化什么?**

先把审批做成**异步通知**(现在是人主动来查),接企业微信/飞书机器人推一条带链接的卡片;再把 `research` 节点里的搜索换成可配置的多路(现在只有一路)。**不会先去加更多节点**——控制流已经够复杂了,再加节点应该先有一个真实需求推着。

## 18. 划重点

1. **判断要不要上状态机,只问一句:这个流程需要"停下来"吗?** 需要停(等人、等外部事件、崩了要续)就上;从头跑到尾就别上。
2. **节点只返回要改的字段,不返回整个 state。** 返回全量会把并行结果冲掉。
3. **并行写同一个键必须有 reducer。** `Annotated[list[X], operator.add]`;不加会抛 `InvalidUpdateError`,**不是静默覆盖**——这一点我原本写错了,跑了一遍才改。
4. **`Send()` 是运行时扇出,`add_edge` 提供自动汇合。** 被 Send 的节点收到的是 payload,不是全局 state,这是刻意的隔离。
5. **`interrupt()` 恢复时节点从头重跑,副作用必须放在它之后。** 节点是最小的重放单位,不是语句。
6. **`Command(resume=X)` 和 `invoke(None)` 用错不会报错**,只会让图纹丝不动。在自己的边界上加显式检查(这一篇是两个 409)。
7. **每个循环都必须有硬性出口,出口值放 config。** 没有 `MAX_ROUNDS`,一个刻薄的评审模型能让你无限重写。
8. **故障注入(`RG_CRASH_AT`)是验证 checkpointer 的唯一诚实办法。** 六行代码,顺带被测试复用。
9. **续跑之后"已完成的节点不重跑"要用调用计数来断言**,不能靠肉眼看日志。
10. **`get_state_history()` 是倒序的**;`update_state(historical_config, ...)` **分叉**出新分支而不是改写历史,返回的新 config 里带着新的 `checkpoint_id`。
11. **`StateSnapshot.metadata` 只有 `step`/`source`/`parents`。** "哪个节点跑了"在上一个快照的 `.next` 里。
12. **agent 测试测的是控制流和成本,不是模型输出。** "驳回到上限时研究员被调 0 次"这种断言,比任何输出质量断言都值钱。
13. **测试替身的行为由输入决定,不能由调用次数决定。** 计数器型替身在多 thread 下串味,而且随机失败。
14. **测试之间共享任何按标识计数的资源都会串味**,失败信息还会指向完全无关的地方(`assert 429 == 422`)。用一次性标识。
15. **`skipif` 而不是 `try/except`,CI 里绝不跳过。** 跳过的测试比没有的测试更危险。
16. **测试用了共享持久化资源,就要自己收拾**(`delete_thread`)。
17. **配置缺失一律 fail closed。** 这一篇有两层:lifespan 里 `raise RuntimeError`,compose 里 `${RG_API_KEYS:?...}`,而且**要验证它该失败时真的失败了**。
18. **`--workers 1` + 横向加容器。** 所有共享状态在 Postgres 里,进程无状态,所以不需要粘性会话。
19. **限流用 Redis 自己的时间(`r.time()`),不用本机时钟。** 分布式限流必须有统一的时钟源。
20. **照抄上一个项目的配件前,先问语义还成立吗。** 限流能搬,答案缓存不能。
21. **任何"框架会怎样"的断言,自己跑一遍再往文档里写。** 二手结论在快迭代的框架上保鲜期很短。
22. **一个失败的测试推动一次接口重新设计,这才是写测试的真正回报。** 如果当时把断言改软,那个 SSE 接口会带着"对等审批的 thread 实际不可用"的缺陷发布,而且不报错。

## 19. 进阶练习

按投入产出排序:

1. **把审批改成异步通知**(半天)。现在是人主动来 `/ui` 查。接一个企业微信/飞书机器人,`plan` 节点跑完就推一条带 `thread_id` 链接的卡片。**注意副作用要放在 `interrupt()` 之后**,否则老板会收到两条——这正是第 16 节坑 4 的实战。
2. **给 `research` 节点加重试**(半天)。搜索接口会超时。现在一次失败整张图就停在那个节点上(靠 `continue` 手动续),加上节点内重试 + 指数退避会好很多。做完想一下:**重试放在节点里,和让 checkpointer 承担重试,区别是什么?**
3. **加一个"人工编辑报告"的中断**(1 天)。在 `review` 之后、`finalize` 之前再插一个 `interrupt()`,让人直接改报告文本。这会暴露一个新问题:**改过的报告要不要再评审一次?** 你的答案决定了那条边怎么连。
4. **把 `MAX_ROUNDS` 换成成本上限**(1 天)。轮数是个粗糙的代理,真正要控的是钱。在 state 里累计 token 用量,`after_review` 改成按累计成本判断。**注意 state 里的累计字段需要 reducer**,否则并行节点一写就报错。
5. **多 agent 辩论**(2 天)。把 `write` 换成两个撰写人 + 一个裁判:`Send` 扇出两份报告,裁判节点选一份或合并。这是 `Send` + reducer 的第二个真实用例,也是检验你有没有真的理解"节点只返回要改的字段"。
6. **接 LangSmith 或自己的 tracing**(半天)。上一篇手写了 157 行 tracing,这一篇一行都没写——因为 langgraph 的 `stream(stream_mode="debug")` 已经给了任务级事件。把它落到上一篇那套 span 结构里,你会同时理解两边的抽象。
7. **换掉 checkpointer 后端**(半天)。`SqliteSaver` 或 `MongoDBSaver`。整个 `graph.py` 一行都不用改——这是"把持久化收在一个接口后面"的回报,亲手验一次比读一遍强。

## 20. 两个项目一起看:你现在手上有什么

19 和 20 是刻意配对的:

| | 19 · 企业文档问答 | 20 · 研究智能体 |
|---|---|---|
| 平面 | **数据平面** | **控制平面** |
| 核心问题 | 怎么找对材料、怎么不胡说 | 怎么停得住、怎么恢复、怎么改主意 |
| 关键技术 | BM25+向量 RRF、cross-encoder 精排、阈值兜底 | StateGraph、Postgres checkpointer、`interrupt()`、时间旅行 |
| 质量怎么保证 | 24 条标注评测集 + CI 质量闸门 | 27 个控制流测试 + 故障注入 |
| 流程形状 | 一条直线 | 有分支、有循环、能暂停 |
| 状态在哪 | 进程无状态,知识库在 Qdrant/PG | 进程无状态,执行状态在 PG |
| 共同点 | fail closed、限流、`--workers 1`、只 mock LLM、Docker 交付 | 同左 |

最后一行不是凑数:**这两个项目在工程纪律上完全一致**,而这些纪律和 agent 没关系,是任何生产服务都要的。区别只在业务问题不同。

如果要在简历上写一句话把两个都带上:"做过一个 RAG 生产系统(混合检索 + 精排 + 评测闸门 + 可观测性)和一个有状态可恢复的 agent 工作流(LangGraph + Postgres checkpointer + 人在环中 + 时间旅行),两者都有 Docker 交付和 CI。"

## 21. 下一章预告

到这里,`docs/2` 的六个手敲项目走完了一条完整的路:

- 从**一个能跑的 agent**(11~13)
- 到**能评测、能观测的 agent**(14~18)
- 到**数据平面的生产化**(19)
- 到**控制平面的生产化**(20)

剩下没覆盖的、而真实工作里一定会遇到的,主要是三块:**多 agent 协作的组织方式**(这一篇的 `Send` 只是最简单的一种)、**agent 的长期记忆**(不是对话历史,是跨会话的知识积累)、以及**上线之后的持续评估**(线上数据怎么回流成评测集)。

它们的共同点是:**都不是"再学一个框架"能解决的,而是要先有一个跑在真实流量上的 agent 才谈得上。** 所以下一步最有价值的动作不是继续读,是把这两个项目里的任意一个,换成你自己工作中的真实场景重做一遍——**同样的架构,你自己的数据和需求。** 那一遍会暴露的问题,比任何教程都具体。

上一篇:[19 · 实战手敲五:企业级文档问答系统](./19-实战手敲五-企业级文档问答系统.md) | 返回目录:[README](./README.md)





