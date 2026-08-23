# 03 · Agent 原理与 HelloAgents 框架精讲

> 本篇把三个项目共用的"大脑部分"一次讲透:LLM 是怎么被调用的、Agent 到底是什么、工具调用怎么实现、MCP 是什么协议、记忆系统怎么工作。学完后,你看项目代码里的 `SimpleAgent`、`MCPTool`、`MemoryManager` 就不再是黑盒。

## 1. 一切的起点:LLM API 调用

抛开所有框架,调用大模型本质上就是一次 HTTP POST(HTTP = 程序之间通过网络传数据的标准协议,POST 是其中"提交数据"的动作,第 04 篇会细讲;此处只需知道:**调模型 = 发一段 JSON 过去,收一段 JSON 回来**):

```python
# 不用任何框架,裸调 LLM(理解用,不用手敲)
import httpx

response = httpx.post(
    "https://api-inference.modelscope.cn/v1/chat/completions",
    headers={"Authorization": "Bearer 你的KEY"},
    json={
        "model": "Qwen/Qwen2.5-72B-Instruct",
        "messages": [
            {"role": "system", "content": "你是旅行规划助手"},
            {"role": "user", "content": "推荐北京的景点"},
        ],
    },
)
print(response.json()["choices"][0]["message"]["content"])
```

三个关键概念:

1. **messages 是一个列表**,每条消息有 `role`(角色)和 `content`(内容)。
2. **role 有三种**:`system`(人设/规则,用户看不到)、`user`(用户说的话)、`assistant`(模型之前的回复)。
3. **LLM 是无状态的**:它不记得上一次调用。所谓"多轮对话",就是每次把历史 messages 全部重发一遍。

> 这就是为什么需要"记忆系统"(项目三):对话一长,历史消息塞不下也不划算,必须有策略地筛选"该带上哪些历史"。

## 2. 从 LLM 到 Agent:差的是什么?

| | LLM | Agent |
| --- | --- | --- |
| 行为 | 你问它答,一次性 | 围绕目标,多步骤行动 |
| 能力边界 | 只能生成文本 | 能调用工具(搜索、地图、数据库…) |
| 知识 | 截止训练日期,不知道实时信息 | 通过工具获取实时信息 |
| 状态 | 无状态 | 可以有记忆、有任务清单 |

**Agent = LLM + 提示词(人设与规则)+ 工具 + 循环控制**。用伪代码表达,一个最小的 Agent 就是:

```python
def agent_run(user_input):
    messages = [system_prompt, user_input]
    while True:
        reply = llm(messages)                # 1. 问模型
        if wants_tool_call(reply):           # 2. 模型说"我要用工具"
            result = execute_tool(reply)     # 3. 真的去执行工具
            messages.append(reply)
            messages.append(result)          # 4. 把工具结果喂回去
        else:
            return reply                     # 5. 模型给出最终答案
```

这个"问模型 → 调工具 → 结果回喂 → 再问模型"的循环,就是所有 Agent 框架的心脏。HelloAgents、LangChain、AutoGen 概莫能外。

## 3. HelloAgents 框架:项目实际用到的部分

HelloAgents 是 Datawhale 配套教材的自研框架。三个项目只用到它的一小部分,我们只讲这一小部分。

### 3.1 HelloAgentsLLM —— 统一的模型入口

```python
from hello_agents import HelloAgentsLLM

llm = HelloAgentsLLM()   # 零参数:自动从环境变量读取配置
```

它做的事:读取 `LLM_MODEL_ID`、`LLM_API_KEY`、`LLM_BASE_URL` 三个环境变量,包装成一个统一的调用接口。也可以显式传参覆盖(项目二就是这么做的):

```python
llm = HelloAgentsLLM(
    model="deepseek-chat",
    api_key="sk-xxx",
    base_url="https://api.deepseek.com/v1",
    temperature=0.0,     # 温度:0=严谨稳定,1=发散有创意。要JSON输出时用低温
)
```

> 出现位置:项目一 `app/services/llm_service.py` 的 `get_llm()`、项目二 `agent.py` 的 `_init_llm()`、项目三 `agents.py` 的 `NPCAgentManager.__init__`。三处本质上都是这几行。

### 3.2 SimpleAgent —— 最常用的 Agent 类

```python
from hello_agents import SimpleAgent

agent = SimpleAgent(
    name="景点搜索专家",              # 名字,打日志用
    llm=llm,                          # 用哪个模型
    system_prompt="你是景点搜索专家…"  # 人设与规则(最重要!)
)

reply = agent.run("搜索北京的历史文化景点")   # 一轮执行,返回字符串
```

