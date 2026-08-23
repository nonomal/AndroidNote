# 04 · Web 开发基础:FastAPI 与前后端分离

> 三个项目的"外壳"都是同一套结构:**FastAPI 后端 + 一个独立前端**(Vue 或 Godot)。本篇把这套结构讲透。学完后,项目代码里的 `@app.post`、`BaseModel`、CORS、SSE 你都能看懂在干嘛。

## 1. 前后端分离是什么意思

```
浏览器里的Vue页面 / Godot游戏          Python进程
┌─────────────────┐    HTTP请求     ┌──────────────────┐
│     前端         │ ─────────────→ │   后端(FastAPI)   │
│  负责界面和交互   │ ←───────────── │  负责调LLM、算逻辑 │
└─────────────────┘    JSON响应     └──────────────────┘
   localhost:5173                     localhost:8000
```

- 前端和后端是**两个独立的程序**,分别启动、分别占一个端口,通过 HTTP + JSON 通信。
- 这就是为什么每个项目都要开两个终端:一个跑 `python main.py`(后端),一个跑 `npm run dev`(前端)。
- 项目三的"前端"是 Godot 游戏,但通信方式一模一样:游戏里的 GDScript 发 HTTP 请求给 FastAPI。

## 2. HTTP 五分钟速成

一次 HTTP 请求包含:**方法 + 路径 + 头 + 体**。

| 方法 | 语义 | 项目里的例子 |
| --- | --- | --- |
| GET | 取数据(参数在 URL 里) | `GET /npcs/张三/affinity?player_id=player` |
| POST | 提交数据(参数在请求体里,JSON) | `POST /chat`,体:`{"npc_name":"张三","message":"你好"}` |
| PUT | 修改数据 | `PUT /npcs/张三/affinity?affinity=80` |
| DELETE | 删除数据 | `DELETE /npcs/张三/memories` |

响应里最重要的是**状态码**:

- `200` 成功
- `404` 路径/资源不存在(NPC 名字打错了)
- `422` 请求体格式不对(FastAPI 特色,字段缺了或类型错了)
- `500` 服务器内部出错(去看后端终端的报错)

## 3. FastAPI:把 Python 函数变成网络接口

最小可运行示例(建议真的跑一遍):

```python
# demo.py
from fastapi import FastAPI

app = FastAPI(title="我的第一个API")

@app.get("/hello")
def say_hello(name: str = "世界"):
    return {"message": f"你好, {name}"}
```

```bash
uv pip install fastapi "uvicorn[standard]"
uvicorn demo:app --reload        # --reload: 改代码自动重启
```

浏览器访问 `http://127.0.0.1:8000/hello?name=张三`,看到 JSON 响应。

三件事值得注意:

1. `@app.get("/hello")` 装饰器把函数**注册**成接口,路径是 `/hello`。
2. 函数参数 `name: str` 自动从 URL 查询参数解析,类型注解在这里是**强制校验**。
3. 返回的 dict 自动序列化成 JSON(**序列化** = 把内存里的对象转成可传输/可存储的文本,反过来叫反序列化)。

**杀手锏:自动文档。** 访问 `http://127.0.0.1:8000/docs`,FastAPI 自动生成可交互的 API 文档页面,能直接在网页上测试每个接口。**三个项目调试后端,全靠这个页面**,不用装 Postman。

## 4. Pydantic:定义"数据长什么样"

POST 请求的 JSON 体,用 Pydantic 的 `BaseModel` 定义结构:

```python
from pydantic import BaseModel, Field

class ChatRequest(BaseModel):
    npc_name: str                                # 必填
    message: str = Field(..., min_length=1)      # 必填,且不能为空串
    player_id: str = "player"                    # 选填,有默认值

@app.post("/chat")
def chat(request: ChatRequest):                  # FastAPI自动解析+校验JSON体
    return {"reply": f"{request.npc_name}收到: {request.message}"}
```

前端发来的 JSON 少了字段、类型不对,FastAPI 直接返回 422 和详细错误说明,你的函数根本不会被执行——**校验是白送的**。

Pydantic 模型可以嵌套,项目一的行程计划就是层层嵌套的模型(`app/models/schemas.py`):

```python
class Location(BaseModel):
    longitude: float
    latitude: float

class Attraction(BaseModel):
    name: str
    address: str
    location: Location          # 嵌套模型
    visit_duration: int
    ticket_price: float = 0

class DayPlan(BaseModel):
    date: str
    attractions: list[Attraction]   # 模型列表

class TripPlan(BaseModel):
    city: str
    days: list[DayPlan]
```

最妙的用法在项目一的解析环节:LLM 输出 JSON 字符串 → `json.loads` 转成 dict → **`TripPlan(**data)` 一行完成整棵树的校验**。LLM 编的数据缺字段,这里立刻抛异常,走备用方案——Pydantic 成了 LLM 输出的"质检员"。

## 5. CORS:前端连不上后端的头号原因

浏览器有个安全策略:`localhost:5173`(前端)的页面,默认**不允许**请求 `localhost:8000`(后端)——端口不同就算"跨域"。后端必须显式声明"我允许谁来访问":

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],   # 允许的前端地址,或 ["*"] 全放行
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

