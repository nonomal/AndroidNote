# 08 · Docker 与部署实战

> 前面七篇里,项目都跑在你自己的电脑上("开发环境")。本篇解决最后一公里:**打包成 Docker 镜像 → 用 docker compose 一键编排 → Nginx 反向代理 → 部署到云服务器**,让别人通过一个网址就能用上你的旅行助手。以项目一为主线示范,项目二、三方法完全相同。预计 1~2 天。

## 0. 为什么需要 Docker

把程序发给别人,经典的死亡三连:"我这边 Python 是 3.9"、"装依赖报错了"、"Windows 上跑不了"。

Docker 的解法:把**代码 + Python 解释器 + 全部依赖 + 系统库**打成一个"镜像"(image),在任何装了 Docker 的机器上以"容器"(container)方式运行,环境 100% 一致。

三个概念的关系:

```
Dockerfile(菜谱) ──build──→ 镜像(做好的预制菜) ──run──→ 容器(端上桌的一份菜)
```

- 镜像是只读模板,容器是运行中的实例,一个镜像可以起 N 个容器
- 容器之间、容器和宿主机之间默认隔离(文件、网络、进程都隔离)。**宿主机** = 运行 Docker 的那台机器,也就是你的电脑或云服务器

## 1. 安装 Docker

- **macOS / Windows**:安装 [Docker Desktop](https://www.docker.com/products/docker-desktop/),装完启动它(菜单栏/托盘出现鲸鱼图标)
- **Linux 服务器**:

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

验证:

```bash
docker --version
docker compose version     # compose 已内置在新版 docker 里
docker run hello-world     # 拉一个测试镜像跑一下
```

> 国内拉镜像慢/失败:配置镜像加速器。Docker Desktop → Settings → Docker Engine,或 Linux 编辑 `/etc/docker/daemon.json`,加入 `{"registry-mirrors": ["https://docker.m.daocloud.io"]}` 后重启 Docker。

## 2. 给后端写 Dockerfile

在 `helloagents-trip-planner/backend/` 下新建 `Dockerfile`(无扩展名):

```dockerfile
# 基础镜像:官方 Python 3.12 精简版
FROM python:3.12-slim

# 容器内的工作目录(后续命令都在这里执行)
WORKDIR /app

# 先只拷贝依赖清单再安装——利用Docker分层缓存:
# 只要requirements.txt没变,重新build时这一步直接用缓存,不用重装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple

# 再拷贝项目代码(代码天天改,放在后面,只让这一层重新构建)
COPY . .

# 声明容器监听的端口(文档作用,真正映射在运行时做)
EXPOSE 8000

# 容器启动命令。注意:不用 --reload,生产环境不需要热重载
CMD ["uvicorn", "app.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

再新建 `.dockerignore`(同目录),避免把垃圾和密钥打进镜像:

```
.venv
venv
__pycache__
*.pyc
.env
.git
```

> **`.env` 必须在 .dockerignore 里。** 镜像会被分发,密钥打进去等于公开。密钥在运行容器时用 `--env-file` 或 compose 的 `env_file` 注入。

构建并运行:

```bash
cd helloagents-trip-planner/backend
docker build -t trip-backend:v1 .            # -t 起名字, . 表示构建上下文是当前目录
docker run --rm -p 8000:8000 --env-file .env trip-backend:v1
```

参数解释:`-p 8000:8000` 把宿主机 8000 端口映射到容器 8000(左边宿主机,右边容器);`--env-file .env` 把密钥注入容器环境变量;`--rm` 容器停止后自动删除。

**Checkpoint 1:** 浏览器开 `http://localhost:8000/docs` 正常 → 后端已容器化。

> 项目一特有的坑:容器里 Agent 要用 `uvx amap-mcp-server` 启动 MCP 子进程,而 `python:3.12-slim` 里没有 uv。好在 `requirements.txt` 里带了 `uv>=0.8.0`(pip 装的 uv 自带 uvx 命令),所以能跑。若你换成项目二、三的 Dockerfile 则无此顾虑。

## 3. 前端怎么打包:构建成静态文件 + Nginx

Vue 项目开发时用 `npm run dev`,但生产环境不这么跑。执行:

```bash
cd ../frontend
npm run build     # 产出 dist/ 目录:纯静态的 HTML/JS/CSS
```

`dist/` 里是编译好的静态文件,任何 Web 服务器都能托管,业界标配是 **Nginx**。前端 `Dockerfile`(放 `frontend/` 下):

```dockerfile
# ---- 第一阶段:构建 ----
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm install --registry=https://registry.npmmirror.com
COPY . .
RUN npm run build

# ---- 第二阶段:只留产物 ----
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

这叫**多阶段构建**:第一阶段用带 Node 的大镜像编译,第二阶段只把 `dist/` 拷进 50MB 的 Nginx 小镜像。最终镜像里没有 node_modules,又小又安全。

`frontend/nginx.conf`:

```nginx
server {
    listen 80;

    # 托管前端静态文件
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;   # Vue路由刷新404的标准解法
    }

    # 反向代理:凡是 /api/ 开头的请求,转发给后端容器
    location /api/ {
        proxy_pass http://backend:8000;     # "backend"是compose里的服务名
        proxy_set_header Host $host;
        proxy_read_timeout 300s;            # Agent生成要1-2分钟,默认60s会超时断开!
    }
}
```

两个关键点:

1. **反向代理消灭了 CORS 问题**:浏览器只和 Nginx(一个源)打交道,前端请求 `/api/trip/plan`,Nginx 内部转给后端。前端代码里的 baseURL 要相应改成相对路径 `/api`(检查 `src/services/api.ts` 或前端 `.env` 的 API 地址配置)。
2. **`proxy_read_timeout 300s`**:LLM 生成慢,不调大这个,行程生成到一半 Nginx 就掐断连接,前端表现为"转圈后突然失败"。项目二的 SSE 长连接同理,还要加 `proxy_buffering off;`(否则 Nginx 会攒着事件不往外发,打字机效果失效)。

## 4. docker compose:一条命令拉起整套系统

服务多了,逐个 `docker run` 太痛苦。在项目根目录 `helloagents-trip-planner/` 新建 `docker-compose.yml`:

```yaml
services:
  backend:
    build: ./backend            # 用backend/Dockerfile构建
    env_file: ./backend/.env    # 注入密钥(此文件不进镜像、不进Git)
    restart: unless-stopped     # 崩了自动拉起
    # 注意:不写ports——后端不直接暴露给外网,只让Nginx内网访问

  frontend:
    build: ./frontend
    ports:
      - "80:80"                 # 整个系统唯一对外的端口
    depends_on:
      - backend
    restart: unless-stopped
