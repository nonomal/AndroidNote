# 11 Docker 与上线: 从"我这能跑"到"别人也能跑"

> **一句话总结**: 上线的本质不是把代码传到服务器, 是把"能跑起来"这件事从你的记忆里搬到文件里 —— 多阶段构建管住镜像体积, 数据卷管住数据不丢, Nginx 的 `proxy_buffering off` 管住流式不被缓冲吞掉, 而这三件事只有写进文件才能在半年后依然生效。

## 本章你能学到什么

1. 为什么必须用 Docker, 以及**它到底解决了哪个具体问题** (不是"环境一致"这种空话)
2. 多阶段构建: 为什么分两段, 以及**分段之后镜像里少了什么**
3. `COPY` 的顺序决定构建速度 —— 改一行业务代码不该重装 torch
4. 为什么 embedding 模型要**在构建期下载进镜像**, 而不是运行时拉
5. `HF_HUB_OFFLINE=1` 的真正作用: 让"离线部署"从承诺变成约束
6. 非 root 运行、`VOLUME`、`HEALTHCHECK` 各自防的是什么事故
7. `entrypoint.sh` 为什么必须**幂等**, 以及 `exec "$@"` 那一行不写会怎样
8. `docker-compose.yml` 里 `start_period: 90s` 是怎么从一次真实的"明明能跑却起不来"里来的
9. **SSE 过 Nginx 的三件套** —— 本章最贵的一个坑, 直连正常、过代理才复现
10. 为什么不能给 `text/event-stream` 开 gzip
11. `--workers 1` 不是偷懒, 是会话存在进程内存里的必然结果; 以及要扩容得先改什么
12. 上线清单: 鉴权、限流、密钥、日志、备份, 以及**上线前必须跑一遍的冒烟脚本**

---

## 一、为什么要 Docker

### 1.1 先说清楚它解决的具体问题

"环境一致"这个说法太抽象了。我们的项目里, 换一台机器实际会出什么事:

| 具体问题 | 不用 Docker 时会发生什么 |
| --- | --- |
| Python 版本 | 项目用 3.12 的语法 (`list[str]` 这类)。别人机器上是 3.9, 报 `TypeError: 'type' object is not subscriptable` |
| torch 装不上 | `sentence-transformers` 拖进 torch, 不同平台不同轮子, Windows 上常年装失败 |
| embedding 模型 | 第一次启动去 HuggingFace 拉 103MB, 国内不挂镜像站基本拉不动 |
| 向量库路径 | `CHROMA_DIR` 在你机器上是相对路径, 在服务器上工作目录不同, 于是"知识库是空的" |
| 忘了灌库 | 服务能起、接口能调、就是选不出题, 报错出现在完全无关的地方 |

**Docker 的价值是把这五件事的答案固化成一个文件, 而不是固化在你的记忆里。**半年后你自己都会忘记"要先跑一遍 `python -m app.rag.index`"。

### 1.2 我们要做出来的东西

```
docker/
├── Dockerfile           # 怎么把项目打成镜像
├── entrypoint.sh        # 容器启动时先干什么 (灌库)
├── docker-compose.yml   # 应用 + Nginx + 数据卷, 一条命令拉起
└── nginx.conf           # 反向代理, 核心是 SSE 不缓冲
```

四个文件, 一条命令:

```bash
docker compose -f docker/docker-compose.yml up -d
```

然后浏览器打开 `http://localhost:8080` 就能面试。

> ⚠️ **关于本章的验证程度, 我要先说清楚。**
>
> 本章**没有 Checkpoint 贴构建日志**, 因为写作环境里 Docker daemon 没有运行 (`failed to connect to the docker API at unix:///Users/c/.docker/run/docker.sock`)。我做到的验证是:
>
> - `docker compose config` **真的跑通了** (exit 0), 说明 compose 文件语法正确、变量插值和路径解析符合预期 —— 这一节有真实输出。
> - **`nginx.conf` 的语法我没有验证过。**机器上没有 nginx 二进制, 容器也起不来, 所以 `nginx -t` 没跑。配置项的含义是可靠的 (来自官方文档和这个项目的真实经验), 但**请你自己 `docker compose up` 之后跑一次 `docker compose exec nginx nginx -t`**。
> - 镜像构建、健康检查转 healthy、SSE 过代理的实际表现, 都需要你在有 Docker 的环境里自己验一遍。
>
> 这和第 5 节 (租卡 QLoRA) 是同一种处理方式: **没跑过的就说没跑过。**你按本章操作时遇到的第一个报错, 很可能就出在我没能验证的那几行里。

---

## 二、Dockerfile: 一行一行讲

先看完整的, 再拆开讲。

```dockerfile
# ---------- builder ----------
FROM python:3.12-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:0.11.32 /uv /usr/local/bin/uv

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never

WORKDIR /build

COPY pyproject.toml uv.lock* ./
RUN uv venv /opt/venv && \
    VIRTUAL_ENV=/opt/venv uv pip install --no-cache -r pyproject.toml

ENV HF_ENDPOINT=https://hf-mirror.com \
    HF_HOME=/opt/hf
RUN --mount=type=cache,target=/root/.cache/huggingface \
    /opt/venv/bin/python -c "\
from sentence_transformers import SentenceTransformer; \
SentenceTransformer('BAAI/bge-small-zh-v1.5', cache_folder='/opt/hf')" && \
    du -sh /opt/hf
```

```dockerfile
# ---------- runtime ----------
FROM python:3.12-slim AS runtime

RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

RUN useradd -m -u 10001 app

COPY --from=builder /opt/venv /opt/venv
COPY --from=builder --chown=app:app /opt/hf /opt/hf

ENV PATH=/opt/venv/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    HF_HOME=/opt/hf \
    HF_HUB_OFFLINE=1 \
    CHROMA_DIR=/data/chroma \
    DB_PATH=/data/app.db \
    CORPUS_DIR=/app/corpus

WORKDIR /app
COPY --chown=app:app app/ ./app/
COPY --chown=app:app corpus/ ./corpus/
COPY --chown=app:app eval/ ./eval/
COPY --chown=app:app static/ ./static/
COPY --chown=app:app scripts/ ./scripts/
COPY --chmod=755 docker/entrypoint.sh /usr/local/bin/entrypoint.sh

RUN mkdir -p /data && chown -R app:app /data
VOLUME ["/data"]

USER app
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
    CMD curl -fsS http://127.0.0.1:8000/healthz | grep -q '"status":"ok"' || exit 1

ENTRYPOINT ["entrypoint.sh"]
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

### 2.1 为什么分成 builder 和 runtime 两段

`sentence-transformers` 会拖进 torch。装完之后, pip 的下载缓存、wheel 解包的临时文件、编译产物加起来有 **1GB 多**。这些东西**运行时一个字节都不需要**。

多阶段构建的逻辑很朴素:

- **builder 段**: 随便造, 装编译器、留缓存、下模型, 不心疼。
- **runtime 段**: 从零开一个干净的 `python:3.12-slim`, 只把两样东西**拷**过来 —— `/opt/venv` (装好的依赖) 和 `/opt/hf` (模型权重)。

```dockerfile
COPY --from=builder /opt/venv /opt/venv
COPY --from=builder --chown=app:app /opt/hf /opt/hf
```

`--from=builder` 就是"从上一段里拷"。**builder 段里除了这两个目录之外的一切 —— 包括 uv 本身、pip 缓存、apt 列表 —— 都不会进入最终镜像。**

**如果不分段呢?** 你得在同一段里手动 `rm -rf` 各种缓存, 而且 Docker 的分层机制会让你的删除白费力气: **前一层写进去的文件, 后一层删掉之后依然占镜像体积** (它还在那一层里)。多阶段构建是唯一干净的解法。

### 2.2 COPY 的顺序: 一个能省掉几百次五分钟的细节

看这两行的位置:

```dockerfile
COPY pyproject.toml uv.lock* ./          # ← 只拷依赖声明
RUN uv venv /opt/venv && \
    VIRTUAL_ENV=/opt/venv uv pip install --no-cache -r pyproject.toml
