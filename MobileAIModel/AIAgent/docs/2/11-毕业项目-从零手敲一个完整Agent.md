# 11 · 毕业项目:从零手敲一个完整 Agent

> 前面三个项目是"跑通官方代码 + 精读 + 改造",这一篇反过来:**每一行代码都由你亲手敲出来**。做完你将拥有一个 100% 属于自己的完整 Agent Demo——带工具调用、带记忆、带流式输出、带网页界面,总共 7 个文件、约 300 行代码。
>
> 它就是整套教程的"期末考试":每一段代码用到的知识,都在前面的篇章里讲过(每节都标了出处)。如果哪里看不懂,回去补对应章节。
>
> 预计 1~2 天。**禁止复制粘贴,必须手敲**——这是你对自己的承诺。

## 0. 成品长什么样

浏览器打开 `http://127.0.0.1:8000`,是一个聊天页面。你输入:

- "北京今天天气怎么样?" → 页面上先显示 `🔧 调用工具 get_weather`,再显示天气回答(**工具调用**)
- "(100-25)*0.8 等于多少?" → Agent 调用计算器工具,精确回答(**LLM 不擅长算数,工具来补**)
- "我叫小明,我喜欢滑雪" → Agent 记住(**长期记忆落盘**)
- 重启服务后再问"我叫什么?" → 它还记得(**记忆持久化**)
- 全程每个中间步骤实时推送到页面(**SSE 流式**)

## 1. 项目设计(动手前先想清楚)

```
浏览器 static/index.html ──POST /chat/stream (SSE)──▶ FastAPI main.py
                                                        │
                                                        ▼
                                                MiniAgent (agent.py)
                                                 │  循环: 问LLM → 检测[TOOL_CALL] → 执行 → 回喂
                        ┌───────────────┬────────┴──────┬─────────────┐
                        ▼               ▼               ▼             ▼
                    llm.py          tools.py        memory.py     .env
                    调模型API       三个工具         短期+长期记忆   密钥配置
```

文件职责(对应第 04 篇第 8 节讲的"分层"):

| 文件 | 职责 | 知识出处 |
| --- | --- | --- |
| `.env` / `requirements.txt` | 配置与依赖 | 01 篇第 9 节、02 篇 |
| `llm.py` | 封装 LLM API 调用 | 03 篇第 1 节 |
| `tools.py` | 工具定义 + 执行器 | 03 篇 3.3 节 |
| `agent.py` | Agent 循环(问→调工具→回喂) | 03 篇第 2 节 |
| `memory.py` | 短期对话 + 长期事实记忆 | 03 篇 3.5 节、07 篇 |
| `main.py` | FastAPI 接口 + SSE | 04 篇 |
| `static/index.html` | 聊天界面 | 04 篇第 6、7 节 |

## 2. PyCharm 建项目(10 分钟)

1. PyCharm → New Project,名称 `mini-agent`,Interpreter 选 **New virtualenv**(或先 `uv venv` 再指向 `.venv/bin/python`,见 02 篇第 5 节)
2. 在项目根目录新建文件 `requirements.txt`:

```
openai>=1.40.0
fastapi>=0.115.0
uvicorn[standard]>=0.32.0
python-dotenv>=1.0.0
httpx>=0.27.0
```

3. PyCharm 底部打开 Terminal(它已自动激活虚拟环境),安装:

```bash
pip install -r requirements.txt        # 用uv的话: uv pip install -r requirements.txt
```

4. 新建 `.env`(填你在 02 篇申请的三件套):

```
LLM_MODEL_ID=deepseek-chat
LLM_API_KEY=sk-你的key
LLM_BASE_URL=https://api.deepseek.com/v1
```

5. 新建 `static` 文件夹(右键项目根目录 → New → Directory)

**✅ Checkpoint 0**:PyCharm 右下角解释器显示项目的虚拟环境;Terminal 里 `pip list` 能看到 fastapi。

## 3. 第一层:llm.py —— 会说话(30 分钟)

```python
"""LLM调用封装: 整个项目只有这个文件和模型API打交道"""
import os

from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()  # 读取.env,写入环境变量

_client = OpenAI(
    api_key=os.getenv("LLM_API_KEY"),
    base_url=os.getenv("LLM_BASE_URL"),
)
MODEL = os.getenv("LLM_MODEL_ID")


def chat(messages: list[dict], temperature: float = 0.3) -> str:
    """发一组messages,返回模型的完整回复文本"""
    resp = _client.chat.completions.create(
        model=MODEL,
        messages=messages,
        temperature=temperature,
    )
    return resp.choices[0].message.content or ""


if __name__ == "__main__":
    # 单测:直接右键Run这个文件
    print(chat([{"role": "user", "content": "用一句话介绍你自己"}]))
```