三个项目的后端都有这段代码。**症状识别:前端页面报错、浏览器控制台(F12)出现红色的 "CORS policy" 字样 → 检查后端这段配置和前端实际端口是否一致。**

## 6. SSE:后端向浏览器"直播"进度(项目二核心)

普通接口是"一问一答":请求 → 等 → 一次性拿到完整响应。但项目二的研究流程要几分钟,用户不能对着白屏干等。**SSE(Server-Sent Events)** 让后端在一个连接里持续推送多条消息:

```
浏览器: POST /research/stream
后端:   data: {"type": "status", "message": "初始化研究流程"}\n\n
        data: {"type": "todo_list", "tasks": [...]}\n\n
        data: {"type": "task_summary_chunk", "content": "根据..."}\n\n
        ...(持续几分钟)
        data: {"type": "done"}\n\n
```

格式约定:每条消息以 `data: ` 开头、以**两个换行**结尾。FastAPI 端用生成器(第 01 篇讲的 `yield`)+ `StreamingResponse` 实现,这是项目二 `main.py` 的真实写法(简化):

```python
from fastapi.responses import StreamingResponse
import json

@app.post("/research/stream")
def stream_research(payload: ResearchRequest):
    def event_iterator():
        for event in agent.run_stream(payload.topic):      # 生成器:边研究边产出事件
            yield f"data: {json.dumps(event, ensure_ascii=False)}\n\n"

    return StreamingResponse(
        event_iterator(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "Connection": "keep-alive"},
    )
```

把整条链路串起来:**Agent 的 `run_stream()` 每 yield 一个事件 → FastAPI 立刻推给浏览器 → 前端 JS 收到就更新界面**。ChatGPT 那种"逐字蹦出"的效果,原理相同。

## 7. 前端(Vue 3)只需要看懂,不需要会写

项目一、二的前端已经写好,你只需 `npm install && npm run dev`。但看懂结构对排错很有帮助:

```
frontend/
├── index.html            # 唯一的HTML,Vue把界面"挂载"进去
├── package.json          # 前端的requirements.txt
├── vite.config.ts        # 构建工具配置(端口、代理都在这)
└── src/
    ├── main.ts           # 入口,创建Vue应用
    ├── App.vue           # 根组件
    ├── services/api.ts   # ★所有请求后端的代码集中在这
    ├── types/            # TypeScript类型(对应后端的Pydantic模型)
    └── views/            # 页面(Home.vue表单页、Result.vue结果页)
```

一个 `.vue` 文件 = 结构 + 逻辑 + 样式三合一:

```vue
<template>            <!-- HTML结构,{{变量}}是插值 -->
  <button @click="submit">生成计划</button>
</template>

<script setup lang="ts">   /* 逻辑,TypeScript */
import { ref } from 'vue'
const city = ref('北京')        // 响应式变量:值变了界面自动刷新
function submit() { /* 调 services/api.ts 里的函数 */ }
</script>

<style scoped>        /* 样式,只作用于本组件 */
button { color: teal; }
</style>
```

**排错时只盯两个文件:** `services/api.ts`(请求的 URL 对不对、端口对不对)和 `vite.config.ts`(有没有配置代理转发)。前端报"网络错误"十有八九是后端没启动或地址不匹配。

## 8. 三个项目的后端结构对照

学会一个,另外两个就是换汤不换药:

```
项目一(分层最规范)           项目二(src布局)         项目三(扁平)
backend/app/                 backend/src/            backend/
├── api/main.py    入口      ├── main.py    入口     ├── main.py    入口+路由
├── api/routes/    路由      ├── agent.py   编排     ├── agents.py  NPC Agent
├── agents/        Agent     ├── services/  服务     ├── state_manager.py
├── services/      外部服务  ├── models.py  模型     ├── relationship_manager.py
├── models/        Pydantic  ├── prompts.py 提示词   ├── models.py  Pydantic
└── config.py      配置      └── config.py  配置     └── config.py  配置
```

共同规律:**入口文件建 FastAPI 应用 + 挂 CORS;路由函数薄薄一层,校验参数就转手调 Agent;Agent 与业务逻辑独立成模块;配置统一从 `.env` 读。** 这是可以带走复用到任何项目的结构。

---

## 自测清单

1. 为什么每个项目都要开两个终端?两个进程各占哪个端口?
2. 前端控制台报 "blocked by CORS policy",应该改前端还是后端?改哪段代码?
3. FastAPI 返回 422 意味着什么?去哪里看具体是哪个字段出了问题?(提示:响应体里写得清清楚楚)
4. SSE 一条消息的文本格式长什么样?后端用什么 Python 语法"边算边发"?
5. 动手:把第 3 节的 `demo.py` 扩展一个 `POST /echo` 接口,接收 `{"text": "..."}` 并原样返回,用 `/docs` 页面测试它。

下一篇:[05 · 项目一:智能旅行助手从 0 到 1](./05-项目一-智能旅行助手从0到1.md)