```

启动/管理:

```bash
docker compose up -d --build   # 构建并后台启动全部服务
docker compose ps              # 看状态
docker compose logs -f backend # 追后端日志(排错主战场)
docker compose down            # 全部停止并删除容器
```

compose 会自动创建一个内部网络,**服务名就是主机名**——这就是 nginx.conf 里能写 `http://backend:8000` 的原因。容器之间用服务名互相访问,和 localhost 没有关系(容器里的 localhost 是它自己)。

**Checkpoint 2:** 浏览器开 `http://localhost`(80 端口,不带 :5173),完整走通"填表单 → 生成行程"。

## 5. 部署到云服务器

### 5.1 买机器

任选阿里云/腾讯云/华为云的入门级云服务器(2核2G 即可,新用户几十块/年),系统选 **Ubuntu 22.04/24.04**。要点:

- 记下公网 IP
- 安全组(防火墙)放行 **22**(SSH)和 **80**(HTTP)端口——控制台里配,新手最容易漏

### 5.2 上传代码并启动

```bash
# 本机:SSH登录服务器
ssh root@你的公网IP

# 服务器:装Docker(见第1节),然后拉代码
git clone https://github.com/datawhalechina/hello-agents.git
cd hello-agents/code/chapter13/helloagents-trip-planner

# 把你写好的 Dockerfile / nginx.conf / docker-compose.yml 也传上去
# (本机执行)scp docker-compose.yml root@IP:/root/hello-agents/code/chapter13/helloagents-trip-planner/

# 服务器:创建.env(密钥手动填,永远不要经过Git)
vim backend/.env

# 一键起飞
docker compose up -d --build
```

**Checkpoint 3:** 任何设备的浏览器访问 `http://你的公网IP`,旅行助手上线。发给朋友试试。

### 5.3 加固清单(可选但建议)

- **域名 + HTTPS**:域名解析到 IP,用 [Certbot](https://certbot.eff.org/) 免签证书,Nginx 加 443 配置
- **限制 CORS/来源**:后端 `.env` 的 `CORS_ORIGINS` 改成你的正式域名
- **别用 root 日常操作**:`adduser deploy && usermod -aG docker deploy`
- **费用保险丝**:LLM 平台设置每日额度上限——公网服务可能被人刷接口,烧的是你的 API 余额

## 6. 项目二、三的部署差异速记

| | 项目二(深度研究) | 项目三(赛博小镇) |
| --- | --- | --- |
| 后端依赖安装 | 用 uv:`COPY pyproject.toml uv.lock ./` + `RUN pip install uv && uv sync --frozen`,CMD 用 `uv run uvicorn main:app ...` | 与项目一相同(requirements.txt) |
| 前端 | 同项目一(Vue),但 Nginx 必须加 `proxy_buffering off;` 保 SSE | 无 Web 前端;Godot 游戏在玩家本机运行,`config.gd` 里把地址改成 `http://你的IP` 即可 |
| 数据持久化 | 笔记目录挂 volume | **记忆必须挂 volume**,否则容器一重建 NPC 全部失忆:`volumes: - ./memory_data:/app/memory_data` |

volume(数据卷)是容器持久化的标准做法:容器里的路径映射到宿主机目录,容器删了数据还在。

---

## 自测清单

1. 镜像和容器的关系?为什么 `COPY requirements.txt` 要放在 `COPY . .` 前面?
2. `.env` 为什么必须进 `.dockerignore`?那容器运行时密钥从哪来?
3. nginx.conf 里 `http://backend:8000` 的 `backend` 是什么?容器里访问 `localhost:8000` 为什么不行?
4. 用户反映"行程生成到一半就报错",而直连后端没问题——先查 Nginx 的哪个配置?
5. 项目三容器重启后 NPC 失忆,原因和解法?
6. 动手:把项目一完整部署到一台云服务器(或至少在本机用 compose 跑通 Checkpoint 2)。

下一篇:[09 · 常见问题与排错手册](./09-常见问题与排错手册.md)