**✅ Checkpoint 1**:右键 `llm.py` → Run,打印出模型的一句自我介绍。不通先回 09 篇"LLM 调用类"排错。

## 4. 第二层:tools.py —— 会干活(1 小时)

三个工具:查时间(零依赖)、查天气(免费的 wttr.in,无需 Key)、计算器(安全求值,不用危险的 `eval`)。其中 `httpx` 是 Python 里发 HTTP 请求的库(第 03 篇 requests 的现代同类,还支持异步),`pip install -r requirements.txt` 时已装好。

```python
"""工具箱: 每个工具是一个普通函数, TOOLS注册表统一管理"""
import ast
import operator
from datetime import datetime

import httpx


def get_time(**kwargs) -> str:
    return datetime.now().strftime("现在是 %Y-%m-%d %H:%M:%S")


def get_weather(city: str = "Beijing", **kwargs) -> str:
    """wttr.in是一个免费天气服务,城市名用英文/拼音更稳定"""
    try:
        resp = httpx.get(f"https://wttr.in/{city}", params={"format": "3"}, timeout=10)
        return resp.text.strip()
    except Exception as e:
        return f"天气查询失败: {e}"


# --- 安全计算器: 只允许数字和四则运算,防止eval执行恶意代码(第10篇第7节讲过为什么) ---
_OPS = {
    ast.Add: operator.add, ast.Sub: operator.sub,
    ast.Mult: operator.mul, ast.Div: operator.truediv,
    ast.Pow: operator.pow, ast.USub: operator.neg,
}


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


def calculate(expression: str = "", **kwargs) -> str:
    try:
        result = _safe_eval(ast.parse(expression, mode="eval"))
        return f"{expression} = {result}"
    except Exception as e:
        return f"计算失败: {e}"


# --- 注册表: Agent通过它知道有哪些工具、怎么描述给模型听 ---
TOOLS = {
    "get_time": {"func": get_time, "desc": "查询当前日期和时间。无参数"},
    "get_weather": {"func": get_weather, "desc": "查询城市实时天气。参数: city=城市名(英文或拼音,如 Beijing)"},
    "calculate": {"func": calculate, "desc": "计算数学表达式。参数: expression=算式,如 (100-25)*0.8"},
}


def tools_manual() -> str:
    """生成给模型看的'工具说明书'(拼进system prompt)"""
    return "\n".join(f"- {name}: {info['desc']}" for name, info in TOOLS.items())


def run_tool(name: str, args: dict) -> str:
    if name not in TOOLS:
        return f"错误: 没有名为 {name} 的工具"
    return TOOLS[name]["func"](**args)


if __name__ == "__main__":
    print(run_tool("get_time", {}))
    print(run_tool("calculate", {"expression": "(100-25)*0.8"}))
    print(run_tool("get_weather", {"city": "Beijing"}))
```

**✅ Checkpoint 2**:右键 Run,三行输出:当前时间、`(100-25)*0.8 = 60.0`、一行天气(网络不通会打印失败原因,不影响后续)。

## 5. 第三层:memory.py —— 记得住(1 小时)

短期记忆 = 每个会话最近 10 轮对话;长期记忆 = 用户说的个人信息,追加写入 `long_term.jsonl` 文件,重启不丢。