```

**为什么不是直接 `COPY . .` 然后装依赖?**

因为 Docker 按层缓存, 而**一层的缓存失效之后, 它后面所有层的缓存全部失效**。

- 写成 `COPY . .` → 你改一个字符的业务代码, 这一层的内容变了 → 缓存失效 → 后面的 `pip install` 重新执行 → **重装 torch, 五分钟。**
- 写成先拷 `pyproject.toml` → 改业务代码时这一层内容没变 → 缓存命中 → 依赖层直接复用 → **十秒。**

`uv.lock*` 那个星号是故意的: **没有 lock 文件时通配符匹配为空, `COPY` 不会报错。**写成 `uv.lock` 的话, 没生成 lock 文件的人第一次构建就是一句 `file not found`。

**这条规则可以抽象成一句话: Dockerfile 里的指令要按"变化频率从低到高"排列。**最不常变的 (系统包、依赖) 放最上面, 最常变的 (业务代码) 放最下面。

### 2.3 为什么模型要在构建期就下载

```dockerfile
ENV HF_ENDPOINT=https://hf-mirror.com \
    HF_HOME=/opt/hf
RUN --mount=type=cache,target=/root/.cache/huggingface \
    /opt/venv/bin/python -c "\
from sentence_transformers import SentenceTransformer; \
SentenceTransformer('BAAI/bge-small-zh-v1.5', cache_folder='/opt/hf')" && \
    du -sh /opt/hf
```

**不这么做的后果**: 容器起来了, `/healthz` 也许还能通, 但**第一个真实用户的第一个请求**会触发去 HuggingFace 拉 103MB 模型 —— 用户等 90 秒, 而且很可能因为网络失败直接 500。

**这是"第一个用户替你测试"的典型场景, 而它完全可以在构建期避免。**

三个细节:

- **`HF_ENDPOINT=https://hf-mirror.com`** —— 国内直连 HuggingFace 基本不通。这一行让构建从"卡住不动"变成几十秒。
- **`--mount=type=cache`** —— BuildKit 的缓存挂载。它让你重复构建镜像时不用重新下载模型, 但**这个缓存不会进入镜像层**, 所以不占体积。这是"既要缓存又不要体积"的标准解法。
- **`du -sh /opt/hf`** —— 构建日志里打印一下模型实际占了多少。加这一行的理由: 镜像变大时你能立刻知道是不是模型的锅, 而不是去猜。

### 2.4 `HF_HUB_OFFLINE=1`: 把承诺变成约束

```dockerfile
ENV HF_HOME=/opt/hf \
    HF_HUB_OFFLINE=1
```

模型已经在镜像里了, 那这一行是干什么的?

**它禁止 huggingface 库联网。**不加这一行, transformers 每次加载模型都会尝试连 HF 检查有没有更新 —— 于是:

- 内网部署时, 每次启动都要等一次网络超时;
- 更糟: 某天 HF 返回了一个"有新版本", 库可能去下载, **你的线上模型悄悄换了一个版本**, 而检索结果跟着变了。

**"数据不出内网"这种要求, 靠"我们不会联网"的承诺是不够的, 要靠让它联不了网。**这和第 10 章第 7.1 节把模型调用收口在一个文件里是同一个思路: 把正确性做成结构上的必然, 而不是行为上的自觉。

### 2.5 非 root 运行

```dockerfile
RUN useradd -m -u 10001 app
# ... 所有 COPY 都带 --chown=app:app
USER app
```

**为什么值得多写这几行:** 容器不是安全边界, 它是隔离机制。真的出现容器逃逸漏洞时, **容器内是 root 还是普通用户, 决定了攻击者拿到的是整台宿主机还是一个目录。**

三个配套细节:

- **`-u 10001`** 显式指定 UID。不指定的话不同基础镜像分配的 UID 不同, 挂 volume 时宿主机上的文件属主会变得不可预测。
- **每个 `COPY` 都带 `--chown=app:app`** —— 不带的话文件属主是 root, 切到 `app` 用户后可能读不了 (更麻烦的是: 大部分文件只读也能跑, 于是你直到需要写文件那天才发现)。
- **`USER app` 必须放在所有 `RUN` 之后。**放前面的话后续 `apt-get` 会因为没权限失败。

### 2.6 `VOLUME ["/data"]`: 不挂就是每次重启清空数据

```dockerfile
ENV CHROMA_DIR=/data/chroma \
    DB_PATH=/data/app.db
RUN mkdir -p /data && chown -R app:app /data
VOLUME ["/data"]
```

容器的文件系统是**临时的**: 容器一删, 里面写的所有东西都没了。我们有两样必须活下来的数据:

| 数据 | 路径 | 丢了会怎样 |
| --- | --- | --- |
| 向量库 (Chroma) | `/data/chroma` | 知识库变空 → 选不出题 → 每次重启要重算十几秒的向量 |
| 业务库 (SQLite) | `/data/app.db` | 所有面试记录、错题本、用量统计清零 |

**把两者放在同一个目录下, 是为了只需要挂一个 volume。**如果一个在 `/app/chroma` 一个在 `/tmp/app.db`, 你就得挂两个, 而**漏挂一个的症状是"数据只丢了一半"** —— 比全丢更难查。

注意环境变量和路径的配合: `CHROMA_DIR` / `DB_PATH` 在第 1 章就是从配置读的, 所以这里改成绝对路径**零代码改动**。

### 2.7 `HEALTHCHECK`: 让"起来了"和"能用了"变成两件事

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
    CMD curl -fsS http://127.0.0.1:8000/healthz | grep -q '"status":"ok"' || exit 1
```

第 8 章我们写的 `/healthz` 在这里终于有了用处。它返回的不只是"进程活着", 还有**知识库块数** —— 库是空的时候 `status` 是 `degraded`。

所以这个 healthcheck 检查的是: **进程活着 + 知识库不是空的。**这正是"能用了"的定义。

三处细节:

- **`grep -q '"status":"ok"'`** —— 不能只看 HTTP 200。`/healthz` 在 degraded 状态下依然返回 200 (它是健康报告, 不是断言), 只看状态码会把"服务起了但选不出题"判成健康。
- **`curl` 必须装。**这就是 runtime 段那句 `apt-get install curl` 的唯一理由。**不装的话 healthcheck 永远是 `starting`** —— 而 compose 里 Nginx 的 `depends_on: service_healthy` 会永远等下去。
- **`--start-period=40s`** —— 首次启动要加载 embedding 模型, 冷启动十几秒很正常。start_period 内的失败不计入 `retries`。

### 2.8 `--workers 1` 不是偷懒

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

第 8 章的会话状态 (`SESSIONS` 字典) 存在**进程内存**里。多 worker 时每个 worker 是独立进程, 有自己的一份字典:

1. 用户 POST `/api/interview/start` → 路由到 worker A → 会话建在 A 的内存里
2. 用户 POST `/api/interview/answer` → 路由到 worker B → **B 不认识这个 session_id** → 报"会话不存在"

**症状是随机的** —— 两个 worker 时约一半请求失败, 四个 worker 时约 75% 失败。这种"时好时坏"的 bug 最消耗人。

**想扩容, 顺序是: 先把会话搬到 Redis, 再加 worker。**顺序反了就是上面那个 bug。单进程能撑多少? 我们的瓶颈是 LLM 调用 (第 10 章实测本地模型单次判分 5.28 秒), 而那是 IO 等待, asyncio 单进程能并发扛住几十个会话。**在你真的量到 CPU 打满之前, 单 worker 完全够用。**

### 2.9 `.dockerignore`: 一个容易漏掉的文件

参考实现里**没有**这个文件, 这是个应该补的疏漏。加上它:

```
.venv/
__pycache__/
*.pyc
.git/
.env
data/
chroma/
*.db
docs/
.pytest_cache/
.ruff_cache/
saves/
finetune/data/
```

**两个理由, 第二个是安全问题:**

1. **构建速度。**`COPY` 时 Docker 要把整个上下文发给 daemon。本地 `.venv` 有 1GB+、`.git` 有几十 MB, 不排除就是每次构建先传 1GB。
2. **`.env` 绝对不能进镜像。**镜像是要推到仓库的, 而**镜像的每一层都可以被任何拿到镜像的人解包读取** —— 就算你在后续层里 `rm` 掉它, 前一层里的密钥依然在。这是最经典的密钥泄露方式之一。

**密钥永远通过环境变量或 secret 机制在运行时注入, 不进镜像。**下一节 compose 里的 `env_file` 就是运行时注入。

---

## 三、entrypoint.sh: 启动前先保证知识库不是空的

```bash
#!/bin/sh
set -e