`run()` 内部就是第 2 节那个循环:发消息 → 检测工具调用 → 执行 → 回喂 → 输出最终文本。

**Agent 的能力上限几乎完全由 system_prompt 决定。** 项目一的四个 Agent 用的是同一个 `SimpleAgent` 类,区别只在提示词:

```python
# 项目一的真实代码结构(简化)
attraction_agent = SimpleAgent(name="景点搜索专家", llm=llm, system_prompt=ATTRACTION_AGENT_PROMPT)
weather_agent    = SimpleAgent(name="天气查询专家", llm=llm, system_prompt=WEATHER_AGENT_PROMPT)
hotel_agent      = SimpleAgent(name="酒店推荐专家", llm=llm, system_prompt=HOTEL_AGENT_PROMPT)
planner_agent    = SimpleAgent(name="行程规划专家", llm=llm, system_prompt=PLANNER_AGENT_PROMPT)
```

### 3.3 工具调用:HelloAgents 的文本协议

主流商业 API 用 "function calling"(模型输出结构化 JSON 表示要调用什么)。HelloAgents 为了兼容各种模型,用了一个更朴素的**文本协议**:让模型直接在回复里输出一个特殊标记:

```
[TOOL_CALL:工具名:参数1=值1,参数2=值2]
```

框架检测到这个格式,就去执行对应工具,把结果喂回给模型。项目一的提示词里大段大段的"格式必须完全正确,包括方括号和冒号",就是在教模型输出这个标记:

```
使用maps_text_search工具时,必须严格按照以下格式:
[TOOL_CALL:amap_maps_text_search:keywords=景点关键词,city=城市名]
```

> 想通这一点,你就明白了两件事:(1) 为什么提示词要写得这么啰嗦——小模型容易把格式写错;(2) 为什么项目一构建查询时干脆直接把 `[TOOL_CALL:...]` 写进用户消息里——等于"手把手替模型把工具调用写好",提高成功率。

给 Agent 挂工具:

```python
agent.add_tool(amap_tool)        # 挂上工具
agent.list_tools()               # 查看已挂载的工具列表
```

### 3.4 MCP:给 Agent 接外部服务的标准插座

**MCP(Model Context Protocol)** 是 Anthropic 提出的开放协议,解决"每接一个外部服务就要写一遍胶水代码"的问题。类比:

- 没有 MCP:每个 App 自己写数据线(高德一套代码、GitHub 一套代码…)
- 有了 MCP:所有服务都做成 Type-C 接口(MCP Server),Agent 框架只要有一个 Type-C 插座(MCP Client)就能全接

项目一接高德地图就是三行配置的事:

```python
from hello_agents.tools import MCPTool

amap_tool = MCPTool(
    name="amap",
    description="高德地图服务",
    server_command=["uvx", "amap-mcp-server"],           # 怎么启动这个MCP服务器
    env={"AMAP_MAPS_API_KEY": settings.amap_api_key},    # 传给它的环境变量
    auto_expand=True,     # 自动把服务器里的每个工具展开成独立工具
)
```

运行时发生的事:

1. 框架执行 `uvx amap-mcp-server`,在**子进程**里启动高德 MCP 服务器
2. 通过标准输入输出(stdio)和它通信,问它"你有哪些工具?"
3. 服务器答复:`maps_text_search`(搜POI)、`maps_weather`(查天气)、`maps_direction_driving_by_address`(驾车路线)……
4. `auto_expand=True` 把这些工具全部注册给 Agent,名字加上前缀变成 `amap_maps_text_search` 等

于是 Agent 输出 `[TOOL_CALL:amap_maps_text_search:keywords=公园,city=上海]` 时,框架就会把请求转发给子进程里的高德服务器,拿到真实的 POI 数据。

### 3.5 记忆系统(项目三用)

`hello_agents.memory` 提供 `MemoryManager`,项目三给每个 NPC 配一个:

```python
from hello_agents.memory import MemoryManager, MemoryConfig

memory_config = MemoryConfig(
    storage_path="memory_data/张三",   # 存哪(SQLite数据库文件)
    working_memory_capacity=10,        # 工作记忆:最近10条对话
    working_memory_tokens=2000,        # 工作记忆最大token数
    max_capacity=100,                  # 长期记忆最多100条
    importance_threshold=0.3,          # 重要性低于0.3的记忆不检索
    decay_factor=0.95,                 # 时间衰减:越久远的记忆权重越低
)

memory = MemoryManager(
    config=memory_config,
    user_id="张三",
    enable_working=True,    # 工作记忆(短期,最近对话)
    enable_episodic=True,   # 情景记忆(长期,重要事件)
    enable_semantic=False,  # 语义记忆(知识图谱,本项目不用)
    enable_perceptual=False # 感知记忆(图像声音,本项目不用)
)
```