```python
"""记忆系统: 短期(会话内最近对话) + 长期(用户事实,落盘jsonl)"""
import json
from collections import defaultdict, deque
from pathlib import Path


class Memory:
    # 用户消息里出现这些词,就值得存进长期记忆
    TRIGGERS = ("我叫", "我是", "我喜欢", "我讨厌", "我住在", "我的")

    def __init__(self, path: str = "long_term.jsonl", short_rounds: int = 10):
        # 短期: 每个session一个双端队列,自动挤掉最旧的(1轮=user+assistant共2条)
        # defaultdict=带默认值的字典: 访问不存在的key时自动帮你创建,不会报KeyError
        self.short = defaultdict(lambda: deque(maxlen=short_rounds * 2))
        # 长期: 启动时从文件加载历史事实
        self.path = Path(path)
        self.facts: list[str] = []
        if self.path.exists():
            for line in self.path.read_text(encoding="utf-8").splitlines():
                if line.strip():
                    self.facts.append(json.loads(line)["fact"])

    def recent(self, session_id: str) -> list[dict]:
        """取出该会话的近期对话,直接拼进messages"""
        return list(self.short[session_id])

    def add_turn(self, session_id: str, user_input: str, reply: str):
        self.short[session_id].append({"role": "user", "content": user_input})
        self.short[session_id].append({"role": "assistant", "content": reply})

    def maybe_remember(self, user_input: str):
        """触发词命中 → 存入长期记忆并落盘"""
        if any(t in user_input for t in self.TRIGGERS):
            self.facts.append(user_input)
            with self.path.open("a", encoding="utf-8") as f:
                f.write(json.dumps({"fact": user_input}, ensure_ascii=False) + "\n")

    def recall(self, query: str, limit: int = 3) -> list[str]:
        """简化版检索: 按字符重合度排序。
        真实系统用embedding语义检索(第10篇RAG节),这里用最朴素的办法说明原理。"""
        def score(fact: str) -> int:
            return len(set(fact) & set(query))

        ranked = sorted(self.facts, key=score, reverse=True)
        return [f for f in ranked[:limit] if score(f) >= 2]


if __name__ == "__main__":
    m = Memory(path="test_memory.jsonl")
    m.maybe_remember("我叫小明,我喜欢滑雪")
    m.maybe_remember("今天天气不错")          # 无触发词,不会存
    print("长期记忆:", m.facts)
    print("检索'我叫什么':", m.recall("我叫什么"))
    Path("test_memory.jsonl").unlink()        # 清理测试文件
```

**✅ Checkpoint 3**:Run 输出长期记忆只有一条(天气那句没存),且能按"我叫什么"检索到它。

## 6. 第四层:agent.py —— 核心循环(2 小时,最重要的文件)

这就是 03 篇第 2 节那个伪代码循环的真实实现:问模型 → 正则检测 `[TOOL_CALL:...]` → 执行工具 → 结果回喂 → 直到模型给出最终答案。

> 名词解释:**正则表达式(regex)**= 一种描述"文本长什么样"的迷你语言。`re.compile(规则)` 先把规则编译好,`.search(文本)` 就能在任意字符串里按规则找匹配、并把括号圈住的部分抓出来(`group(1)`、`group(2)`)。本文只用这一个规则,看懂它即可:`\[TOOL_CALL:` 匹配字面文字,`(\w+)` 抓工具名(连续的字母数字),`([^\]]*)` 抓参数串(直到右方括号为止的一切)。