echo "[entrypoint] 检查知识库..."
COUNT=$(python -c "
from app.rag.index import KnowledgeIndex
print(KnowledgeIndex().count())
" 2>/dev/null || echo 0)

if [ "$COUNT" -eq 0 ]; then
    echo "[entrypoint] 知识库为空, 开始灌库 (首次启动要算向量, 约 10~30 秒)..."
    python -m app.rag.index "${CORPUS_DIR:-/app/corpus}"
else
    echo "[entrypoint] 知识库已有 ${COUNT} 块, 跳过灌库"
fi

echo "[entrypoint] 启动服务..."
exec "$@"
```

### 3.1 为什么需要它

**镜像里带了语料 (`corpus/`), 但向量库在 `/data` 这个 volume 上。**第一次 `docker compose up` 时 volume 是空的, 于是:

1. 进程能起来 (FastAPI 不依赖知识库非空)
2. `/healthz` 返回 `degraded`
3. healthcheck 判定不健康
4. Nginx 的 `depends_on: service_healthy` 永远等不到
5. **结果: 服务起了, 但你访问 8080 端口什么都没有**

这个故障链很长, 而起点只是"volume 是空的"。entrypoint 把它掐死在第一步。

### 3.2 幂等是关键

`if [ "$COUNT" -eq 0 ]` 这个判断不能省。**不判断直接灌库的后果:**

- 每次重启都重算一遍向量 → 冷启动多等一分钟
- 更严重: 重新灌库可能让 chunk_id 变化, 而**用户历史面试记录里存的 chunk_id 就对不上了** —— 错题本点进去是空的

**"容器可能随时被重启"是一个必须假设的前提** (OOM、宿主机重启、滚动更新都会触发)。所以启动脚本必须能安全地重复执行任意多次。

### 3.3 `2>/dev/null || echo 0` 这一段

```bash
COUNT=$(python -c "..." 2>/dev/null || echo 0)
```

如果 `KnowledgeIndex()` 因为目录不存在而抛异常怎么办? 不兜底的话 `COUNT` 是空字符串, 下一行 `[ "" -eq 0 ]` 会报语法错误, 加上 `set -e` 就是**容器直接退出**。

兜底成 `0` 的语义是: **拿不到数量就当它是空的, 去灌一次库。**这是一个安全的默认值 —— 最坏情况是多花二十秒, 而不是起不来。

### 3.4 `exec "$@"` 那一行不写会怎样

```bash
exec "$@"     # 而不是直接 "$@"
```

`exec` 让 uvicorn **替换掉** shell 进程, 从而成为 PID 1。

不加 `exec` 的话, PID 1 是 shell, uvicorn 是它的子进程。`docker stop` 发送的 `SIGTERM` 送给 PID 1 也就是 shell —— **而 sh 收到 SIGTERM 不会转发给子进程。**于是:

1. Docker 发 SIGTERM, 没人理
2. 等 10 秒 (默认超时)
3. Docker 发 `SIGKILL`, 整个容器被硬杀

**硬杀的后果**: 正在写的 SQLite 事务可能留下 `-wal` 残留文件, 正在进行的面试会话直接消失, 用户看到连接中断。

**一个 `exec` 关键字的区别, 就是优雅停机和 `kill -9` 的区别。**

---

## 四、docker-compose.yml: 一条命令拉起全部

```yaml
name: android-interviewer

services:
  app:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    image: android-interviewer:1.0.0
    restart: unless-stopped
    expose:
      - "8000"
    env_file:
      - path: ../.env
        required: false
    environment:
      OFFLINE: "${OFFLINE:-false}"
      CHROMA_DIR: /data/chroma
      DB_PATH: /data/app.db
      HF_HOME: /opt/hf
      HF_HUB_OFFLINE: "1"
    volumes:
      - itv-data:/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:8000/healthz | grep -q '\"status\":\"ok\"'"]
      interval: 30s
      timeout: 5s
      start_period: 90s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 2g

  nginx:
    image: nginx:1.29-alpine
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy

volumes:
  itv-data:
```

### 4.1 `name:` 必须显式写

compose 默认拿**所在目录名**当项目名。而我们这个文件在 `docker/` 目录下 —— 不写 `name:` 的话, 所有容器和 volume 都会叫 `docker_app`、`docker_itv-data`。

**症状**: 你在同一台机器上跑第二个也放在 `docker/` 下的项目, **两个项目的 volume 撞名, 数据互相污染。**

### 4.2 `env_file` 的 `required: false`

```yaml
env_file:
  - path: ../.env
    required: false
```

写成简写形式 `- ../.env` 的话, `.env` 不存在时 compose 直接报 `env file not found` 拒绝启动。

**而 `.env` 是 `.gitignore` 掉的**, 所以每一个新 clone 项目的人第一次 `up` 都会撞到这个错, 白卡十分钟。

配合下面的 `environment:` 兜底值 (`OFFLINE: "${OFFLINE:-false}"` 等), 效果是: **没建 `.env` 也能起来, 走离线模式。**这比一句报错友好得多 —— 一个能起来的降级服务, 比一个起不来的完整服务更容易让人继续往下走。

### 4.3 `expose` 而不是 `ports`

```yaml
expose:
  - "8000"        # 只在 compose 内部网络可见
```

`ports: ["8000:8000"]` 会把端口绑到宿主机, 意味着**绕过 Nginx 也能直连**。这有两个问题:

1. 直连绕过了 Nginx 上的限流、超时、请求体大小限制;
2. **直连时 SSE 是正常的** (第 5 节会讲), 于是你在直连下测试一切完美, 用户从 8080 进来却看到流式失效 —— **然后你会去查前端代码, 而问题在 nginx.conf。**

想调试时临时加一行 `ports` 就好, **但不要留在文件里。**

### 4.4 `start_period: 90s` 比 Dockerfile 里的 40s 更长

Dockerfile 里写的是 40s, compose 里是 90s (compose 的设置覆盖镜像的)。为什么要放宽?

因为**首次启动要多做一件事: 灌库。**entrypoint 检测到 volume 是空的, 会算 25 块语料的向量。加上加载 embedding 模型, 首启动十几到几十秒都正常。

**给太短的后果是最典型的"明明能跑却起不来"**: healthcheck 在服务还在灌库时就开始判, 连续失败 3 次 → 容器被判定 unhealthy → `restart: unless-stopped` 把它重启 → 重启后又是灌库中 → **无限重启循环**, 而日志里每次都能看到"正在灌库", 看起来一切正常。

**遇到"容器反复重启但日志没有报错"时, 第一个要怀疑的就是 `start_period` 太短。**

### 4.5 `depends_on: condition: service_healthy`

```yaml
depends_on:
  app:
    condition: service_healthy
```

不加 `condition` 的话, `depends_on` 只保证"app 容器启动了", 不保证"app 能服务了"。于是每次 `up` 或滚动重启的头几十秒, 用户访问会看到 **502 Bad Gateway**。

**这一行的前提是 healthcheck 必须真的能转成 healthy** —— 也就是第 2.7 节那个 `curl` 必须装。**curl 没装 + 这一行, 组合起来就是 Nginx 永远不启动。**这两处的耦合很不显眼, 排查时容易只看一边。

### 4.6 内存限制 2g

```yaml
deploy:
  resources:
    limits:
      memory: 2g
```

`sentence-transformers` 加载模型时内存会冲到 1GB 上下, 给 2GB 有余量。

**为什么要设上限?** 不设的话, 内存泄漏或者一次异常的大批量请求会把宿主机的内存吃光, **拖垮的不只是这个容器, 是整台机器上所有服务。**设了上限, 最坏情况是这个容器被 OOM kill 然后重启 —— 故障被限制在一个容器里。

### 4.7 ✅ Checkpoint 1: 校验 compose 配置

这是本章唯一一个真实执行的 Checkpoint。Docker daemon 没运行, 但 `docker compose config` 只做解析和插值, 不需要 daemon:

```bash
cd docker && docker compose config --quiet; echo "exit=$?"
```

真实输出:

```
exit=0
```

`--quiet` 只校验不打印。想看解析后的完整结果 (**变量插值、相对路径展开成绝对路径、简写展开成完整形式**都能看到):

```bash
docker compose config
```

真实输出的关键片段:

```yaml
name: android-interviewer
services:
  app:
    build:
      context: /tmp/ref/ai              # ← .. 被解析成了绝对路径
      dockerfile: docker/Dockerfile
    environment:
      OFFLINE: "false"                  # ← ${OFFLINE:-false} 的兜底值生效了
    healthcheck:
      start_period: 1m30s               # ← 90s 被规范化成 1m30s
  nginx:
    depends_on:
      app:
        condition: service_healthy
        required: true
    volumes:
      - type: bind
        source: /tmp/ref/ai/docker/nginx.conf
        read_only: true
volumes:
  itv-data: {}
```

**三个值得确认的点:**

1. **`context: /tmp/ref/ai`** —— `..` 被正确解析到项目根目录。写错的话会拿 `docker/` 当构建上下文, 结果是 `COPY app/ ./app/` 找不到文件。
2. **`OFFLINE: "false"`** —— 没有 `.env` 时兜底值确实生效了, 印证第 4.2 节。
3. **`start_period: 1m30s`** —— compose 的 90s 覆盖了镜像里的 40s。

**`docker compose config` 应该成为你上线流程的第一步。**它免费、不需要 daemon、一秒返回, 而且能抓住所有"变量没定义"和"路径写错"的问题 —— 这两类占了 compose 文件错误的大多数。第 12 章会把它放进 CI。

---

## 五、nginx.conf: 本章最贵的一个坑

> 提醒: 这一节的配置**我没有用 `nginx -t` 验证过** (环境里没有 nginx)。配置项的含义可靠, 但请你 `up` 之后跑一次 `docker compose exec nginx nginx -t`。

### 5.1 先说结论: `proxy_buffering off`

**如果这一章只记一件事, 记这个。**

Nginx 默认开启 `proxy_buffering`。它的行为是: **等后端攒够一块数据 (或者响应结束) 才转发给客户端。**对普通的 JSON 接口这是优化 —— 减少小包、让后端更快释放连接。

**对 SSE 这是灾难。**第 8 章我们精心设计的事件流:

```
event: grade      ← 评分先到, 用户立刻看到反馈
event: question   ← 下一题随后到
```

过了默认配置的 Nginx 之后, 变成: **面试全部结束时, 所有事件一次性刷出来。**

**这个坑之所以贵, 是因为它的复现条件很窄:**

| 环境 | 表现 |
| --- | --- |
| `uvicorn` 直连 8000 | 流式完全正常 |
| pytest / TestClient (第 8 章的测试) | 正常 —— TestClient 不过代理 |
| `scripts/smoke_api.py` 打 8000 | 正常 |
| **过 Nginx 打 8080** | **卡半天, 然后一次性刷出来** |

所以你的测试全绿、本地全对, **只有真实用户会遇到**。而症状 ("界面卡着不动") 会让你第一反应去查前端 JS —— 而前端一个字都没错。

这就是第 4.3 节坚持用 `expose` 而不是 `ports` 的原因: **让你自己也只能从 8080 进来, 和用户走同一条路。**

### 5.2 SSE 三件套

```nginx
location = /api/interview/answer {
    proxy_pass http://app_backend;
    proxy_http_version 1.1;

    # --- SSE 三件套 ---
    proxy_buffering off;            # 不缓冲, 后端吐一个字节就转发一个字节
    proxy_cache off;                # 不缓存
    proxy_set_header Connection ""; # 清掉 Connection, 保持长连接不被降级

    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

逐条讲:

| 配置 | 不写会怎样 |
| --- | --- |
| `proxy_buffering off` | 流式退化成一次性返回 (第 5.1 节) |
| `proxy_cache off` | 如果上游配了缓存, SSE 响应可能被缓存, 第二个用户拿到第一个用户的面试内容 —— **这是数据泄露, 不只是 bug** |
| `proxy_set_header Connection ""` | Nginx 默认给上游发 `Connection: close`, 长连接被降级成短连接 |
| `proxy_http_version 1.1` | 默认是 1.0, **1.0 不支持长连接**, SSE 无从谈起 |
| `proxy_read_timeout 300s` | 默认 60s。一次判分要等 LLM 十几秒 (第 10 章实测本地模型 5.28 秒/次, 云端也有几秒), 追问链路叠起来能超 60s → **在评分快出来时被掐断** |

**`location = /api/interview/answer` 里的 `=` 是精确匹配, 优先级高于末尾那个 `location /`。**这样 SSE 用宽松超时和无缓冲, 其他接口用正常配置 (120s 超时 + 默认缓冲)。**不要给所有接口都关缓冲** —— 那会让静态文件和普通 JSON 也失去优化。

### 5.3 不要给 `text/event-stream` 开 gzip

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
# 注意: 不要给 text/event-stream 开 gzip
```

**gzip 有自己的缓冲区。**压缩算法要攒够一定数据才能输出一块 —— 于是你关掉了 `proxy_buffering`, 又被 gzip 的缓冲把流式压没了。

**这是一个特别隐蔽的组合坑**: 你已经关掉了 buffering, 结论是"配置对了", 但现象依旧。然后你会怀疑后端、怀疑浏览器、怀疑一切 —— 而问题在一行看起来完全无关的 `gzip_types`。

`gzip_types` 里默认不含 `text/event-stream`, 所以**只要你别手动加上去**就是安全的。这条写在这里是为了防止有人"顺手优化一下"。

### 5.4 其余配置

```nginx
upstream app_backend {
    server app:8000;
    keepalive 32;
}

server {
    listen 80;
    server_name _;

    client_max_body_size 256k;
    server_tokens off;

    location /healthz {
        proxy_pass http://app_backend;
        access_log off;
    }
}
```

- **`server app:8000`** —— `app` 是 compose 里的服务名, 由 compose 的内部 DNS 解析。**不要写 `127.0.0.1`** —— 在 nginx 容器里那指向 nginx 自己。
- **`keepalive 32`** —— 保持 32 个到上游的长连接, 省掉每次请求的 TCP 握手。
- **`client_max_body_size 256k`** —— 面试答案不会很大, 限死防滥用。默认 1MB, 但没有理由给这么多。
- **`server_tokens off`** —— 不在响应头和错误页里暴露 Nginx 版本号。**版本号是攻击者的第一手信息** (知道版本就知道有哪些已知漏洞)。
- **`location /healthz` 加 `access_log off`** —— healthcheck 每 30 秒一次, 不关掉的话访问日志会被它淹没, **真正的用户请求埋在里面找不到。**

**注意这里没配 HTTPS。**单机 demo 用 HTTP 可以, 但真要对外提供服务, 证书是必须的 —— 最省事的做法是套一层 Caddy (自动申请 Let's Encrypt 证书) 或者用云厂商的负载均衡终结 TLS。**SSE 走 HTTP 时, 中间任何一层代理都可能缓冲你的流。**

---

## 六、上线前必须跑一遍的冒烟脚本

镜像构建成功、容器 healthy, 都不等于**能用**。healthcheck 只查了 `/healthz`, 而一次面试要走 start → answer(SSE) → report 三个接口加落库。

`scripts/smoke_api.py` 就是这条全链路的一次性验证:

```python
"""接口冒烟测试: 不开浏览器, 用 httpx 把 start -> answer(SSE) -> report 全链路跑一遍。

写这个脚本的理由: 前端出问题的时候要能立刻分清是接口坏了还是页面坏了。
接口这一层跑通了, 剩下的问题一定在 JS 里。
"""
```

**这段 docstring 是整个脚本存在的理由。**上线之后用户报"页面卡住了", 你有两个嫌疑人: 接口和前端。有了这个脚本, 三十秒就能排除一个。

### 6.1 关键设计: 三档答案轮换

```python
answers = [
    "跳转时先走 A.onPause, 然后 B.onCreate onStart onResume, A.onStop 在 B.onResume 之后。"
    "返回时 B.onPause -> A.onRestart onStart onResume -> B.onStop onDestroy。",   # 好答案
    "大概知道一点, 好像和生命周期有关系。",                                          # 含糊
    "",                                                                        # 白卷
]
ans = answers[i % len(answers)]
```

**为什么是三档而不是一直发好答案?** 因为要把第 7 章 FSM 的三条分支都走到:

| 答案档位 | 触发的 FSM 分支 |
| --- | --- |
| 好答案 | `followup` —— 答得好才追问 |
| 含糊 | `hint` 或换题 —— 中间档 |
| 白卷 | `blank` 分支 + 直接换题 |

**只发好答案的冒烟测试, 只能证明 happy path 是通的。**而线上出问题的从来是分支。

### 6.2 断言 content-type

```python
with c.stream("POST", "/api/interview/answer", json={...}) as resp:
    resp.raise_for_status()
    assert "text/event-stream" in resp.headers["content-type"]
    body = "".join(resp.iter_text())
```

这个 `assert` 是为第 5 节那个坑准备的。**如果代理配错把 SSE 变成了普通响应, content-type 会变** —— 脚本立刻失败, 而不是等用户来告诉你。

但要诚实地说清楚它测不到什么: **`"".join(resp.iter_text())` 是把整个流收完再解析, 所以它验证不了"事件是不是逐个到达的"。**要验证真正的流式性, 得记录每个 chunk 的到达时间戳。这个脚本能抓住"content-type 错了"和"事件内容错了", 抓不住"事件都对但一次性到达"。

**知道一个测试测不到什么, 和知道它测到什么一样重要** —— 第 8 章讲 `parse_sse` 时说过同一句话。

### 6.3 ✅ Checkpoint 2: 全链路冒烟

先起服务 (离线模式 + 一个已知的 API Key):

```bash
OFFLINE=true API_KEYS=smoke-key DB_PATH=/tmp/ck11_db/app.db CHROMA_DIR=/tmp/ck_chroma \
  python -m uvicorn app.main:app --host 127.0.0.1 --port 8099
```

```bash
python scripts/smoke_api.py --base http://127.0.0.1:8099 --key smoke-key --rounds 12
```

真实输出 (中间四题略):

```
健康检查: {'status': 'ok', 'chunks': 25, 'model': 'deepseek-chat', 'rerank': False, 'db': {'sessions': 2, 'qa_records': 12, 'avg_score': 0.0}}
会话 14675771c522

[第 1/6 题] 四大组件/Activity生命周期 (难度 2)
  面试官: 从 Activity A 跳到 B, 再按返回键回到 A, 两个 Activity 的生命周期回调顺序分别是什么?

你: 跳转时先走 A.onPause, 然后 B.onCreate onStart o...
  -> 7.0/10 solid (0 ms) [规则判分] 命中 2/3 个要点
  追问(第 1 层): 如果 B 是 Dialog 主题的半透明 Activity, A 还会走 onStop 吗?

你: 大概知道一点, 好像和生命周期有关系。
  -> 0.0/10 blank (0 ms) [规则判分] 追问轮只记深度, 不计分

[第 2/6 题] Kotlin/协程 (难度 2)
  面试官: 为什么说协程比线程省资源? 它省在哪?

你: (交白卷)
  -> 0.0/10 blank (0 ms) 没有作答
```

```
[报告] 题数 6 均分 1.17 水平 待加强
    Kotlin/协程                     0.00 blank
    Jetpack/ViewModel             0.00 blank
    性能优化/ANR                      0.00 blank
    四大组件/Fragment生命周期             0.00 blank
    四大组件/Activity生命周期             7.00 solid
    建议: [Kotlin/协程] 补: 协程是线程之上的调度单位, 一个线程可承载大量协程
    建议: [Kotlin/协程] 补: 挂起时把线程让出去, 而不是占着线程等待
    建议: [Jetpack/ViewModel] 补: 横竖屏切换属于配置变更, 会销毁重建宿主

历史会话: [{'session_id': '14675771c522', ..., 'question_count': 6, 'avg_score': 1.17, 'level': '待加强'}, ...]
错题本: {'topics': ['Kotlin/协程', '四大组件/Activity生命周期', 'Jetpack/ViewModel', '四大组件/Fragment生命周期', '性能优化/ANR', '性能优化/内存泄漏', '四大组件/Service']}
落库报告读回: 200 待加强
问答模式命中: [('perf-memory-leak#0-0', 'vec#3,bm25#1'), ('perf-anr#2-2', 'vec#1,bm25#4')]
```

**这一次输出里要确认六件事**, 每一件对应一个可能坏掉的模块:

| 看到什么 | 说明什么活着 |
| --- | --- |
| `chunks: 25` | 知识库灌进去了 (第 4 章) |
| `7.0/10 solid` 且 `命中 2/3` | 检索选对了题 + 判分器工作 (第 5、7 章) |
| 好答案后出现 `追问(第 1 层)` | FSM 的 followup 分支活着 (第 7 章) |
| `[报告] 题数 6 均分 1.17` | 报告聚合正确 —— **算一下: 7.0 / 6 题 = 1.17, 对得上** |
| `落库报告读回: 200 待加强` | **落库真的成功了** —— 用另一条路径 (GET /report) 读回来验证, 而不是相信 POST 的返回值 |
| `问答模式命中: [(...)]` | 问答模式和混合检索的解释字段都在 (`vec#3,bm25#1` 是 RRF 的两路来源) |

### 6.4 两个真实踩到的坑

**坑一: 不带 `--key` 直接跑, 拿到 401。**

第一次跑我忘了传 key:

```
httpx.HTTPStatusError: Client error '401 Unauthorized' for url 'http://127.0.0.1:8099/api/interview/start'
```

注意 `/healthz` 是通的 (那一行输出正常), 只有业务接口 401。**这恰好证明了第 8 章的鉴权设计是对的: 健康检查必须免鉴权** (否则编排系统探测不了), 业务接口必须鉴权。

**这个 401 是好消息。**如果它返回了 200, 说明 `API_KEYS` 没生效, 那才是要紧的问题。

**坑二: 用错虚拟环境, `ModuleNotFoundError: No module named 'chromadb'`。**

```
File "/private/tmp/ref/ai/app/rag/index.py", line 39, in __init__
    import chromadb
ModuleNotFoundError: No module named 'chromadb'
ERROR:    Application startup failed. Exiting.
```

**这个报错正是本章第 1.1 节那张表里的第二行, 活生生地发生了一次。**而这就是 Docker 要解决的问题: 容器里的解释器和依赖是构建时钉死的, 不存在"用错了哪个 venv"。

顺带注意 `import chromadb` 出现在 `KnowledgeIndex.__init__` 里面而不是文件顶部 —— **这是第 4 章故意做的延迟导入**: `import app.rag.index` 这个动作本身不需要装 chromadb, 只有真的去构造一个索引对象时才需要。所以纯离线的测试和工具脚本 (它们只 import 类型定义) 在没装 chromadb 的机器上照样能跑。而这次报错是因为服务启动时要报知识库块数, 真的构造了索引。

---

## 七、真正上线: 从本机到一台云服务器

### 7.1 最小可行方案

**一台 2 核 4G 的云服务器就够。**不需要 GPU (embedding 模型是 CPU 跑的, 103MB), 不需要大内存 (compose 里限了 2G)。

```bash
# 服务器上
git clone <你的仓库> && cd <项目>

# 写 .env —— 注意这是唯一一份不进 git 也不进镜像的文件
cat > .env <<'EOF'
LLM_API_KEY=sk-xxxx
LLM_BASE_URL=https://api.deepseek.com/v1
LLM_MODEL=deepseek-chat
API_KEYS=给用户发的key1,给用户发的key2
DAILY_LIMIT=100
EOF
chmod 600 .env

docker compose -f docker/docker-compose.yml config --quiet   # 先校验
docker compose -f docker/docker-compose.yml up -d --build

# 等 healthy (首启动要灌库, 给足时间)
docker compose -f docker/docker-compose.yml ps

# 冒烟
python scripts/smoke_api.py --base http://127.0.0.1:8080 --key 给用户发的key1
```

**`chmod 600 .env` 不是形式主义。**默认权限 644 意味着服务器上任何用户都能读你的 API Key。

### 7.2 上线清单

上线前逐项过一遍。**每一项都对应第 8 章或本章里的一个具体机制, 不是泛泛的最佳实践:**

| 项 | 怎么确认 | 不做的后果 |
| --- | --- | --- |
| **鉴权已开** | `curl http://.../api/interview/start` 不带 key, **必须返回 401** | 接口对全网开放。第 8 章那句 `[auth] 警告: 没有配置 API_KEYS, 接口对所有人开放` 就是为这一刻准备的 |
| **限流已开** | `DAILY_LIMIT` 已设, 且用一个 key 打满能看到 429 | 一个人能刷爆你的 LLM 账单 |
| **`.env` 权限** | `ls -l .env` 是 `-rw-------` | 同机其他用户可读密钥 |
| **`.env` 不在镜像里** | `docker run --rm <镜像> ls -a /app \| grep -c '\.env'` 应为 0 | **推镜像等于推密钥** (第 2.9 节) |
| **数据卷已挂** | `docker volume ls \| grep itv-data` | 容器重建数据全丢 |
| **健康检查转 healthy** | `docker compose ps` 显示 `(healthy)` | Nginx 永远不启动 (第 4.5 节) |
| **SSE 过代理正常** | 浏览器打开 8080 做一次面试, **看评分是不是先出现** | 流式白做 (第 5 节) |
| **nginx 配置语法** | `docker compose exec nginx nginx -t` | —— **我没验证过这一步, 请你自己跑** |
| **冒烟全链路通** | `scripts/smoke_api.py` 跑到 `落库报告读回: 200` | 只知道进程活着, 不知道能不能用 |
| **备份** | `docker run --rm -v itv-data:/d -v $PWD:/b alpine tar czf /b/backup.tgz /d` | SQLite 里的面试记录和错题本是唯一副本 |

### 7.3 更新一次版本

```bash
git pull
docker compose -f docker/docker-compose.yml up -d --build
```

因为 entrypoint 是幂等的 (第 3.2 节), 更新时**不会重新灌库**, volume 里的数据原样保留。

**但要注意一件事: 语料改了的时候, 幂等反而成了障碍。**entrypoint 只看"块数是不是 0", 不看"语料有没有变"。所以改了 `corpus/` 之后要手动重灌:

```bash
docker compose exec app python -m app.rag.index /app/corpus
```

`python -m app.rag.index` 走的是 `KnowledgeIndex.rebuild()`, 它**每次都是全量重建** (先 `delete_collection` 再重新算向量), 所以直接跑就行, 不需要额外的清空参数。第 4 章选全量而不是增量的理由是"语料一改就重跑, 不用操心增量的边界情况" —— **代价就是这里必须手动触发。**

**这是幂等设计的代价, 值得写在 README 里** —— 否则你会遇到"我明明改了语料, 检索结果一点没变"。

### 7.4 想上 K8s 的话

compose 文件几乎可以逐行翻译:

| compose | K8s |
| --- | --- |
| `services.app` | Deployment |
| `healthcheck` | `livenessProbe` + `readinessProbe` |
| `volumes: itv-data:/data` | PersistentVolumeClaim |
| `expose` + nginx service | Service + Ingress |
| `env_file` | ConfigMap (非敏感) + Secret (密钥) |
| `deploy.resources.limits` | `resources.limits` |

**但有一个前提必须先解决: `--workers 1` 那个问题在 K8s 里会变成"多副本"问题。**多个 Pod 时会话依然在各自的进程内存里, 而 Ingress 会把请求轮询到不同 Pod。

**要么开 session affinity (粘性会话, 简单但扩容不均), 要么把会话搬到 Redis (正确解法)。**在做完这件事之前, replicas 只能是 1 —— 那样上 K8s 的收益仅剩自动重启, compose 的 `restart: unless-stopped` 已经给了。

**不要因为"上了 K8s 更专业"而上 K8s。**单机 compose 能撑到你有真实流量为止, 而那时候你要解决的第一个问题也不是编排, 是把会话搬出进程内存。

---

## 八、踩坑与专家提示

**1. `proxy_buffering off` 是本章最贵的一行。**
不写它, SSE 退化成"面试结束时一次性刷出来"。**而且直连 8000 完全正常, 只有过代理才复现** —— 于是你会去查前端, 而前端一个字都没错。

**2. 给 `text/event-stream` 开 gzip = 前一条白做。**
gzip 有自己的缓冲区。你关掉了 proxy_buffering 却现象依旧时, 去看 `gzip_types`。

**3. `proxy_http_version 1.1` 漏了, SSE 从根上不成立。**
默认 1.0 不支持长连接。

**4. `proxy_read_timeout` 默认 60s 不够。**
一次判分等 LLM 十几秒, 加追问能超 60s。**掐断的时机往往正是评分要出来的时候。**

**5. `expose` 而不是 `ports`, 逼自己走用户的路。**
留着 `ports: ["8000:8000"]` 的话, 你所有测试都在直连下做, 而用户从代理进来。**测试环境和用户环境的差异, 就是这样一行行攒出来的。**

**6. `COPY . .` 放在装依赖之前 = 每改一行代码重装一次 torch。**
Dockerfile 指令按变化频率从低到高排。`COPY pyproject.toml` 单独一层, 五分钟变十秒。

**7. `COPY uv.lock*` 那个星号是故意的。**
写成 `uv.lock` 时, 没有 lock 文件的人第一次构建就是 `file not found`。**通配符匹配为空时 `COPY` 不报错。**

**8. 前一层写进去的文件, 后一层 `rm` 掉也还占体积。**
所以清缓存要靠多阶段构建, 不能靠 `rm -rf`。

**9. `.dockerignore` 里必须有 `.env`。**
**镜像的每一层都能被任何拿到镜像的人解包读取。**在后续层 `rm` 掉密钥没用, 它还在前一层里。密钥永远运行时注入。

**10. embedding 模型要在构建期下载。**
不然第一个真实用户等 90 秒, 而且很可能网络失败 500。**"第一个用户替你测试"是完全可以避免的。**

**11. `HF_HUB_OFFLINE=1` 把"不联网"从承诺变成约束。**
不加的话每次加载都会尝试连 HF —— 内网部署每次启动等超时, 更糟的是**某天悄悄换了模型版本, 检索结果跟着变。**

**12. healthcheck 只看 HTTP 200 会漏掉 degraded。**
`/healthz` 在知识库空的时候依然返回 200 (它是健康报告不是断言)。**必须 `grep '"status":"ok"'`。**

**13. 忘装 curl → healthcheck 永远 `starting` → Nginx 永远不启动。**
这两处相距很远 (Dockerfile 的 apt 行 和 compose 的 `depends_on`), 排查时容易只看一边。

**14. `start_period` 太短 = 无限重启循环, 而日志里看不出错。**
首启动要灌库 + 加载模型。healthcheck 在灌库中就开始判 → unhealthy → 重启 → 又在灌库。**日志里每次都显示"正在灌库", 一切看起来正常。**

**15. compose 文件里 `name:` 不写, 拿目录名当项目名。**
文件在 `docker/` 下, 所以所有 volume 都叫 `docker_xxx`, **两个项目撞名, 数据互相污染。**

**16. `env_file` 用 `required: false`。**
`.env` 是 gitignore 掉的, 每个新 clone 的人第一次 `up` 都会撞 `env file not found`。配上 `environment:` 兜底值, 没 `.env` 也能起 (离线模式)。

**17. `entrypoint.sh` 必须幂等。**
不判断就灌库的话, 每次重启重算向量, **而且 chunk_id 可能变化 —— 用户错题本点进去是空的。**

**18. 幂等的代价: 改了语料不会自动重灌。**
entrypoint 只看块数是否为 0。改语料后要手动 `docker compose exec app python -m app.rag.index /app/corpus`。**症状是"我明明改了语料, 检索结果一点没变"。**

**19. `exec "$@"` 少了就是 `kill -9`。**
不用 exec 时 PID 1 是 sh, 而 sh 收到 SIGTERM 不转发给子进程 → 10 秒后被 SIGKILL → **SQLite 事务残留、进行中的面试消失。**

**20. entrypoint 里的 `2>/dev/null || echo 0` 要有。**
拿不到块数时 `COUNT` 是空串, `[ "" -eq 0 ]` 报语法错, 配上 `set -e` 就是容器直接退出。**兜底成 0 最坏情况是多花二十秒。**

**21. `--workers 1` 不是偷懒, 是会话在进程内存里的必然。**
多 worker 时约一半请求报"会话不存在", **而且是随机的**。要扩容先把会话搬 Redis, 顺序反了就是这个 bug。

**22. `VOLUME` 漏挂一个比全不挂更难查。**
把 Chroma 和 SQLite 都放 `/data` 下, 只挂一个卷。分开放两处时**漏挂一个的症状是"数据只丢了一半"**。

**23. 非 root 运行时每个 `COPY` 都要 `--chown`。**
不带的话属主是 root。**麻烦的是大部分文件只读也能跑, 于是你直到需要写文件那天才发现。**

**24. `USER app` 必须在所有 `RUN` 之后。**
放前面后续 `apt-get` 会没权限。

**25. `server app:8000` 不能写 `127.0.0.1`。**
在 nginx 容器里 `127.0.0.1` 指向 nginx 自己。用 compose 的服务名, 由内部 DNS 解析。

**26. `location /healthz` 记得 `access_log off`。**
每 30 秒一次探测, 不关的话**真实用户请求埋在探测日志里找不到。**

**27. `chmod 600 .env`。**
默认 644, 服务器上任何用户都能读你的 API Key。

**28. 冒烟测试要发三档答案, 不能只发好答案。**
只发好答案只证明 happy path 通。**线上出问题的从来是分支** (白卷、含糊、追问)。

**29. 冒烟脚本验证落库要用另一条路径读回。**
POST 返回成功不代表写进去了。`GET /report` 拿到 200 才算。

**30. 冒烟脚本测不到"真流式"。**
`"".join(resp.iter_text())` 是收完再解析, 验证不了事件逐个到达。要测这个得记 chunk 到达时间戳。**知道测试测不到什么, 和知道它测到什么一样重要。**

**31. 不带 key 拿到 401 是好消息。**
如果返回 200, 说明 `API_KEYS` 没生效 —— 那才是要紧的问题。**注意 `/healthz` 必须免鉴权**, 否则编排系统探测不了。

**32. `docker compose config` 应该是上线第一步。**
免费、不需要 daemon、一秒返回, 能抓住绝大多数"变量没定义"和"路径写错"。第 12 章会把它放进 CI。

**33. 不要因为"更专业"上 K8s。**
多副本会把 `--workers 1` 的问题原样放大成多 Pod 问题。**在会话搬出进程内存之前, replicas 只能是 1, 而那时 K8s 的收益仅剩自动重启 —— compose 的 `restart: unless-stopped` 已经给了。**

---

## 九、面试视角

**Q1: 多阶段构建解决什么问题? 不用它行不行?**

我们的项目里 `sentence-transformers` 拖进 torch, 装完之后 pip 缓存加编译产物有 1GB 多, 而这些运行时一个字节都不用。

不用多阶段的话, 你只能在同一层里 `rm -rf` 缓存 —— **但 Docker 的分层机制让这个删除白费力气: 前一层写进去的文件, 后一层删掉之后依然占镜像体积。**多阶段构建是唯一干净的解法: builder 段随便造, runtime 段从零开始只拷 `/opt/venv` 和模型权重。

**Q2: 怎么让 Docker 构建变快?**

核心是理解缓存粒度: **一层的缓存失效, 它后面所有层的缓存全部失效。**

所以指令要按变化频率从低到高排。我们的关键是把 `COPY pyproject.toml uv.lock* ./` 单独成一层, 放在装依赖之前, 业务代码的 `COPY` 放在最后。**改一行代码从"重装 torch 五分钟"变成"十秒"。**

另外用了 BuildKit 的 `--mount=type=cache` 缓存模型下载 —— 它让重复构建不用重新下 103MB, **同时这个缓存不进镜像层, 不占体积。**

**Q3: SSE 在生产环境有什么坑?**

一个坑, 但很贵: **Nginx 默认的 `proxy_buffering` 会把流式响应攒起来一次性转发。**我们精心设计的"评分先到、下一题随后到"退化成"面试结束时全部到达"。

**难查的原因是复现条件很窄**: 直连 uvicorn 正常, pytest 的 TestClient 正常, 冒烟脚本打 8000 也正常 —— **只有过 Nginx 才复现。**所以测试全绿, 只有真实用户遇到, 而症状 ("界面卡着不动") 会让人第一反应查前端。

配置是三件套: `proxy_buffering off`、`proxy_cache off`、`proxy_set_header Connection ""`, 加上 `proxy_http_version 1.1` 和放宽的 `proxy_read_timeout 300s`。**还有一个隐蔽的组合坑: 别给 `text/event-stream` 开 gzip, 压缩自带缓冲会把你刚关掉 buffering 的努力抵消掉。**

我在 compose 里故意用 `expose` 而不是 `ports`, 就是为了逼自己也走代理这条路, 和用户环境一致。

**Q4: 容器里的数据怎么持久化? 你怎么设计的?**

两样东西必须活下来: Chroma 向量库和 SQLite 业务库。我把它们都放在 `/data` 下, 挂一个 volume。

**放在同一个目录是有意的**: 分开放两处的话就要挂两个卷, 而**漏挂一个的症状是"数据只丢了一半"** —— 比全丢难查得多。

配合的一点: `CHROMA_DIR` / `DB_PATH` 从第 1 章起就是从配置读的, 所以改成容器里的绝对路径是**零代码改动**。

**Q5: `HEALTHCHECK` 应该检查什么?**

不能只检查"进程活着"。我们的 `/healthz` 返回知识库块数, 库空时 `status` 是 `degraded` —— 所以 healthcheck 是 `curl ... | grep -q '"status":"ok"'`, 检查的是**进程活着 + 知识库不空**, 也就是"能用了"。

**关键细节: 不能只看 HTTP 200。**`/healthz` 在 degraded 时依然返回 200 (它是健康报告不是断言), 只看状态码会把"服务起了但选不出题"判成健康。

还有两个配套的坑: **curl 必须装** (不装的话 healthcheck 永远 `starting`, 而 compose 里 Nginx 的 `depends_on: service_healthy` 会永远等), 以及 **`start_period` 要给足** —— 首启动要灌库加载模型, 给太短就是无限重启循环, 而日志里看不出任何错误。

**Q6: 为什么你的服务只跑一个 worker? 怎么扩容?**

因为会话状态存在进程内存里。多 worker 时每个进程有自己的一份字典 —— 用户 start 落在 worker A, answer 路由到 worker B, **B 不认识这个 session_id。**症状是随机失败: 两个 worker 约一半请求挂。

扩容的顺序必须是: **先把会话搬到 Redis, 再加 worker。**反了就是上面那个 bug。

而且在此之前不该扩: 我们的瓶颈是 LLM 调用 (本地模型实测单次判分 5.28 秒), 那是 IO 等待, asyncio 单进程能并发扛几十个会话。**在真的量到 CPU 打满之前, 单 worker 够用。**

同样的道理适用于 K8s: 多副本会把这个问题原样放大, 所以在会话搬出进程内存之前 replicas 只能是 1 —— 那时候 K8s 的收益仅剩自动重启, 而 compose 的 `restart: unless-stopped` 已经给了。

**Q7: 上线前你会检查什么?**

我有一份十项清单, 每项对应一个具体机制而不是泛泛的最佳实践。最关键的四条:

1. **不带 API Key 请求业务接口必须返回 401。**返回 200 说明鉴权没生效, 接口对全网开放。
2. **`.env` 不在镜像里。**`docker run --rm <镜像> ls -a /app` 确认。镜像每一层都能被解包读取, 推镜像等于推密钥。
3. **浏览器过代理做一次真面试, 看评分是不是先出现。**这是唯一能验证 SSE 没被缓冲的方法。
4. **冒烟脚本跑到 `落库报告读回: 200`。**注意是**用另一条路径 (GET /report) 读回来**验证落库, 而不是相信 POST 的返回值。

清单之外还有一条我会诚实说明的: nginx 配置我在写作环境里没能跑 `nginx -t` 验证 (没有 nginx 二进制), 所以那一项写的是"请自己验"。**上线清单里最危险的项, 是你以为验过其实没验的那一项。**

---

## 十、划重点

1. **Docker 的价值不是"环境一致"这句空话, 是把五件具体的事 (Python 版本、torch 装不上、模型下载、路径、忘了灌库) 从你的记忆里搬进文件。**
2. **多阶段构建是清缓存的唯一干净解法。**前一层写进去的文件, 后一层 `rm` 掉依然占体积。
3. **Dockerfile 指令按变化频率从低到高排。**`COPY pyproject.toml` 单独一层 = 改代码从五分钟变十秒。
4. **`COPY uv.lock*` 的星号是故意的** —— 文件不存在时通配符匹配为空, `COPY` 不报错。
5. **embedding 模型在构建期下载**, 否则第一个真实用户等 90 秒还可能 500。
6. **`--mount=type=cache` 让重复构建不重新下模型, 且缓存不进镜像层。**
7. **`HF_HUB_OFFLINE=1` 把"不联网"从承诺变成约束。**不加会每次尝试连 HF, 甚至悄悄换模型版本。
8. **`.dockerignore` 必须有 `.env`。**镜像每一层都能被解包读取, 后续层 `rm` 无用。**密钥永远运行时注入。**
9. **非 root 运行 + 显式 UID + 每个 `COPY` 带 `--chown` + `USER` 放最后。**容器不是安全边界, 逃逸时 root 和普通用户差的是整台机器和一个目录。
10. **Chroma 和 SQLite 都放 `/data`, 只挂一个卷。**分开放时漏挂一个的症状是"数据只丢了一半"。
11. **healthcheck 要 `grep '"status":"ok"'` 而不是只看 200** —— `/healthz` 是健康报告不是断言, degraded 时也返回 200。
12. **curl 必须装, 否则 healthcheck 永远 `starting`, Nginx 永远不启动。**
13. **`start_period` 太短 = 无限重启循环, 而日志里每次都显示"正在灌库", 看起来一切正常。**
14. **`--workers 1` 是会话存在进程内存里的必然结果。**扩容顺序: 先搬 Redis, 再加 worker。
15. **`entrypoint.sh` 必须幂等** —— 重复灌库不只是慢, 还可能让 chunk_id 变化, 用户错题本点进去是空的。
16. **幂等的代价: 改了语料不会自动重灌**, 要手动跑一次 `python -m app.rag.index`。症状是"改了语料检索结果没变"。
17. **`exec "$@"` 少了就是 `kill -9`。**sh 收到 SIGTERM 不转发给子进程, 10 秒后被硬杀, SQLite 事务残留。
18. **compose 的 `name:` 必须显式写**, 否则拿目录名 (`docker`) 当项目名, 两个项目 volume 撞名。
19. **`env_file` 用 `required: false` + `environment` 兜底**, 让没建 `.env` 的人也能起 (离线模式)。
20. **`expose` 而不是 `ports`, 逼自己走用户的路。**测试环境和用户环境的差异就是这样一行行攒出来的。
21. **`depends_on: condition: service_healthy` 消除重启时的 502**, 但它依赖 healthcheck 真能转 healthy。
22. **`proxy_buffering off` 是本章最贵的一行。**不写它 SSE 就退化成一次性返回, **而且只有过代理才复现。**
23. **SSE 三件套**: `proxy_buffering off` + `proxy_cache off` + `Connection ""`, 加 `proxy_http_version 1.1` 和 `proxy_read_timeout 300s`。
24. **不要给 `text/event-stream` 开 gzip** —— 压缩自带缓冲, 会把你关 buffering 的努力抵消掉。
25. **`location = /path` 精确匹配优先级高于 `location /`**, 所以 SSE 用特殊配置、其他接口用常规配置。
26. **`server app:8000` 用服务名, 不能写 `127.0.0.1`** (那指向 nginx 自己)。
27. **`location /healthz` 加 `access_log off`**, 否则真实请求埋在 30 秒一次的探测日志里。
28. **冒烟测试发三档答案 (好/含糊/白卷)**, 把 FSM 的 followup、hint、blank 三条分支都走到。**只发好答案只证明 happy path 通, 而线上出问题的从来是分支。**
29. **验证落库要用另一条路径读回来** (`GET /report` 拿 200), 不能相信 POST 的返回值。
30. **冒烟脚本测不到"真流式"** —— `"".join(iter_text())` 是收完再解析。知道测试测不到什么, 和知道它测到什么一样重要。
31. **不带 key 拿到 401 是好消息;返回 200 才是问题。**`/healthz` 必须免鉴权, 业务接口必须鉴权。
32. **`docker compose config` 是上线第一步** —— 免费、不要 daemon、一秒返回, 能抓住变量没定义和路径写错。实测 `exit=0`, 并确认了 `context` 解析成绝对路径、`OFFLINE` 兜底值生效、`start_period` 被 compose 覆盖成 `1m30s`。
33. **`chmod 600 .env`。**默认 644 意味着服务器上任何用户都能读你的 API Key。
34. **不要因为"更专业"上 K8s。**多副本会原样放大 `--workers 1` 的问题;在会话搬出进程内存之前 replicas 只能是 1, 而那时 K8s 只多给了自动重启。
35. **本章有两处我没能验证: nginx 配置语法和镜像构建全过程** (环境里 Docker daemon 没跑)。**上线清单里最危险的项, 是你以为验过其实没验的那一项。**

---

## 十一、下一章预告

现在项目能一条命令拉起来了。但还有一个问题: **上面这三十几条注意事项, 靠谁来记?**

第 8 章那 15 个 API 测试、第 9 章那套评估指标、`docker compose config` 这一步校验 —— 今天你会跑, 三个月后改一行代码时你不会跑。**而不跑的检查等于不存在的检查。**

**第 12 章讲 CI 建设与质量闸门**, 把这些检查变成"不通过就不许合并":

- **GitHub Actions 的基本结构**, 以及为什么要分成几个并行的 job
- **离线测试策略**: 为什么 CI 里必须 `OFFLINE=true` —— 32 个测试要在没有 API Key 的环境里全绿, 而且不花一分钱
- **哪些评估层能进 CI, 哪些不能** (第 9 章已经给出答案: 检索层和判分层可以, 对话层不行 —— 它花钱、慢、还有抖动)
- **质量闸门的阈值怎么定**: `Recall@3` 掉了不许合并、判分器 `MAE > 2.5` 不许合并, 而阈值本身要留多少余量
- **`ruff` + 类型检查**: 便宜的检查放在最前面, 快速失败
- **镜像构建验证**: CI 里 build 一次但不推送, 确认 Dockerfile 没坏
- **`docker compose config` 进 CI** —— 本章 Checkpoint 1 那一步的自动化

**第 12 章结束时, 你的仓库会拒绝合并任何让指标变差的 PR** —— 而这件事写在简历上, 比"我做过 RAG"有分量得多。