两个核心操作:

```python
# 写入:每轮对话结束后存进去
memory.add_memory(
    content="玩家说: 我喜欢喝咖啡",
    memory_type="working",
    importance=0.5,                  # 重要性打分,影响将来是否被检索到
    metadata={"speaker": "player"},
)

# 读取:下轮对话前,按当前话题检索相关记忆
relevant = memory.retrieve_memories(
    query="咖啡",                     # 和这个query语义相关的记忆
    memory_types=["working", "episodic"],
    limit=5,
    min_importance=0.3,
)
```

检索出的记忆会被拼进提示词(所谓 RAG 式记忆增强):

```
【之前的对话记忆】
[14:32] 玩家说: 我喜欢喝咖啡
[14:33] 我说: 我也是!休息区的手冲不错

【当前对话】
玩家: 推荐个饮料?
```

模型看到上下文,自然回答"要不试试咖啡?"——**"记忆"不是模型记住了,而是我们把相关历史检索出来重新塞给它**。

### 3.6 ToolAwareSimpleAgent(项目二用)

项目二用的是 `SimpleAgent` 的增强版:

```python
from hello_agents import ToolAwareSimpleAgent

agent = ToolAwareSimpleAgent(
    name="研究规划专家",
    llm=llm,
    system_prompt="...",
    enable_tool_calling=True,
    tool_registry=registry,             # 工具注册表(而非逐个add_tool)
    tool_call_listener=tracker.record,  # 每次工具调用都回调这个函数(用于前端实时展示)
)
```

多出来的能力就一个:**工具调用会触发监听器回调**。项目二靠它把"Agent 正在记笔记"这类事件实时推送到浏览器。

## 4. 多 Agent 协作:三种模式对号入座

三个项目正好展示了三种协作模式:

**模式一:流水线(项目一)** —— 固定顺序,前一个的输出是后一个的输入:

```
景点Agent搜景点 → 天气Agent查天气 → 酒店Agent搜酒店 → 规划Agent汇总生成JSON
```

协作逻辑写死在 Python 代码里(依次调用 4 个 `agent.run()`),**不是** Agent 自己决定找谁。简单、可控、好调试,是生产环境最常用的模式。

**模式二:规划-执行(项目二)** —— 第一个 Agent 生成任务清单(TODO list),然后逐个(实际是多线程并发)执行:

```
规划Agent拆出N个子任务 → 每个子任务:搜索→总结Agent提炼 → 报告Agent汇总成文
```

任务数量是动态的(模型根据题目复杂度决定拆几个),比流水线灵活一档。

**模式三:独立个体(项目三)** —— 每个 NPC 是完全独立的 Agent,有自己的提示词、记忆库、好感度,互不通信,只和玩家交互。

> 面试级别的总结:多Agent系统的"协作智能"大多是**工程编排**(orchestration)而非模型自发协作。先想清楚数据流,再决定用几个 Agent。

## 5. 关于"框架裁剪"的心法

看 HelloAgents 源码(仓库 `HelloAgents/` 目录)时不要试图读完,项目只用到:

```
hello_agents
├── SimpleAgent            # 项目一、三
├── ToolAwareSimpleAgent   # 项目二
├── HelloAgentsLLM         # 全部
├── tools
│   ├── MCPTool            # 项目一
│   ├── ToolRegistry       # 项目二
│   └── builtin/note_tool  # 项目二(记笔记工具)
└── memory
    ├── MemoryManager      # 项目三
    └── MemoryConfig       # 项目三
```

其余的(A2A 协议、训练、评估……)是教材其他章节的内容,跳过不影响做项目。

---

## 自测清单

不看本文,能回答以下问题再进入下一篇:

1. LLM API 的 messages 里,`system` / `user` / `assistant` 三种角色分别装什么?
2. "LLM 是无状态的",那多轮对话是怎么实现的?记忆系统又是解决什么问题?
3. `[TOOL_CALL:amap_maps_weather:city=北京]` 这行字是谁输出的?谁负责执行?执行结果去了哪?
4. MCP 解决了什么问题?`MCPTool` 的 `server_command` 参数是干嘛的?
5. 项目一的四个 Agent 用的是同一个类,它们的行为差异来自哪里?

动手练习(强烈建议):在上一篇的 `agent-env-test` 目录里,创建两个不同 `system_prompt` 的 `SimpleAgent`,让第一个的输出作为第二个的输入,体验一次最小的"流水线协作"。

下一篇:[04 · Web 开发基础:FastAPI 与前后端分离](./04-Web开发基础-FastAPI与前后端分离.md)