```python
"""MiniAgent: 问LLM → 检测工具调用 → 执行 → 回喂 → 循环"""
import re

from llm import chat
from memory import Memory
from tools import run_tool, tools_manual

# 匹配 [TOOL_CALL:工具名:参数串] ,参数串可为空
TOOL_CALL_RE = re.compile(r"\[TOOL_CALL:(\w+):?([^\]]*)\]")

SYSTEM_PROMPT = f"""你是一个乐于助人的中文智能助手,名叫小助。

你可以使用以下工具:
{tools_manual()}

需要使用工具时,只输出一行调用指令,格式必须严格如下(方括号和冒号一个都不能少):
[TOOL_CALL:工具名:参数名=参数值]

示例:
[TOOL_CALL:get_weather:city=Beijing]
[TOOL_CALL:calculate:expression=(100-25)*0.8]
[TOOL_CALL:get_time:]

规则:
1. 一次只调用一个工具,拿到结果后再决定下一步
2. 不需要工具时,直接用简洁自然的中文回答
3. 涉及实时信息(时间/天气)或数学计算时必须用工具,不要自己编造
"""


class MiniAgent:
    def __init__(self, memory: Memory, max_steps: int = 5):
        self.memory = memory
        self.max_steps = max_steps  # 防止模型无限调用工具(死循环保险丝)

    def _build_messages(self, session_id: str, user_input: str) -> list[dict]:
        """动态拼装上下文: 人设 + 相关长期记忆 + 短期对话 + 当前输入(07篇的三段式)"""
        messages = [{"role": "system", "content": SYSTEM_PROMPT}]
        facts = self.memory.recall(user_input)
        if facts:
            messages.append({
                "role": "system",
                "content": "关于用户的已知信息:\n" + "\n".join(facts),
            })
        messages.extend(self.memory.recent(session_id))
        messages.append({"role": "user", "content": user_input})
        return messages

    @staticmethod
    def _parse_args(arg_str: str) -> dict:
        """'city=Beijing,unit=c' → {'city': 'Beijing', 'unit': 'c'}"""
        args = {}
        for pair in arg_str.split(","):
            if "=" in pair:
                key, value = pair.split("=", 1)
                args[key.strip()] = value.strip()
        return args

    def run(self, session_id: str, user_input: str, on_event=None) -> str:
        """执行一轮对话。on_event是可选回调,每个中间步骤都会通知它(供SSE推送)"""
        def emit(event: dict):
            if on_event:
                on_event(event)

        messages = self._build_messages(session_id, user_input)

        for _ in range(self.max_steps):
            reply = chat(messages)
            match = TOOL_CALL_RE.search(reply)

            if not match:  # 没有工具调用 = 最终答案
                self.memory.add_turn(session_id, user_input, reply)
                self.memory.maybe_remember(user_input)
                emit({"type": "final", "content": reply})
                return reply

            # 有工具调用: 解析 → 执行 → 把结果回喂给模型
            name, args = match.group(1), self._parse_args(match.group(2))
            emit({"type": "tool_call", "name": name, "args": args})
            result = run_tool(name, args)
            emit({"type": "tool_result", "name": name, "result": result})

            messages.append({"role": "assistant", "content": reply})
            messages.append({
                "role": "user",
                "content": f"[工具 {name} 的返回结果]\n{result}\n请基于这个结果继续回答。",
            })

        reply = "抱歉,尝试多次工具调用后仍未解决,请换个问法。"
        self.memory.add_turn(session_id, user_input, reply)
        emit({"type": "final", "content": reply})
        return reply


if __name__ == "__main__":
    agent = MiniAgent(Memory())
    print(agent.run("test", "现在几点了?", on_event=print))
```

**✅ Checkpoint 4**:Run 后能看到事件依次打印:`tool_call`(get_time)→ `tool_result` → `final`(带真实时间的回答)。如果模型不调用工具而是编造时间,说明它没听话——换个更强的模型,或把 SYSTEM_PROMPT 里的示例部分再强调一遍(这就是 03 篇说的"提示词工程实战")。

## 7. 第五层:main.py —— 变成服务(1 小时)

```python
"""FastAPI入口: 普通接口 + SSE流式接口 + 静态页面"""
import json
import queue
import threading

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import FileResponse, StreamingResponse
from pydantic import BaseModel

from agent import MiniAgent
from memory import Memory

app = FastAPI(title="Mini Agent Demo")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 学习环境全放行,生产要收紧(第08篇)
    allow_methods=["*"],
    allow_headers=["*"],
)

memory = Memory()
agent = MiniAgent(memory)


class ChatRequest(BaseModel):
    session_id: str = "default"
    message: str


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post("/chat")
def chat(req: ChatRequest):
    """普通版: 等全部完成后一次性返回"""
    reply = agent.run(req.session_id, req.message)
    return {"reply": reply}


@app.post("/chat/stream")
def chat_stream(req: ChatRequest):
    """流式版: Agent在后台线程干活,每个事件实时推给浏览器(06篇的生产者-消费者)"""
    q: queue.Queue = queue.Queue()

    def worker():
        try:
            agent.run(req.session_id, req.message, on_event=q.put)
        except Exception as e:
            q.put({"type": "error", "detail": str(e)})
        q.put(None)  # 结束哨兵: 约定好的"没有更多数据了"信号,消费方看到它就收工

    threading.Thread(target=worker, daemon=True).start()

    def events():
        while True:
            event = q.get()
            if event is None:
                break
            yield f"data: {json.dumps(event, ensure_ascii=False)}\n\n"

    return StreamingResponse(events(), media_type="text/event-stream")


@app.get("/")
def index():
    return FileResponse("static/index.html")


if __name__ == "__main__":
    import uvicorn

    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

**✅ Checkpoint 5**:Run `main.py`,打开 `http://127.0.0.1:8000/docs`,用 `POST /chat` 发 `{"message": "现在几点了?"}` 得到带真实时间的回复;后端终端能看到整个过程。

## 8. 第六层:static/index.html —— 有脸面(1 小时)

在 `static` 文件夹里新建 `index.html`(前端零基础照着敲即可,每段都有注释):

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="utf-8">
<title>Mini Agent</title>
<style>
  body { font-family: system-ui, sans-serif; max-width: 640px; margin: 40px auto; padding: 0 16px; }
  #log { border: 1px solid #ddd; border-radius: 8px; padding: 12px; height: 420px; overflow-y: auto; }
  .user  { color: #0a58ca; margin: 8px 0; }
  .agent { color: #222;    margin: 8px 0; }
  .sys   { color: #999;    margin: 4px 0; font-size: 13px; }
  form  { display: flex; gap: 8px; margin-top: 12px; }
  input { flex: 1; padding: 8px; }
</style>
</head>
<body>
<h3>Mini Agent Demo</h3>
<div id="log"></div>
<form id="f">
  <input id="msg" placeholder="试试: 北京天气怎么样 / 我叫小明 / (100-25)*0.8等于多少" autocomplete="off">
  <button>发送</button>
</form>

<script>
const log = document.getElementById("log");

function add(cls, text) {                    // 往聊天记录里加一行
  const div = document.createElement("div");
  div.className = cls;
  div.textContent = text;
  log.appendChild(div);
  log.scrollTop = log.scrollHeight;          // 自动滚到底
}

document.getElementById("f").onsubmit = async (e) => {
  e.preventDefault();                        // 阻止表单默认的刷新页面行为
  const input = document.getElementById("msg");
  const message = input.value.trim();
  if (!message) return;
  input.value = "";
  add("user", "我: " + message);

  // 调SSE接口,持续读取事件流(04篇第6节讲的原理)
  const resp = await fetch("/chat/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ session_id: "default", message }),
  });
  const reader = resp.body.getReader();
  const decoder = new TextDecoder();
  let buf = "";
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buf += decoder.decode(value, { stream: true });
    const parts = buf.split("\n\n");         // SSE消息以空行分隔
    buf = parts.pop();                       // 最后一段可能不完整,留到下一轮
    for (const part of parts) {
      if (!part.startsWith("data: ")) continue;
      const ev = JSON.parse(part.slice(6));
      if (ev.type === "tool_call")   add("sys", `🔧 调用工具 ${ev.name} ${JSON.stringify(ev.args)}`);
      if (ev.type === "tool_result") add("sys", `📄 工具返回: ${ev.result}`);
      if (ev.type === "final")       add("agent", "小助: " + ev.content);
      if (ev.type === "error")       add("sys", "出错: " + ev.detail);
    }
  }
};
</script>
</body>
</html>
```

**✅ 最终验收**(逐条打勾,全过 = 毕业):

- [ ] `http://127.0.0.1:8000` 打开聊天页
- [ ] 问天气 → 页面依次出现 🔧 工具调用、📄 工具返回、最终回答
- [ ] 问 "(100-25)*0.8 等于多少" → 计算器工具给出 60.0
- [ ] 说 "我叫小明,我喜欢滑雪" → 项目目录出现 `long_term.jsonl`
- [ ] **重启 main.py** 后问 "我叫什么?" → 它还记得
- [ ] 连续追问 "那我喜欢什么?" → 短期记忆让它接得上上下文

## 9. 毕业后的三个升级方向(每个都指回教程)

1. **把文本协议换成原生 Function Calling**(第 10 篇第 2 节):`tools.py` 的注册表改成 JSON Schema,`agent.py` 改读 `tool_calls` 字段——体会可靠性差异
2. **把记忆检索换成真语义检索**(第 10 篇第 4 节):用 `sentence-transformers` 给每条事实算 embedding,余弦相似度排序替换字符重合度——你就手写了一个最小 RAG
3. **生产化三件套**(第 10 篇第 5 节):给 `/chat` 加 Redis 结果缓存、`slowapi` 限流、locust 压测出 P95 拐点——写进简历

## 10. 面试怎么讲这个项目

一分钟版本:"我从零手写了一个 Agent 服务,没有用框架:自己实现了工具调用的文本协议和解析循环、双层记忆(会话内短期 + 落盘长期 + 检索拼装上下文)、FastAPI 的 SSE 流式推送中间步骤。后来对照 HelloAgents 框架和三个完整项目,理解了框架在这些环节上各自做了什么增强,比如 MCP 接工具、embedding 语义检索记忆。"——**"手写过底层 + 用过框架"是初级候选人最有说服力的组合。**

下一篇:[12 · 实战手敲二:多智能体旅行规划系统](./12-实战手敲二-多智能体旅行规划系统.md) | 返回目录:[README](./README.md)
