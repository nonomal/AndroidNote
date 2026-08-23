# 08 · Web 服务: FastAPI + SSE + 鉴权 + 落库

> **一句话总结**: 第 7 章的状态机只 `yield Event` 不 `print`, 这一章就是把那些 Event 变成 HTTP 响应 —— 用 SSE 让判分结果先到、下一题后到, 用 SQLite 把每一轮存下来从而让"上次答错的知识点"能在下一场面试里被优先重考, 再用 API Key 和每日配额挡住账单。

## 本章你能学到什么

- 为什么面试接口必须用 **SSE** 而不是普通 POST, 以及为什么不用 WebSocket
- SSE 报文的三行格式、末尾那个空行为什么少不了、Nginx 会怎么把它憋住
- **服务层为什么要单独存在**: FSM 不知道数据库, API 不知道 `Turn` 长什么样
- 活跃会话放内存、已完成明细落库的分工, 以及内存不能无限涨该怎么清
- 四张 SQLite 表的分工, 以及**错题本闭环**是怎么接上的
- API Key 鉴权的三个安全点: `compare_digest`、配额落库、没配 Key 时的醒目警告
- 限流为什么挂在"开一场"而不是"答一题"上
- `/healthz` 为什么要查向量库条数 —— 容器起来了不等于服务能用
- 单文件前端为什么用 `fetch + ReadableStream` 而不是 `EventSource`
- 15 个接口测试怎么做到不起真服务器、不联网、不污染真实数据库

---

## 一、核心知识点: 为什么是 SSE

### 1.1 先看问题: 一次提交产生几个事件

回忆第 7 章 `submit()` 的行为 —— 提交一次答案, 会连续产出 2 到 3 个事件:

| 情形 | 事件序列 |
| --- | --- |
| 答得半懂 | `grade` → `followup` |
| 答完换题 | `grade` → `question` |
| 最后一题 | `grade` → `report` |

其中 `grade` 是**调 LLM 判分**得来的 (2~5 秒), 而 `question` 只是从黄金集里选一道题 (毫秒级)。

如果用普通 POST 返回一个 JSON:

```json
{"events": [{"type": "grade", ...}, {"type": "question", ...}]}
```

用户点了"提交", 然后**盯着转圈 5 秒**, 什么都看不到, 直到所有事件算完一起返回。

用 SSE 就变成: 判分一出来先推给浏览器, 用户马上看到"得分 7.5, 你漏了这两条", 下一道题随后到达。**同样的总耗时, 感知上快了一倍。**

### 1.2 SSE 是什么

**Server-Sent Events**, 服务器主动往浏览器推消息的一种标准。本质就是一个**不关闭的 HTTP 响应**: `Content-Type: text/event-stream`, 服务器写一段、浏览器收一段。

一条报文长这样:

```
event: grade
data: {"score": 7.5, "depth": "solid"}

```

三条规则:

- `event:` 行是事件名 (可选, 不写就是 `message`)
- `data:` 行是数据 (我们统一放 JSON 字符串)
- **末尾必须有一个空行** (也就是连续两个 `\n`) 表示这条事件结束

**最后那个空行是最容易漏的。** 少一个 `\n`, 浏览器会一直等"这条事件还没完", 你的判分结果就永远显示不出来 —— 而服务端日志一切正常, 排查起来极其痛苦。

### 1.3 为什么不用 WebSocket

| | SSE | WebSocket |
| --- | --- | --- |
| 协议 | 纯 HTTP | 独立协议, 要 Upgrade 握手 |
| 方向 | 单向 (服务器 → 浏览器) | 双向 |
| 过 Nginx | 关掉 buffering 就行 | 要额外配 Upgrade 转发 |
| 断线重连 | 浏览器原生支持 | 自己写 |
| 服务端心跳 | 不需要 | 通常要 |

**我们的场景就是单向推送**: 请求本身还是普通 POST (提交答案), 只有响应需要分段到达。上 WebSocket 等于为了单向需求付双向的运维成本。

第 11 章配 Nginx 时你会看到, SSE 只需要一行 `proxy_buffering off;`。

### 1.4 三层结构: FSM / Service / API

这一章新增两个文件, 加上第 7 章的 FSM 是三层:

| 层 | 文件 | 知道什么 | 不知道什么 |
| --- | --- | --- | --- |
| 状态机 | `app/agent/fsm.py` | 面试怎么走 | 数据库、HTTP |
| 服务层 | `app/service.py` | FSM + 数据库怎么配合 | HTTP、状态码 |
| 接口层 | `app/api.py` | HTTP 契约、SSE 格式 | `Turn` 长什么样 |

**中间那层为什么必须存在?** 因为有两件事既不属于 FSM 也不属于 HTTP:

1. **"一个事件产生了, 顺手把它落库"** —— FSM 不该知道数据库存在 (第 7 章那 17 个测试零 IO 就靠这一条)
2. **进程内的活跃会话表** —— HTTP 是无状态的, 但一场面试是有状态的多轮对话

把这两件事塞进 API 层, 接口函数会变成 50 行的大杂烩; 塞进 FSM, 第 7 章的测试就要连数据库。

---

## 二、动手实战: 数据库层 `app/core/db.py`

### 2.1 四张表的分工

```python
SCHEMA = """
CREATE TABLE IF NOT EXISTS sessions (
    session_id   TEXT PRIMARY KEY,
    user_key     TEXT NOT NULL DEFAULT 'anon',
    started_at   TEXT NOT NULL DEFAULT (datetime('now')),
    finished_at  TEXT,
    difficulty   INTEGER NOT NULL DEFAULT 2,
    question_count INTEGER NOT NULL DEFAULT 0,
    avg_score    REAL NOT NULL DEFAULT 0.0,
    level        TEXT,
    report_json  TEXT
);
CREATE INDEX IF NOT EXISTS idx_sessions_user ON sessions(user_key, started_at DESC);
```

| 表 | 一行代表 | 用途 |
| --- | --- | --- |
| `sessions` | 一场面试 | 历史列表; `report_json` 存完整报告 |
| `qa_records` | 一问一答 (含追问轮) | 复盘; **第 10 章导出微调数据** |
| `wrong_book` | (用户, 知识点) 的累计失误 | 下次面试优先重考 |
| `usage_log` | (用户, 日期) 的调用次数 | 每日配额 |

**为什么用 SQLite 而不是 Postgres:** 单机部署、零运维、**一个文件就是整个数据库** —— 第 11 章 docker 里挂一个 volume 就完事。真有并发写入压力再换, 换的时候只改这一个文件。

**`finished_at` 可以为 NULL**, 而且是判断"这场面试完了没"的唯一依据。`/api/interview/{id}/report` 就是靠 `report_json` 是否为空返回 404 的。

**索引建在 `(user_key, started_at DESC)` 上而不是单列**: 历史列表的查询是 `WHERE user_key=? ORDER BY started_at DESC`, 复合索引一次走完; 两个单列索引只能用上一个。

### 2.2 qa_records: 为第 10 章埋的表

```python
CREATE TABLE IF NOT EXISTS qa_records (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id     TEXT NOT NULL,
    turn_index     INTEGER NOT NULL,
    followup_depth INTEGER NOT NULL DEFAULT 0,
    qid            TEXT NOT NULL,
    topic          TEXT NOT NULL,
    difficulty     INTEGER NOT NULL DEFAULT 2,
    question       TEXT NOT NULL,
    answer         TEXT NOT NULL DEFAULT '',
    score          REAL,
    depth          TEXT,
    scored         INTEGER NOT NULL DEFAULT 1,
    hit_json       TEXT,
    missed_json    TEXT,
    wrong_json     TEXT,
    comment        TEXT,
    latency_ms     INTEGER NOT NULL DEFAULT 0,
    created_at     TEXT NOT NULL DEFAULT (datetime('now'))
);
```

**这张表的字段和第 7 章的 `Turn` 几乎一一对应, 这不是巧合。** 第 10 章要把真实面试记录导出成微调样本, `question` 是 prompt、`answer` + `score` + `hit_json` 是 completion —— **现在多存三个 JSON 字段, 第 10 章就不用重新跑一遍面试来攒数据。**

**`scored INTEGER` 而不是 `BOOLEAN`**: SQLite 没有布尔类型, 存 0/1。写入时 `1 if turn.get("scored", True) else 0`, **默认 True** —— 老数据 (如果有) 按主问题算。

**三个 `_json` 后缀的字段**: SQLite 不支持数组, 三个列表字段序列化成 JSON 文本存。**加 `_json` 后缀是刻意的**, 读代码时一眼知道这一列要 `json.loads`。

**`idx_qa_topic ON qa_records(topic)`**: 为的是"按知识点统计全站正确率"这类查询。现在还没用到, 但索引成本极低, 而事后加索引要锁表。

### 2.3 连接管理: `check_same_thread=False` 加一把全局锁

```python
_LOCK = threading.Lock()


class Database:
    """极薄的一层封装。刻意不引 ORM: 表只有四张, SQL 直接写反而更好读、更好调。"""

    @contextmanager
    def conn(self) -> Iterator[sqlite3.Connection]:
        # check_same_thread=False: FastAPI 的线程池会在不同线程里用同一个 Database 实例。
        # 配合模块级 _LOCK 串行化写入 —— SQLite 的写是全库锁, 自己排队比等它抛 "database is locked" 好。
        c = sqlite3.connect(self.path, check_same_thread=False, timeout=10.0)
        c.row_factory = sqlite3.Row
        try:
            with _LOCK:
                yield c
                c.commit()
        finally:
            c.close()
```

**这 10 行踩过一个坑, 值得展开。**

FastAPI 的同步接口函数 (`def` 而不是 `async def`) 跑在线程池里。两个请求同时写库时, Python 的 `sqlite3` 默认会检查"连接是不是在创建它的线程里用", 不是就抛 `ProgrammingError`。所以要 `check_same_thread=False`。

但关掉这个检查之后, 真正的并发写就暴露了: **SQLite 的写锁是全库级别的**, 两个线程同时 `INSERT` 会有一个拿不到锁, 等 `timeout` 秒之后抛 `database is locked`。

**修法是自己排队**: 模块级 `_LOCK` 把所有数据库操作串行化。代价是吞吐上限等于单线程写库速度 (每秒几千次 INSERT), 对一个面试应用绰绰有余。**好处是永远不会看到 `database is locked` 那个错误。**

**`c.row_factory = sqlite3.Row`**: 让查询结果能按列名取 (`r["topic"]`) 而不是按下标 (`r[0]`)。加一列时下标全乱, 列名不会。

**`yield c` 在 `with _LOCK` 里面, `c.close()` 在 `finally` 里**: 锁只在事务期间持有, 连接关闭不需要锁。而 `commit()` 必须在锁内 —— 否则两个事务的提交顺序不可控。

**刻意不引 ORM**: 四张表、十几条 SQL, SQLAlchemy 带来的是"要学一套新语法才能看懂 `SELECT`"。表多起来 (二十张以上) 再说。

### 2.4 UPSERT: 错题本一条 SQL 搞定

```python
    def bump_wrong_book(self, user_key: str, topic: str, score: float) -> None:
        """低分才记。UPSERT 一条 SQL 搞定, 不用先 SELECT 再判断。"""
        with self.conn() as c:
            c.execute(
                """INSERT INTO wrong_book(user_key, topic, miss_count, last_score)
                   VALUES(?,?,1,?)
                   ON CONFLICT(user_key, topic) DO UPDATE SET
                       miss_count = miss_count + 1,
                       last_score = excluded.last_score,
                       updated_at = datetime('now')""",
                (user_key, topic, score),
            )
```

**"先 SELECT 看有没有, 有就 UPDATE 没有就 INSERT"是错的** —— 两个请求之间有竞态: 都 SELECT 到"没有", 都去 INSERT, 第二个撞主键。

`ON CONFLICT ... DO UPDATE` 是 SQLite 3.24+ 的 UPSERT 语法, **一条语句原子完成**。`excluded` 是个特殊表名, 指"本次想插入的那一行"。

`miss_count = miss_count + 1` 是累加, `last_score = excluded.last_score` 是覆盖 —— 累计错几次 + 最近一次多少分, 正好是 `weak_topics` 排序要的两个维度:

```python
    def weak_topics(self, user_key: str, limit: int = 5) -> list[str]:
        with self.conn() as c:
            rows = c.execute(
                """SELECT topic FROM wrong_book WHERE user_key=?
                   ORDER BY miss_count DESC, last_score ASC LIMIT ?""",
                (user_key, limit),
            ).fetchall()
        return [r["topic"] for r in rows]
```

**排序是"错得最多的优先, 同样次数则分最低的优先"。** 这个列表会被服务层塞进 `SessionState.weak_topics`, 然后被第 7 章 4.2 节的选题器第 1 档消费 —— **错题本闭环在这里接上。**

### 2.5 配额自增: 写和读必须在同一个事务里

```python
    def incr_usage(self, user_key: str, day: str) -> int:
        """自增并返回当天累计值。写和读在同一个事务里, 避免两个请求同时读到旧值。"""
        with self.conn() as c:
            c.execute(
                """INSERT INTO usage_log(user_key, day, n) VALUES(?,?,1)
                   ON CONFLICT(user_key, day) DO UPDATE SET n = n + 1""",
                (user_key, day),
            )
            row = c.execute(
                "SELECT n FROM usage_log WHERE user_key=? AND day=?", (user_key, day)
            ).fetchone()
            return int(row["n"]) if row else 0
```

**两条 SQL 在同一个 `with self.conn()` 块里, 也就是同一个事务、同一把锁。** 如果拆成两次调用 (`incr()` 然后 `get()`), 中间可能插进另一个请求的自增, 于是两个请求都读到 `n=3`, 配额 3 的限制被突破成 4 次。

**配额落在数据库而不是内存里**: 进程重启不清零。用内存计数的话, 重启一次就等于给所有人续了一天的额度 —— 而崩溃重启恰恰最容易发生在被人刷接口的时候。

**✅ Checkpoint 1**: 数据库层自测。

```bash
uv run python -m app.core.db
```

真实输出:

```
stats      = {'sessions': 1, 'qa_records': 1, 'avg_score': 4.0}
弱项      = ['Kotlin/协程']
历史      = [{'session_id': 's1', 'started_at': '2026-08-18 07:27:50', 'finished_at': '2026-08-18 07:27:50', 'question_count': 1, 'avg_score': 4.0, 'level': '初级'}]
报告      = {'question_count': 1, 'avg_score': 4.0, 'level': '初级'}
轮次数    = 1
配额自增  = [1, 2, 3]
```

**这个 `__main__` 块跑在 `tempfile.TemporaryDirectory()` 里**, 用完自动删, 不会污染 `data/app.db`。**给每个模块写一个自测块, 但自测块绝对不能碰真实数据** —— 这条规则在下面 6.1 节的 `conftest.py` 里还会再出现一次。

最后一行 `[1, 2, 3]` 就是 2.5 节那个"写读同事务"的验证: 连续调三次, 返回值严格递增。

---

## 三、服务层 `app/service.py`

### 3.1 活跃会话放内存, 完成明细落库

```python
"""面试服务层: 把 FSM、SQLite、错题本粘在一起, 给 API 层一个干净的接口。

活跃会话为什么放内存: 一场面试是有状态的多轮对话, 每次提交都从库里反序列化整个 SessionState
既慢又容易和 Pydantic 版本打架。内存里放 SessionState, 库里放已完成的明细 —— 各干各的活。
单机单进程够用; 要多进程就把 _SESSIONS 换成 Redis, 换的时候只改这一个类。
"""
```

**这个取舍要想清楚。** 三个选项:

| 方案 | 每次提交要做什么 | 问题 |
| --- | --- | --- |
| 全部落库 | 反序列化整个 `SessionState` | 慢; 而且 Pydantic 改字段就读不出老数据 |
| 全部内存 | 直接取对象 | 重启丢会话; 没有历史记录 |
| **内存放活跃, 库放明细** | 直接取对象 + 顺手写一行 | 重启丢**进行中**的会话 (可接受) |

第三种的关键判断是: **"进行中的面试"丢了不算大事** (用户重新开一场就行), 但"已完成的报告"丢了不可接受。所以进行中的放内存图快, 完成的落库图稳。

`"要多进程就把 _SESSIONS 换成 Redis, 换的时候只改这一个类"` —— **这句注释是给三个月后的自己写的。** 它说明了当前设计的边界 (单进程) 和突破边界的成本 (改一个类)。

```python
class InterviewService:
    def __init__(self, db=None, offline=None, golden="eval/golden.yaml"):
        self.db = db or get_db()
        self.offline = settings.offline if offline is None else offline
        self.index = KnowledgeIndex()
        self.retriever = Retriever(self.index, use_rerank=settings.use_rerank)
        self.selector = QuestionSelector(self.retriever, golden)
        self.fsm = InterviewFSM(selector=self.selector, grader=get_grader(offline=self.offline))
        self._sessions: dict[str, SessionState] = {}
        self._users: dict[str, str] = {}
        self._lock = threading.Lock()
```

**整条依赖链在这里一次性组装完**: index → retriever → selector → fsm。**这是全项目唯一一处知道"完整装配方式"的地方** —— 前面每一层都只接受注入, 不自己 new 依赖。

`self.offline = settings.offline if offline is None else offline` 而不是 `offline or settings.offline`: 因为 `offline=False` 是一个有意义的值, `or` 会把它当成"没传"。**布尔参数的默认值处理必须用 `is None` 判断。**

`_users` 单独一个字典存 `session_id -> user_key`: 落库时要知道这轮属于谁 (错题本是按用户存的), 但 `SessionState` 里没有用户字段 —— **FSM 不该知道"用户"这个概念。**

### 3.2 create: 错题本注入的地方

```python
    def create(self, user_key: str, difficulty: int = 2) -> tuple[str, Event]:
        st = self.fsm.start(difficulty=difficulty)
        # 把历史错题本注入这场面试: 上次没答好的知识点这次优先重考
        for topic in self.db.weak_topics(user_key):
            if topic not in st.asked_topics:
                st.weak_topics.append(topic)
        with self._lock:
            self._sessions[st.session_id] = st
            self._users[st.session_id] = user_key
        self.db.create_session(st.session_id, user_key, difficulty)
        return st.session_id, self.fsm.next_question(st)
```

**这 4 行 for 循环是整个"错题本"功能的全部实现。** 链条完整走一遍:

1. 上一场面试某个知识点得分 < 6.0 → `bump_wrong_book` 记一次 (3.4 节)
2. 这一场 `create` 时 `weak_topics(user_key)` 查出来 → 塞进 `SessionState.weak_topics`
3. 选题器 `pick()` 的第 1 档优先选这些 topic (第 7 章 4.2 节)

**三个文件各管一段, 没有一个文件知道全貌** —— 这正是分层的意思。而这一层是唯一同时看得见"数据库"和"FSM"的地方, 所以注入只能发生在这里。

`return session_id, self.fsm.next_question(st)`: **开场同步返回第一题**, 不走 SSE。因为这里只有一个事件, 用 SSE 是徒增前端复杂度。**协议选择要按事件个数来, 不要图统一。**

### 3.3 submit: 边流边落库

```python
    def submit(self, session_id: str, answer: str) -> Iterator[Event]:
        st = self._sessions.get(session_id)
        if st is None:
            yield Event("error", {"message": "会话不存在或已过期, 请重新开始"})
            return
        if st.finished:
            yield Event("error", {"message": "本场面试已结束"})
            return

        turn_before = st.current
        for ev in self.fsm.submit(st, answer):
            if ev.type == "grade" and turn_before is not None:
                self._persist_turn(session_id, st, turn_before, ev)
            if ev.type == "report":
                self.db.finish_session(session_id, ev.data)
            yield ev
```

**`turn_before = st.current` 这一行必须在循环之前取。** 因为 `fsm.submit()` 会往 `st.turns` 里追加新的追问轮或下一题, 循环开始后 `st.current` 已经变了。**要落库的是"刚判完分的那一轮", 不是"现在最新的那一轮"。**

这类 bug 的症状是: 数据库里的 `question` 和 `answer` 错位一行, 而且只在触发追问时才错 —— 极难发现。

**服务层也是 generator, 用 `yield ev` 转发。** 落库发生在 `yield` 之前, 所以顺序是: 判分 → 落库 → 推给前端。**先落库再推送**, 万一推送时连接断了, 数据也已经安全了。

`if st.finished` 那个检查返回的是 `error` **事件**, 而不是抛 HTTPException。因为这是 SSE 流, 前端已经在读流了 —— 用 4xx 状态码的话前端拿到的是"网络错误", 用 error 事件前端能显示"本场面试已结束"。**协议一旦开始流式, 错误也要走流。**

### 3.4 _persist_turn: 落库 + 记错题本

```python
    def _persist_turn(self, session_id: str, st: SessionState, turn, ev: Event) -> None:
        q = turn.question
        self.db.save_turn(session_id, {
            "turn_index": turn.index,
            "followup_depth": turn.followup_depth,
            "qid": q.qid, "topic": q.topic, "difficulty": q.difficulty,
            "question": q.question, "answer": turn.answer or "",
            "score": ev.data["score"], "depth": ev.data["depth"],
            "scored": bool(turn.grade and turn.grade.scored),
            "hit": ev.data["hit"], "missed": ev.data["missed"], "wrong": ev.data["wrong"],
            "comment": ev.data["comment"], "latency_ms": ev.data["latency_ms"],
        })
        user_key = self._users.get(session_id, "anon")
        if turn.grade and turn.grade.scored and turn.grade.score < WEAK_SCORE:
            self.db.bump_wrong_book(user_key, q.topic, turn.grade.score)
```

**最后那个 `if` 有三个条件, 每一个都不能删:**

- `turn.grade` —— 判分可能失败 (第 7 章 5.5 节的兜底)
- `turn.grade.scored` —— **追问轮不进错题本**, 否则每次追问都会给原 topic 记一笔失误
- `score < WEAK_SCORE` (6.0) —— 只记低分的

`WEAK_SCORE = 6.0` 定义在文件顶部的模块常量里, 和第 7 章 `report.py` 里"薄弱项 < 6.0"是同一个阈值。**同一个业务概念的阈值出现在两个文件里, 是一个已知的小隐患** —— 改的时候要记得改两处。真要收口就提到 `settings` 里, 但那会让"读 report.py 时看不到阈值"变成新问题, 所以现在选择留着。

`self._users.get(session_id, "anon")` 有默认值: 没鉴权的本地开发模式下 `user_key` 就是 `"anon"`, 所有人共用一个错题本。**开发便利和生产正确性的取舍要显式写出来**, 而不是让 `KeyError` 来提醒你。

### 3.5 evict_finished: 内存不能无限涨

```python
    def evict_finished(self, keep: int = 200) -> int:
        """内存里的会话不能无限涨。已结束的先清, 超过 keep 的按插入顺序清最老的。"""
        with self._lock:
            done = [k for k, v in self._sessions.items() if v.finished]
            for k in done:
                self._sessions.pop(k, None)
                self._users.pop(k, None)
            while len(self._sessions) > keep:
                k = next(iter(self._sessions))
                self._sessions.pop(k, None)
                self._users.pop(k, None)
        return len(done)
```

**两级淘汰**: 先清已完成的 (它们的数据已经落库了, 内存里留着没用), 还超就按插入顺序清最老的 (Python 3.7+ 字典保序, `next(iter(d))` 就是最早插入的键)。

**"用户开了一场面试然后关掉浏览器"是最常见的行为**, 这些会话永远不会 `finished`, 会永久占内存。`keep=200` 这个上限就是给它们兜底的 —— 200 个 `SessionState` 大约几 MB, 而没有上限的话跑一个月就是 OOM。

`.pop(k, None)` 而不是 `del`: 两个字典可能不同步 (比如某个 session 只在一个字典里), `del` 会抛 `KeyError` 把整个清理流程中断。**清理逻辑本身不能失败。**

`return len(done)` 返回清了几个: 方便挂到日志或监控上。这个方法目前需要外部触发 (定时任务或管理接口), **没有自动调用 —— 这是一个已知的待办**, 单机小流量下手动重启就够了, 真上量再加一个后台线程定时调。

### 3.6 ask: 问答模式刻意不调 LLM

```python
    def ask(self, question: str, top_k: int | None = None) -> dict:
        hits = self.retriever.search(question, top_k=top_k or settings.top_k)
        return {
            "question": question,
            "hits": [
                {"chunk_id": h.chunk.chunk_id, "doc_id": h.chunk.doc_id,
                 "topic": h.chunk.topic, "heading": h.chunk.heading,
                 "score": round(h.score, 4), "explain": h.explain, "text": h.chunk.raw}
                for h in hits
            ],
            "context": build_context(hits),
        }
```

**这个接口只做检索, 把命中的原文直接返回, 不调 LLM 生成答案。** 理由写在 `api.py` 的 docstring 里:

> 刻意不在这里调 LLM 生成答案 —— 先把检索质量暴露给用户看。
> 检索错了 LLM 一定答错, 让人直接看到证据比看一段流畅的胡话有用。

这是第 5、6 章那套"先量检索再谈生成"的延续。**一个 RAG 系统最常见的失败模式是: 检索召回了错的东西, 生成层用流畅的语言把它包装成看起来很对的答案。** 直接返回原文和 `explain` 字段 (`vec#3,bm25#1` 这种命中来源), 用户自己就能判断"这资料对不对题"。

`"context": build_context(hits)` 同时返回拼好的上下文: 想接生成的话, 这个字段直接喂给 LLM 就行。**把"要不要生成"的决定权留给调用方。**

---

## 四、接口层 `app/api.py`

### 4.1 SSE 的三个 header

```python
SSE_HEADERS = {
    "Cache-Control": "no-cache",
    "Connection": "keep-alive",
    "X-Accel-Buffering": "no",
}
```

**三个都不是可选的:**

| header | 少了会怎样 |
| --- | --- |
| `Cache-Control: no-cache` | 浏览器或中间代理缓存了流, 第二次提交拿到上次的事件 |
| `Connection: keep-alive` | 连接被提前关掉, 流断在半路 |
| `X-Accel-Buffering: no` | **Nginx 攒够一整块 (通常 4KB) 才转发** —— 于是 SSE 变回了"等全部算完一起到" |

**第三个是最坑的**, 因为它只在生产环境暴露: 本地开发直连 uvicorn 一切正常, 一挂上 Nginx 流式效果就消失了, 而代码没有任何改动。`X-Accel-Buffering: no` 是 Nginx 认的一个响应头, 等价于在 Nginx 配置里写 `proxy_buffering off;` —— 但写在响应头里的好处是**部署方不需要知道这件事** (第 11 章的 nginx.conf 里两个地方都写了, 双保险)。

### 4.2 sse(): 一个 3 行的函数

```python
def sse(event: str, data: dict) -> str:
    """一条 SSE 报文。末尾必须是空行, 少一个 \\n 浏览器就一直等不到事件结束。"""
    return f"event: {event}\ndata: {json.dumps(data, ensure_ascii=False)}\n\n"
```

**把格式收口在一个函数里, 那个 `\n\n` 就只可能写错一次。** 如果每个接口自己拼字符串, 迟早有一处漏掉。

`ensure_ascii=False`: 不加的话中文变成 `中文`, 前端能正常解析但你自己 `curl` 调试时完全看不懂。

### 4.3 三个请求体模型: 校验发生在业务代码之前

```python
class StartReq(BaseModel):
    difficulty: int = Field(default=2, ge=1, le=3)


class AnswerReq(BaseModel):
    session_id: str
    answer: str = ""


class AskReq(BaseModel):
    question: str
    top_k: int = Field(default=5, ge=1, le=20)
```

**`ge=1, le=3` 和 `ge=1, le=20` 是免费的安全边界。** FastAPI 会在调用你的函数**之前**校验, 不合法直接返回 422, 业务代码里永远不用写 `if difficulty not in (1,2,3)`。

`top_k` 的上限 20 尤其重要: 没有上限的话 `{"top_k": 999999}` 会让检索层去取十万条结果, 一个请求就能打满内存。**任何来自外部的"数量"参数都必须有上限。**

`answer: str = ""` 默认空串: **交白卷是合法输入** (第 7 章 3.4 节), 不能要求前端必须传。

### 4.4 start: 同步返回第一题

```python
@router.post("/interview/start")
def start(
    req: StartReq,
    user_key: str = Depends(check_quota),
    svc: InterviewService = Depends(get_service),
) -> dict:
    """开一场面试, 同步返回第一题。配额在 check_quota 里已经扣过。"""
    session_id, ev = svc.create(user_key, difficulty=req.difficulty)
    if ev.type == "error":
        raise HTTPException(status_code=503, detail=ev.data.get("message", "选题失败"))
    return {"session_id": session_id, "event": ev.type, "data": ev.data}
```

**`Depends(check_quota)` 一行同时做了鉴权和限流** —— 因为 `check_quota` 自己又 `Depends(verify_key)` (5.2 节)。依赖可以套依赖, FastAPI 会自动去重: 一个请求里 `verify_key` 只执行一次, 即使被多个依赖引用。

**只有这个接口挂 `check_quota`, 其他接口挂 `verify_key`。** 理由在 5.3 节。

`503` 而不是 `500`: 选不出题通常是"知识库空了"或"LLM 生成失败", 属于**服务暂时不可用**而非代码 bug。状态码要让调用方能判断"该重试还是该报 bug"。

### 4.5 answer: SSE 流, 以及那个必须有的 try/except

```python
@router.post("/interview/answer")
async def answer(
    req: AnswerReq,
    user_key: str = Depends(verify_key),
    svc: InterviewService = Depends(get_service),
) -> StreamingResponse:
    """提交答案, SSE 流式推回 grade / followup / hint / question / report。"""

    async def gen() -> AsyncIterator[str]:
        try:
            for ev in svc.submit(req.session_id, req.answer):
                yield sse(ev.type, ev.data)
        except Exception as e:  # 生成器里抛异常会让连接直接断, 前端只看到 "网络错误"
            yield sse("error", {"message": f"服务端异常: {type(e).__name__}: {e}"})
        yield sse("done", {})

    return StreamingResponse(gen(), media_type="text/event-stream", headers=SSE_HEADERS)
```

**这个 `try/except` 和普通接口的异常处理完全不是一回事。**

普通接口抛异常, FastAPI 的异常处理器会返回一个 500 JSON, 前端能读到错误信息。但**流式响应的 header 已经发出去了** (状态码 200 已经确定), 中途抛异常的唯一效果是**连接被掐断**, 前端拿到的是浏览器层面的 "network error", 没有任何有用信息。

所以流里的异常必须**转成一个 error 事件**, 让前端能显示出来。

**`yield sse("done", {})` 在 `try/except` 外面**, 所以无论成功还是失败都会发。前端靠 `done` 判断"这次提交处理完了, 可以解锁输入框" —— 没有它, 出错时输入框会永远卡在 disabled。

**`async def gen()` 里调的是同步的 `svc.submit()`。** 严格说这会阻塞事件循环 (判分那 2~5 秒里整个进程不处理别的请求)。单机小流量下可以接受; 要改的话是把 `svc.submit` 包进 `run_in_threadpool`。**这是一个已知的性能边界, 不是没想到。**

### 4.6 其余四个接口

```python
@router.post("/interview/finish")
def finish(session_id: str = Query(...), ...) -> dict:
    """提前收工: 已答的题照样出报告。"""
    ev = svc.finish(session_id)
    if ev.type == "error":
        raise HTTPException(status_code=404, detail=ev.data["message"])
    return ev.data


@router.get("/interview/{session_id}/report")
def report(session_id: str, ..., db: Database = Depends(get_db)) -> dict:
    r = db.get_report(session_id)
    if r is None:
        raise HTTPException(status_code=404, detail="报告不存在, 这场面试可能还没结束")
    return r
```

**`finish` 走服务层 (内存里的活跃会话), `report` 直接走数据库。** 区别是: `finish` 要**生成**报告 (需要 `SessionState`), `report` 只是**取**已生成的报告。所以老会话 (进程重启前的) 能查报告但不能 finish —— 这个不对称是内存/数据库分工的必然结果。

`404` 那句 detail 写的是 `"报告不存在, 这场面试可能还没结束"` 而不是 `"not found"`: **错误信息要给出最可能的原因。** 用户看到前者知道去 finish 一下, 看到后者只能去问开发。

```python
@router.get("/weak")
def weak(user_key: str = Depends(verify_key), db: Database = Depends(get_db)) -> dict:
    """我的错题本。"""
    return {"topics": db.weak_topics(user_key, limit=10)}
```

**`/weak` 是一个 3 行的接口, 但它是错题本功能对用户可见的唯一窗口。** 没有它, "系统记住了你的弱项"这件事用户完全感知不到 —— 只能靠"咦怎么老问我协程"来猜。**隐式的智能要给一个显式的出口。**

---

## 五、鉴权与限流 `app/core/auth.py`

### 5.1 为什么是 API Key 而不是用户体系

```python
"""鉴权与限流: 一个 API Key 一个身份, 按天算配额。

为什么不做完整的用户体系: 这是个人项目 / 小团队内部工具, 上来就做注册登录邮箱验证
是给自己找事。API Key 模式的好处是 —— 发 Key 就是发账号, 撤 Key 就是封号, 一行环境变量搞定。
真要接公司统一登录, 只需要把 current_user 换成解析 SSO token, 下游全不用改。
"""
```

**"发 Key 就是发账号, 撤 Key 就是封号"** —— 一个完整的注册登录系统要: 用户表、密码哈希、邮箱验证、找回密码、会话管理、CSRF 防护。全部换成一行 `API_KEYS=key1,key2,key3`。

代价是没法自助注册, 得手动发 Key。**对一个自己用 + 给几个朋友用的工具, 这个代价是零。**

### 5.2 verify_key: 三个安全细节

```python
def verify_key(x_api_key: str | None = Header(default=None, alias="X-API-Key")) -> str:
    """返回调用方身份 (user_key)。没配 API_KEYS 时一律当 anon 放行。"""
    allowed = settings.allowed_keys
    if not allowed:
        return "anon"
    if not x_api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="缺少 X-API-Key 请求头",
            headers={"WWW-Authenticate": "ApiKey"},
        )
    for k in allowed:
        if secrets.compare_digest(x_api_key, k):
            return _mask(k)
    raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="API Key 无效")
```

**细节 1: `secrets.compare_digest` 而不是 `==`。**

`==` 比较字符串是**短路**的 —— 第一个字符不同就返回 False, 全部相同才走完整个串。于是**比较耗时泄漏了信息**: 攻击者逐字符试, 哪个字符让响应变慢一点点, 那个字符就是对的。这叫时序侧信道攻击。

`compare_digest` 的比较时间和内容无关, 只和长度有关。**这是一行成本的防御, 没有理由不做。**

**细节 2: 401 和 403 分开。**

| 状态码 | 含义 | 本项目的场景 |
| --- | --- | --- |
| 401 Unauthorized | 你没给凭证 | 没传 `X-API-Key` |
| 403 Forbidden | 凭证有但不对/没权限 | Key 传了但不在白名单 |

分清这两个的实际价值: 前端看到 401 该弹"请输入 API Key", 看到 403 该说"这个 Key 无效"。混成一个的话前端只能给一句含糊的提示。

`headers={"WWW-Authenticate": "ApiKey"}` 是 401 响应的标准配套头, 告诉客户端该用什么认证方式。

**细节 3: 返回指纹而不是明文 Key。**

```python
def _mask(key: str) -> str:
    """日志里只留指纹, 不留明文 Key。"""
    return hashlib.sha256(key.encode()).hexdigest()[:12]
```

`user_key` 会被写进 `sessions.user_key`、`wrong_book.user_key`、`usage_log.user_key` 三张表。**如果存明文 Key, 那么数据库泄漏 = 所有人的凭证泄漏。** 存 sha256 前 12 位: 够唯一 (12 个 hex 字符 = 48 bit), 而且反推不出原 Key。

**代价是"撤 Key 之后历史记录对不上"** —— 换 Key 就等于换了个新身份, 错题本从零开始。对这个规模的项目可以接受。

### 5.3 check_quota: 限的是"开几场"不是"答几题"

```python
def check_quota(user_key: str = Depends(verify_key), db: Database = Depends(get_db)) -> str:
    """按天限流。挂在"开一场面试"这个接口上, 不挂在提交答案上 ——
    限的是"开多少场", 而不是"答多少题", 否则一场面试答到一半被掐掉体验极差。"""
    limit = settings.daily_limit
    if limit <= 0:
        return user_key
    n = db.incr_usage(user_key, _today())
    if n > limit:
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail=f"今日额度已用完 ({limit} 场/天), 明天再来",
            headers={"Retry-After": "3600"},
        )
    return user_key
```

**"限的是开多少场而不是答多少题"是一个产品决策, 不是技术决策。**

按题限流的话: 用户开了一场 6 题的面试, 答到第 4 题额度用完, 前面的努力全废, 报告也拿不到。按场限流的话: 开场时就知道能不能开, 开了就一定能答完。

**成本上两者差不多** (一场 6 题, 6 次 LLM 调用), 但体验差得远。**限流点要选在"用户还没投入成本"的地方。**

`if limit <= 0: return user_key` —— `DAILY_LIMIT=0` 或负数表示不限流。**给一个"关掉这个功能"的开关**, 本地开发和压测时要用。

`Retry-After: 3600` 是 429 的标准配套头。严格说应该是"到明天零点的秒数", 这里简化成一小时 —— 客户端一小时后重试, 拿到 429 再等一小时, 行为是正确的, 只是多试几次。**简化要在正确性成立的前提下做。**

### 5.4 warn_if_open: 一行警告

```python
def warn_if_open() -> None:
    if not settings.allowed_keys:
        print("[auth] 警告: 没有配置 API_KEYS, 接口对所有人开放。生产环境必须配置!")
```

**"没配 Key 就放行"是为了本地开发方便, 但这个便利有可能被带上生产。** 所以启动时打一条醒目的警告 —— 它在 `main.py` 的 `lifespan` 里被调用, 每次启动都会出现在日志第一屏。

**这是"默认不安全但很吵"的设计。** 另一个选项是"默认拒绝, 必须显式配 `ALLOW_ANONYMOUS=true`", 更安全但会让第一次跑起来的人卡住。选前者是因为**这一章的读者是照着文档从零搭的人**, 第一次启动就 401 会让人以为哪里配错了。

**✅ Checkpoint 2**: 鉴权自测。

```bash
uv run python -m app.core.auth
```

真实输出:

```
allowed_keys = (空, 放行 anon)
daily_limit  = 100
指纹示例     = a52782e3a2d4
compare_digest 恒定时间比较: True False
[auth] 警告: 没有配置 API_KEYS, 接口对所有人开放。生产环境必须配置!
```

---

## 六、应用入口 `app/main.py`

### 6.1 lifespan 启动自检

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """启动自检。用 lifespan 而不是 @app.on_event —— 后者在新版 FastAPI 里已废弃。"""
    warn_if_open()
    get_db()  # 建表
    n = KnowledgeIndex().count()
    print(f"[startup] 知识库 {n} 块, 判分模型 {settings.llm_model}, 重排 {settings.use_rerank}")
    if n == 0:
        print("[startup] 警告: 知识库是空的! 先跑 python -m app.rag.index corpus")
    yield
```

**启动时打印三个关键配置, 是一个成本极低、回报极高的习惯。** 部署出问题时第一件事是看日志第一屏 —— 知识库几块 (0 就是没挂 volume)、用的哪个模型 (改了 .env 没生效?)、重排开没开 (性能不对?)。

**这三行日志能省掉一半的"为什么线上和本地不一样"。**

`get_db()` 在这里调一次: `Database.__init__` 里有 `init_schema()`, 所以**建表发生在启动时而不是第一个请求时**。第一个请求就不用等建表, 而且建表失败会在启动阶段暴露 (容器起不来), 而不是在用户面前报 500。

`@asynccontextmanager` + `yield`: `yield` 之前是启动逻辑, 之后是关闭逻辑 (我们没有需要清理的东西, 所以 `yield` 后面是空的)。

### 6.2 CORS: 一个必须收窄的默认值

```python
# 前后端同域部署的话其实不需要 CORS。留着是为了本地开发时前端跑在别的端口。
# 生产环境把 allow_origins 收窄到自己的域名, 不要留 ["*"] 上线。
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)
```

**`allow_origins=["*"]` 配 `allow_credentials=False` 是安全的组合。** 危险的是 `["*"]` 配 `allow_credentials=True` —— 那意味着任何网站都能带着用户的 Cookie 调你的接口。我们用 header 传 Key 不用 Cookie, 所以 `allow_credentials=False` 是对的。

`allow_methods=["GET", "POST"]` 而不是 `["*"]`: 我们只有这两种方法, 显式列出来。**白名单比黑名单可靠。**

第 11 章前后端同域部署 (Nginx 同时服务静态页和 API), 那时 CORS 其实完全不需要 —— 但留着不影响, 而且方便别人把前端拆出去。

### 6.3 /healthz: 为什么要查向量库条数

```python
@app.get("/healthz")
def healthz() -> dict:
    """存活 + 就绪一起查。K8s 里可以拆成 liveness/readiness, 单机 compose 合成一个够用。"""
    n = KnowledgeIndex().count()
    return {
        "status": "ok" if n > 0 else "degraded",
        "chunks": n,
        "model": settings.llm_model,
        "rerank": settings.use_rerank,
        "db": get_db().stats(),
    }
```

**"进程活着"和"服务能用"是两件事。**

最典型的失败: 第 11 章 docker 部署时忘记挂 volume, 于是容器里的 `data/chroma` 是空目录。进程正常启动, `/` 能打开, 但面试第一题就选不出来。如果 `/healthz` 只返回 `{"status":"ok"}`, 你的监控会显示一切正常。

**返回 `chunks: 0` + `status: "degraded"`, compose 的 healthcheck 就能判定不健康, 不把流量导过来。**

`"db": get_db().stats()` 顺带返回数据库统计: 这让 `/healthz` 同时成了一个最简单的运维面板 —— 一个 curl 知道跑了多少场面试、平均分多少。

**✅ Checkpoint 3**: 启动服务并检查健康。

```bash
# 终端 1: 起服务 (OFFLINE=true 走规则判分, 不花钱)
OFFLINE=true MAX_QUESTIONS=3 uv run uvicorn app.main:app --port 8077
```

启动日志:

```
INFO:     Started server process [47001]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8077 (Press CTRL+C to quit)
[auth] 警告: 没有配置 API_KEYS, 接口对所有人开放。生产环境必须配置!
[startup] 知识库 25 块, 判分模型 deepseek-chat, 重排 False
```

```bash
# 终端 2
curl -s localhost:8077/healthz | python -m json.tool --no-ensure-ascii
```

真实输出:

```json
{
    "status": "ok",
    "chunks": 25,
    "model": "deepseek-chat",
    "rerank": false,
    "db": {
        "sessions": 0,
        "qa_records": 0,
        "avg_score": 0.0
    }
}
```

**注意 `model` 显示 `deepseek-chat` 但我们启动时设了 `OFFLINE=true`。** 这是一个**日志误导**: `settings.llm_model` 只是配置值, 离线模式下并不会真的调它。严格说 `/healthz` 该返回"实际生效的判分器"。**记下来当作一个待改项** —— 它不影响功能, 但会在排查"为什么判分这么快"时浪费五分钟。

---

## 七、用 curl 走完整条链路

### 7.1 开一场面试

```bash
curl -s -X POST localhost:8077/api/interview/start \
  -H 'Content-Type: application/json' -d '{"difficulty":2}' \
  | python -m json.tool --no-ensure-ascii
```

真实输出:

```json
{
    "session_id": "1be1f55a6f19",
    "event": "question",
    "data": {
        "index": 1,
        "total": 3,
        "qid": "android-hard-001",
        "topic": "Jetpack/ViewModel",
        "difficulty": 2,
        "question": "手机横过来之后我页面上填的东西全没了, 有什么官方组件能解决这个问题?",
        "followup_depth": 0
    }
}
```

`"total": 3` 来自启动时的 `MAX_QUESTIONS=3`。`session_id` 是第 7 章 `uuid.uuid4().hex[:12]` 生成的 12 位十六进制。

### 7.2 ✅ Checkpoint 4: 看到真正的 SSE 报文

```bash
SID=$(curl -s -X POST localhost:8077/api/interview/start \
  -H 'Content-Type: application/json' -d '{"difficulty":2}' \
  | python -c 'import sys,json; print(json.load(sys.stdin)["session_id"])')

curl -sN -X POST localhost:8077/api/interview/answer \
  -H 'Content-Type: application/json' \
  -d "{\"session_id\":\"$SID\",\"answer\":\"ViewModel 在配置变更时不会被销毁, 靠 ViewModelStore 保存\"}"
```

`curl -N` 关掉 curl 自己的输出缓冲, 否则你看到的还是"一次性全出来"。

真实输出:

```
event: grade
data: {"score": 0.0, "depth": "blank", "hit": [], "missed": ["挂起不阻塞线程, 挂起时线程被释放去执行其他任务", "阻塞是线程占着资源什么都不做", "协程是线程之上的调度单位, 一个线程可承载大量协程"], "wrong": [], "comment": "[规则判分] 命中 0/3 个要点", "latency_ms": 0}

event: question
data: {"index": 2, "total": 3, "qid": "android-hard-004", "topic": "Kotlin/协程", "difficulty": 2, "question": "为什么说协程比线程省资源? 它省在哪?", "followup_depth": 0}

event: done
data: {}
```

**三条报文, 每条以空行结尾, 完全符合 4.2 节的格式。** 这就是前端要解析的东西。

**为什么这次得了 0 分:** 每次 `start` 都是新会话, 选题器随机抽题 —— 这一场抽到的第一题是协程, 而我回答的是 ViewModel。**这恰好演示了第 7 章 7.3 节那个坑**: 固定文本的假答案碰上随机抽的题, 一律 0 分。

### 7.3 ✅ Checkpoint 5: 完整跑完三题拿到报告

真实要验证的是"答对了会怎样", 所以这次按每道题自己的 rubric 作答:

```bash
uv run python - <<'EOF'
import json, urllib.request, yaml, pathlib
qs = yaml.safe_load(pathlib.Path('eval/golden.yaml').read_text(encoding='utf-8'))
G = {q['id']: q for q in qs}

def post(path, body):
    req = urllib.request.Request('http://127.0.0.1:8077' + path,
                                 data=json.dumps(body).encode(),
                                 headers={'Content-Type': 'application/json'})
    return urllib.request.urlopen(req)

r = json.load(post('/api/interview/start', {'difficulty': 2}))
sid, cur = r['session_id'], r['data']
print('session_id =', sid)
print('Q1', cur['qid'], '|', cur['question'])
for i in range(8):
    q = G.get(cur['qid'].split('#')[0])
    ans = '。'.join(p['point'] for p in q['rubric']) if q else '不太清楚'
    raw = post('/api/interview/answer', {'session_id': sid, 'answer': ans}).read().decode()
    print('--- 提交', i + 1, '---'); print(raw.strip())
    done = False
    for blk in raw.split('\n\n'):
        if blk.startswith(('event: question', 'event: followup', 'event: hint')):
            cur = json.loads(blk.split('data: ', 1)[1])
        if blk.startswith('event: report'):
            done = True
    if done:
        break
EOF
```

真实输出 (为可读性省略了中间两次提交的完整 JSON):

```
session_id = 8ceb2f766222
Q1 android-kotlin-001 | 协程的挂起和线程的阻塞有什么本质区别?
--- 提交 1 ---
event: grade
data: {"score": 10.0, "depth": "expert", "hit": ["挂起不阻塞线程, 挂起时线程被释放去执行其他任务", "阻塞是线程占着资源什么都不做", "协程是线程之上的调度单位, 一个线程可承载大量协程"], "missed": [], "wrong": [], "comment": "[规则判分] 命中 3/3 个要点", "latency_ms": 0}

event: question
data: {"index": 2, "total": 3, "qid": "android-hard-004", "topic": "Kotlin/协程", "difficulty": 2, "question": "为什么说协程比线程省资源? 它省在哪?", "followup_depth": 0}

event: done
data: {}
```

第三次提交拿到报告:

```
--- 提交 3 ---
event: grade
data: {"score": 10.0, "depth": "expert", "hit": [...], "missed": [], "wrong": [], "comment": "[规则判分] 命中 3/3 个要点", "latency_ms": 0}

event: report
data: {"session_id": "8ceb2f766222", "question_count": 3, "avg_score": 10.0, "level": "高级", "topics": [{"topic": "Kotlin/协程", "avg_score": 10.0, "depth": "expert", "weak_points": []}], "strengths": ["Kotlin/协程 (均分 10.0)"], "weaknesses": [], "study_plan": ["基础知识点已扎实, 建议加练系统设计与源码级追问"], "total_latency_ms": 0, "total_tokens": 0}

event: done
data: {}
```

**逐条对照前面的设计:**

| 观察 | 对应设计 |
| --- | --- |
| 第 2 题难度还是 2, 第 3 题变成 3 | 第 7 章 5.6: 连续两次 expert 升档 |
| 三题全是 Kotlin/协程 | 选题器第 1 档: 上一场全答错留下的错题本 |
| `study_plan` 是那句兜底文案 | 第 7 章 6.2: 全满分时 `plan` 为空走 fallback |
| `total_tokens: 0` | 离线规则判分不消耗 token |
| 没有一次 followup | 三题都是 expert, `_decide_next` 直接 SWITCH |

**第二行"三题全是 Kotlin/协程"值得停一下** —— 这不是 bug, 是错题本在起作用: 前面那次全交白卷的会话把协程记进了错题本, 于是这一场优先重考它。**功能正常工作的样子有时候看着像 bug, 这也是为什么 `/api/weak` 必须存在。**

### 7.4 ✅ Checkpoint 6: 错题本闭环

先把某个知识点答烂 (全交白卷), 再看错题本和下一场的选题:

```bash
SID=$(curl -s -X POST localhost:8077/api/interview/start \
  -H 'Content-Type: application/json' -d '{"difficulty":2}' \
  | python -c 'import sys,json;print(json.load(sys.stdin)["session_id"])')
for i in 1 2 3 4 5 6; do
  R=$(curl -s -X POST localhost:8077/api/interview/answer \
      -H 'Content-Type: application/json' -d "{\"session_id\":\"$SID\",\"answer\":\"\"}")
  echo "$R" | grep -q 'event: report' && break
done
curl -s localhost:8077/api/weak | python -m json.tool --no-ensure-ascii
```

真实输出:

```json
{
    "topics": [
        "Kotlin/协程",
        "四大组件/Fragment生命周期"
    ]
}
```

再连开三场, 看第一题落在哪:

```bash
for i in 1 2 3; do
  curl -s -X POST localhost:8077/api/interview/start \
    -H 'Content-Type: application/json' -d '{"difficulty":2}' \
    | python -c 'import sys,json;d=json.load(sys.stdin)["data"];print("  第一题:", d["qid"], "|", d["topic"])'
done
```

真实输出:

```
  第一题: android-hard-004 | Kotlin/协程
  第一题: android-lifecycle-003 | 四大组件/Fragment生命周期
  第一题: android-lifecycle-003 | 四大组件/Fragment生命周期
```

**三场的第一题全部落在错题本的两个知识点里, 一次都没落到别的 topic。** 这条链路走完了: `bump_wrong_book` → `weak_topics` → `SessionState.weak_topics` → `pick()` 第 1 档。

**这是整个项目里第一个"跨越两场会话"的功能。** 前面所有东西都在单次请求或单场面试内闭环, 而这个功能的价值只有在用户第二次来的时候才体现 —— 也正因如此, **它是最容易写完之后没人验证的功能**。这个 Checkpoint 就是它的验证。

### 7.5 ✅ Checkpoint 7: 四种错误路径

```bash
echo "--- 报告还没生成 ---"
SID2=$(curl -s -X POST localhost:8077/api/interview/start \
  -H 'Content-Type: application/json' -d '{"difficulty":2}' \
  | python -c 'import sys,json;print(json.load(sys.stdin)["session_id"])')
curl -s -o /dev/null -w '%{http_code}\n' localhost:8077/api/interview/$SID2/report

echo "--- 提前收工 ---"
curl -s -X POST "localhost:8077/api/interview/finish?session_id=$SID2" \
  | python -m json.tool --no-ensure-ascii

echo "--- 不存在的会话 ---"
curl -sN -X POST localhost:8077/api/interview/answer \
  -H 'Content-Type: application/json' -d '{"session_id":"nope","answer":"x"}'

echo "--- 难度越界 ---"
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8077/api/interview/start \
  -H 'Content-Type: application/json' -d '{"difficulty":99}'
```

真实输出:

```
--- 报告还没生成 ---
404
--- 提前收工 ---
{
    "session_id": "8aa536f28799",
    "question_count": 0,
    "avg_score": 0.0,
    "level": "待加强",
    "topics": [],
    "strengths": [],
    "weaknesses": [],
    "study_plan": [
        "基础知识点已扎实, 建议加练系统设计与源码级追问"
    ],
    "total_latency_ms": 0,
    "total_tokens": 0
}
--- 不存在的会话 ---
event: error
data: {"message": "会话不存在或已过期, 请重新开始"}

event: done
data: {}
--- 难度越界 ---
422
```

**四条路径各验证一个设计决定:**

- **404**: 报告靠 `report_json` 是否为空判断, 没 finish 就没有 (2.1 节)
- **一题没答也出报告**: 第 7 章 6.3 节那个 `test_report_empty_session` 守的路径, 在 HTTP 层同样成立
- **error 事件而不是 4xx**: 4.5 节 —— 流式协议的错误也要走流, 而且**后面跟了 `done`**, 前端能正常解锁
- **422**: 4.3 节的 `ge=1, le=3`, 校验发生在业务代码之前

**第二条有个瑕疵**: 一题没答的报告里 `study_plan` 是那句"基础知识点已扎实"。这是 fallback 文案在"没有数据"和"全部满分"两种情况下被复用了。**逻辑没错但文案不对** —— 一个真实项目里该改成"本场没有作答记录"。记在待改项里。

### 7.6 ✅ Checkpoint 8: 鉴权与限流

换个配置重启服务:

```bash
API_KEYS=demo-key-123 DAILY_LIMIT=2 OFFLINE=true MAX_QUESTIONS=3 \
  uv run uvicorn app.main:app --port 8078
```

```bash
echo "--- 无 Key ---"
curl -s -X POST localhost:8078/api/interview/start \
  -H 'Content-Type: application/json' -d '{"difficulty":2}'

echo "--- 错 Key ---"
curl -s -X POST localhost:8078/api/interview/start \
  -H 'X-API-Key: wrong-key' -H 'Content-Type: application/json' -d '{"difficulty":2}'

echo "--- 对 Key, 连开 3 场 (配额 2) ---"
for i in 1 2 3; do
  curl -s -o /tmp/q$i.json -w "  第 $i 场: HTTP %{http_code}\n" \
    -X POST localhost:8078/api/interview/start \
    -H 'X-API-Key: demo-key-123' -H 'Content-Type: application/json' -d '{"difficulty":2}'
done
cat /tmp/q3.json
```

真实输出:

```
--- 无 Key ---
HTTP 401
{"detail":"缺少 X-API-Key 请求头"}
--- 错 Key ---
HTTP 403
{"detail":"API Key 无效"}
--- 对 Key, 连开 3 场 (配额 2) ---
  第 1 场: HTTP 200
  第 2 场: HTTP 200
  第 3 场: HTTP 429
--- 第 3 场响应体 ---
{"detail":"今日额度已用完 (2 场/天), 明天再来"}
```

**401 / 403 / 200 / 429 四个状态码全部按 5.2 和 5.3 节的设计生效。** 注意启动日志里那条 `[auth] 警告` 这次**没有出现** —— 因为配了 `API_KEYS`。**警告消失本身就是配置生效的信号。**

### 7.7 ✅ Checkpoint 9: 数据真的落库了

```bash
python - <<'EOF'
import sqlite3
c = sqlite3.connect('data/app.db'); c.row_factory = sqlite3.Row
print('表:', [r[0] for r in c.execute(
    "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")])
print('sessions   =', c.execute('SELECT COUNT(*) FROM sessions').fetchone()[0])
print('qa_records =', c.execute('SELECT COUNT(*) FROM qa_records').fetchone()[0])
print('wrong_book:')
for r in c.execute('SELECT user_key, topic, miss_count, last_score FROM wrong_book '
                   'ORDER BY miss_count DESC'):
    print(f'  {r["user_key"]:14s} {r["topic"]:22s} miss={r["miss_count"]} last={r["last_score"]}')
print('前三条 qa_record:')
for x in c.execute('SELECT turn_index, followup_depth, qid, topic, score, depth, scored, '
                   'latency_ms FROM qa_records ORDER BY id LIMIT 3'):
    print('  ', dict(x))
print('usage_log:', [dict(x) for x in c.execute('SELECT * FROM usage_log')])
EOF
```

真实输出:

```
表: ['qa_records', 'sessions', 'sqlite_sequence', 'usage_log', 'wrong_book']
sessions = 10
qa_records = 7
wrong_book:
  anon           Kotlin/协程              miss=3 last=0.0
  anon           四大组件/Fragment生命周期      miss=1 last=0.0
前三条 qa_record:
   {'turn_index': 0, 'followup_depth': 0, 'qid': 'android-kotlin-001', 'topic': 'Kotlin/协程', 'score': 0.0, 'depth': 'blank', 'scored': 1, 'latency_ms': 0}
   {'turn_index': 0, 'followup_depth': 0, 'qid': 'android-kotlin-001', 'topic': 'Kotlin/协程', 'score': 10.0, 'depth': 'expert', 'scored': 1, 'latency_ms': 0}
   {'turn_index': 1, 'followup_depth': 0, 'qid': 'android-hard-004', 'topic': 'Kotlin/协程', 'score': 10.0, 'depth': 'expert', 'scored': 1, 'latency_ms': 0}
```

```
usage_log: [{'user_key': 'anon', 'day': '2026-08-18', 'n': 9}, {'user_key': 'a52782e3a2d4', 'day': '2026-08-18', 'n': 3}]
```

**三个值得注意的地方:**

**`sessions = 10` 但 `qa_records = 7`。** 因为很多会话是 `start` 之后就没答题的 (我在测各种错误路径)。`create_session` 在开场时就插一行, `save_turn` 只在提交答案时插 —— **这个不对称是对的**, 它让"开了但没答"这种行为也被记录下来 (可以用来分析流失率)。

**`usage_log` 里有两个 `user_key`。** `anon` 是没配 Key 时那一批请求, `a52782e3a2d4` 是配了 `demo-key-123` 之后的 —— 正好是 5.2 节 `_mask("demo-key-123")` 的输出。**明文 Key 一次都没有落到数据库里。**

**`n=3` 对应"连开 3 场"**, 包括被 429 拒掉的那一次。因为 `incr_usage` 先自增再判断 —— 严格说被拒的那次不该计数。**这是一个真实的小缺陷**: 配额是 2, 用户开第 3 场被拒, 计数变成 3, 下次判断 `3 > 2` 继续拒 —— 行为上没问题 (该拒的还是拒), 但如果 detail 里显示"已用 3/2 场"就会很奇怪。修法是把 `if n > limit` 改成先查再自增, 代价是要多一次查询或者一段更复杂的 SQL。**当前实现是"计数偏大但行为正确", 我选择留着。**

---

## 八、前端: 一个 HTML 文件

### 8.1 为什么不用 EventSource

```javascript
// fetch + ReadableStream 而不是 EventSource: EventSource 只能发 GET, 带不了 body 和自定义 header。
// 手写解析很短 —— 按 \n\n 切事件, 每个事件里认 event: 和 data: 两行。
async function readSSE(resp, onEvent) {
  const reader = resp.body.getReader();
  const dec = new TextDecoder();
  let buf = "";
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buf += dec.decode(value, { stream: true });
    let i;
    while ((i = buf.indexOf("\n\n")) >= 0) {
      const raw = buf.slice(0, i);
      buf = buf.slice(i + 2);
      let type = "message", data = "";
      for (const line of raw.split("\n")) {
        if (line.startsWith("event:")) type = line.slice(6).trim();
        else if (line.startsWith("data:")) data += line.slice(5).trim();
      }
      if (!data) continue;
      try { onEvent(type, JSON.parse(data)); }
      catch (e) { console.warn("坏事件", raw, e); }
    }
  }
}
```

**`EventSource` 是浏览器原生的 SSE 客户端, 但它有两个硬限制:**

1. **只能发 GET** —— 我们要 POST 一个 `{session_id, answer}` 的 body
2. **不能带自定义 header** —— 我们的 `X-API-Key` 没法传

绕过办法是把参数放 URL 查询串、把 Key 放 Cookie。但答案可能很长 (URL 有长度限制), Key 放 Cookie 又要处理 CSRF。**所以手写 22 行解析器, 换掉两个架构妥协。**

**`buf` 那个跨 chunk 的缓冲是关键。** 网络会在任意位置切断数据 —— 一条 SSE 报文可能分两个 chunk 到达, 甚至可能在 `data: {"sco` 这种位置断开。所以不能对每个 chunk 单独解析, 必须**攒进 buffer, 只处理已经出现 `\n\n` 的完整部分**, 剩下的留着等下一个 chunk。

**这是流式处理最容易写错的一处。** 症状是"偶尔丢一个事件", 而且只在网络慢的时候出现 —— 本地测试永远重现不了。

`dec.decode(value, { stream: true })` 那个 `stream: true` 是同一个道理的字符级版本: 一个 UTF-8 中文字符占 3 字节, 可能被切断在中间。`stream: true` 让 decoder 记住不完整的字节序列, 等下次补齐。**没有它, 中文答案偶尔会出现乱码。**

`try/catch` 包住 `JSON.parse` 并 `console.warn`: 一个坏事件不该让整个流处理中断。

### 8.2 RENDER: 事件类型到 DOM 的映射表

前端用一个对象把事件类型映射到渲染函数:

```javascript
const RENDER = {
  question: d => {...},
  followup: d => {...},
  hint: d => {...},
  grade: d => {...},
  report: d => {...},
  error: d => add(`<div class="who">出错了</div><div class="s-lo">${esc(d.message)}</div>`),
};
```

然后消费流的地方是一行:

```javascript
await readSSE(r, (type, data) => {
  if (type === "done") return;
  (RENDER[type] || (() => {}))(data);
});
```

**`(RENDER[type] || (() => {}))` 那个空函数兜底**: 服务端将来加了新事件类型, 老前端会**静默忽略**而不是报错。这让前后端可以独立升级。

**这个映射表和第 7 章 CLI 的 `render_event` 是同一个东西的两个实现** —— 一个 if/elif 链, 一个对象查表。**同一个 FSM, 两个表现层, 零改动。** 这就是第 7 章 5.1 节"FSM 只 yield 不 print"的兑现。

### 8.3 esc(): 一个不能省的函数

```javascript
esc(d.message)
```

所有插入 DOM 的内容都过一遍 `esc()` 做 HTML 转义。**因为面试的问题、答案、rubric 要点全部会被渲染进 `innerHTML`。** 其中"答案"是用户自己输入的 —— 如果不转义, 用户输入 `<img src=x onerror=alert(1)>` 就会被执行。

**这是自己用的工具, 所以只能 XSS 自己?** 不完全 —— 题目文本来自黄金集和 LLM 生成, LLM 输出是不可信的。而且第 10 章会导出这些数据、第 11 章会部署到公网。**转义是一次性成本, 漏掉是永久风险。**

`setBusy(true/false)` 在 `try/finally` 里配对: 请求出错时输入框也要解锁。这和 4.5 节 `yield sse("done", {})` 在 try 外面是同一个考虑 —— **UI 状态的恢复不能依赖成功路径。**

---

## 九、接口测试: `tests/conftest.py` + `tests/test_api.py`

### 9.1 conftest: 三件事必须在 import 应用之前做完

```python
"""pytest 全局夹具。

这个文件解决一个很容易踩的坑: 测试直接 import app.main 的话, 会连上 data/app.db 和
data/chroma —— 跑一遍测试, 你的真实历史记录里就多出十几场假面试。

所以三件事必须在 import 应用之前做完:
1. 把 OFFLINE 设成 true, 判分走规则、向量走 HashEmbedder, 不联网不花钱
2. 把 DB_PATH / CHROMA_DIR 指到临时目录
3. 往临时知识库里灌一遍 corpus, 否则 /healthz 是 degraded, 选题也会失败

环境变量必须在模块顶层设置。因为 app.core.config 是在 import 时读 os.environ 的,
放到 fixture 里就晚了。
"""
_TMP = tempfile.mkdtemp(prefix="itv-test-")
os.environ["OFFLINE"] = "true"
os.environ["DB_PATH"] = f"{_TMP}/test.db"
os.environ["CHROMA_DIR"] = f"{_TMP}/chroma"
os.environ["CORPUS_DIR"] = str(ROOT / "corpus")
os.environ["MAX_QUESTIONS"] = "3"      # 测试里跑 3 题就出报告, 省时间
os.environ["API_KEYS"] = ""            # 默认不鉴权; 需要鉴权的用例自己打开
os.environ["DAILY_LIMIT"] = "1000"
```

**"环境变量必须在模块顶层设置"这句注释救过一整个下午。**

链条是: `app/core/config.py` 里的 `settings = Settings()` 是**模块级语句**, 在 `import app.core.config` 那一刻就执行了, 那时候 `os.getenv` 已经读完。如果把 `os.environ["DB_PATH"] = ...` 放在 fixture 里, fixture 运行时应用早就 import 完了, 设置无效 —— **然后测试就会往你真实的 `data/app.db` 里写数据。**

`conftest.py` 的模块顶层代码在任何测试文件 import 之前执行, 这是 pytest 保证的。所以这里是唯一正确的位置。

**`MAX_QUESTIONS=3` 是纯粹的省时间**: 默认 6 题, 15 个测试里有几个要跑完整场面试, 减一半时间。

**`API_KEYS=""` 默认不鉴权**: 大部分测试关心的是业务逻辑, 每个请求都带 header 是噪音。需要鉴权的三个用例自己打开 (9.2 节)。

```python
@pytest.fixture(scope="session", autouse=True)
def _seed_index() -> None:
    """整个测试会话只灌一次库。HashEmbedder 很快, 25 块不到一秒。"""
    from app.rag.index import KnowledgeIndex
    idx = KnowledgeIndex()
    if idx.count() == 0:
        n = idx.rebuild(os.environ["CORPUS_DIR"])
        assert n > 0, "语料灌不进去, 后面所有用例都没意义"
```

**`scope="session"` + `autouse=True`**: 整个测试会话灌一次库, 而且不用每个测试显式声明依赖。灌库是最慢的一步 (即使用 HashEmbedder), 每个用例灌一次会让 15 个测试跑成两分钟。

`assert n > 0, "语料灌不进去, 后面所有用例都没意义"` —— **在 fixture 里断言, 让前置条件的失败立刻可见。** 没有这一行, 语料路径写错的症状是 15 个测试全部以各种奇怪的方式失败, 你会去逐个查业务代码。

```python
@pytest.fixture
def client_authed(monkeypatch, _seed_index):
    """打开鉴权的客户端。改 settings 实例而不是环境变量 —— 环境变量已经读完了。"""
    monkeypatch.setattr(settings, "api_keys", "test-key")
    with TestClient(app) as c:
        yield c
```

**这里必须改 `settings` 实例而不是环境变量**, 原因就是上面那条: 环境变量早读完了。`monkeypatch.setattr` 的好处是**测试结束自动还原**, 不会影响别的用例。

```python
@pytest.fixture
def client_limited(monkeypatch, _seed_index):
    """配额只有 2 场的客户端, 专门测 429。

    user_key 是 Key 的 sha256 指纹, 每个用例都一样, 所以配额表要清一次,
    否则本用例会被前面用例的计数带偏。
    """
    monkeypatch.setattr(settings, "api_keys", "test-key")
    monkeypatch.setattr(settings, "daily_limit", 2)
    with get_db().conn() as c:
        c.execute("DELETE FROM usage_log")
    with TestClient(app) as c:
        yield c
```

**`DELETE FROM usage_log` 那一行是测试隔离的一个真实教训。** `client_authed` 的用例也用 `test-key`, 它们的调用已经在 `usage_log` 里留了计数。轮到 `client_limited` 时, 第一次 `start` 可能就已经超配额了 —— **测试通过与否取决于执行顺序**, 而 pytest 的顺序会因为 `-k` 过滤、`-p xdist` 并行而变。

**共享状态的测试必须自己清理前置数据。**

### 9.2 parse_sse: 测试里的 SSE 解析器

```python
def parse_sse(text: str) -> list[tuple[str, dict]]:
    out = []
    for block in text.split("\n\n"):
        etype, data = "message", ""
        for line in block.splitlines():
            if line.startswith("event:"):
                etype = line[6:].strip()
            elif line.startswith("data:"):
                data += line[5:].strip()
        if data:
            out.append((etype, json.loads(data)))
    return out
```

**比 8.1 节前端那个版本简单得多, 因为 `TestClient` 返回的是完整响应文本, 不需要处理跨 chunk 的缓冲。** 这也意味着**接口测试测不出 8.1 节那个 buffer bug** —— 测试覆盖的是格式契约, 不是流式行为。**知道测试测不到什么, 和知道它测到什么一样重要。**

### 9.3 十五个测试分五组

```python
def test_healthz(client) -> None:
    r = client.get("/healthz")
    assert r.status_code == 200
    j = r.json()
    assert j["status"] in ("ok", "degraded")
    assert "chunks" in j and "db" in j
```

**`assert j["status"] in ("ok", "degraded")` 而不是 `== "ok"`**: 测试不该假定知识库一定灌上了。真正要验证的是"这个接口返回了合法的结构"。**断言要贴着契约, 不要贴着某一次的运行结果。**

```python
def test_full_interview_flow(client) -> None:
    r = client.post("/api/interview/start", json={"difficulty": 2})
    sid = r.json()["session_id"]
    assert r.json()["data"]["index"] == 1

    report = None
    for _ in range(20):
        if report is not None:
            break
        resp = client.post("/api/interview/answer",
                           json={"session_id": sid, "answer": "onPause 先走"})
        assert resp.headers["content-type"].startswith("text/event-stream")
        events = parse_sse(resp.text)
        assert events[-1][0] == "done", "流必须以 done 结尾, 否则前端不知道读完了"
        assert any(t == "grade" for t, _ in events), "每次提交必须有评分事件"
        for etype, data in events:
            if etype == "report":
                report = data
    assert report is not None, "面试没有正常结束"
    assert report["question_count"] >= 1
    assert report["study_plan"]

    # 报告落库了才算真的完成
    r2 = client.get(f"/api/interview/{sid}/report")
    assert r2.status_code == 200
    assert r2.json()["level"] == report["level"]
```

**这个测试是整章的验收单, 四条断言各守一个不变量:**

- `content-type` 是 `text/event-stream` —— 协议对了
- `events[-1][0] == "done"` —— **流必须以 done 结尾**, 4.5 节那个 try 外面的 yield
- `any(t == "grade")` —— 每次提交必有评分
- 最后三行**从数据库再读一遍报告并比对 level** —— 落库真的发生了

**最后那三行是这个测试的灵魂。** 前面所有断言都只验证了 HTTP 响应; 只有从 `/report` 再读一遍, 才证明 3.3 节 `self.db.finish_session(...)` 真的执行了。**验证副作用要用另一条路径去读。**

`for _ in range(20)` 那个上限: 防止逻辑出错时测试死循环。`MAX_QUESTIONS=3` 加追问最多也就十几轮, 20 是安全的上限。**任何"循环到某个条件成立"的测试都要有硬上限。**

```python
def test_unknown_session_yields_error_event(client) -> None:
    """会话不存在不能返回 500, 要在流里给 error 事件 —— 前端能显示"请重新开始"。"""
    r = client.post("/api/interview/answer", json={"session_id": "nope", "answer": "x"})
    assert r.status_code == 200
    events = parse_sse(r.text)
    assert events[0][0] == "error"
```

**`assert r.status_code == 200` 看着很反直觉** —— 会话不存在居然返回 200? 是的, 这正是 3.3 节的设计: 流式响应的 header 已经发出去了, 错误只能走流。**这个测试的价值是把这个反直觉的决定钉死**, 防止后来的人"顺手改成 404"。

```python
def test_sessions_and_weak_book(client) -> None:
    sid = client.post("/api/interview/start", json={"difficulty": 2}).json()["session_id"]
    for _ in range(12):
        events = parse_sse(client.post("/api/interview/answer",
                                      json={"session_id": sid, "answer": ""}).text)
        if any(t == "report" for t, _ in events):
            break
    assert any(s["session_id"] == sid for s in client.get("/api/sessions").json()["sessions"])
    assert client.get("/api/weak").json()["topics"], "全交白卷, 错题本不能是空的"
```

**"全交白卷, 错题本不能是空的"** —— 这是 7.4 节那个手工 Checkpoint 的自动化版本。错题本是唯一跨会话的功能, 也是最容易在重构中静默失效的功能 (删掉 3.4 节那个 `if` 里的任意一个条件, 它就废了)。**跨会话的功能必须有自动化测试, 因为手工验证要开两场面试。**

```python
def test_daily_limit(monkeypatch, client_limited) -> None:
    """配额是"开几场", 不是"答几题"。第 3 次开场必须 429。"""
    h = {"X-API-Key": "test-key"}
    codes = [client_limited.post("/api/interview/start", json={"difficulty": 2},
                                 headers=h).status_code for _ in range(3)]
    assert codes[:2] == [200, 200]
    assert codes[2] == 429
```

**`assert codes[:2] == [200, 200]` 而不是只断言第三个是 429。** 因为"前两次成功"和"第三次失败"是一对 —— 如果配额逻辑写成了"永远拒绝", 只测第三个照样通过。**测限流要测"限之前是通的"。**

**✅ Checkpoint 10**: 接口测试全绿。

```bash
uv run pytest tests/test_api.py
```

真实输出:

```
...............                                                          [100%]
15 passed in 20.49s
```

**20.49 秒对比第 7 章的 9.54 秒, 多出来的 11 秒是灌向量库 + 起 15 次 TestClient。** 这个代价换来的是"HTTP 契约、SSE 格式、鉴权、限流、落库"全部被覆盖。

**注意每个用例一个新 `TestClient`** (`client` fixture 不是 session scope): 每次都跑一遍 `lifespan`, 也就是每次都是干净的 `InterviewService` 实例。代价是慢, 收益是**用例之间的活跃会话表不互相污染** —— 而 `_SESSIONS` 是模块级单例, 不这样做的话前一个用例的会话会漏到下一个用例。

---

## 十、踩坑与专家提示

**坑 1: SSE 报文少一个 `\n`, 前端永远收不到事件。**
一条报文必须以空行结尾 (`\n\n`)。少一个的症状是前端一直等、服务端日志一切正常。修法是把格式收口在 `sse()` 一个函数里 —— 只可能写错一次。

**坑 2: 本地流式正常, 上了 Nginx 变成一次性返回。**
Nginx 默认攒够一块 (4KB) 才转发。修法是响应头加 `X-Accel-Buffering: no`, 或者 Nginx 配 `proxy_buffering off;`。**写在响应头里的好处是部署方不需要知道这件事。**

**坑 3: 流里抛异常, 前端只看到"网络错误"。**
流式响应的 header 已经发出去了 (200 已定), 中途抛异常只会掐断连接。必须 `try/except` 转成 error 事件。而且 `done` 事件要在 `try` 外面 —— 否则出错时前端的输入框永远解锁不了。

**坑 4: 前端逐 chunk 解析 SSE, 偶尔丢事件。**
网络会在任意位置切断数据, 一条报文可能跨两个 chunk。必须攒 buffer, 只处理已出现 `\n\n` 的完整部分。同理 `TextDecoder` 要传 `{stream: true}`, 否则被切断的中文字符会乱码。**这类 bug 只在网络慢时出现, 本地永远重现不了。**

**坑 5: `turn_before` 在循环里取, 数据库里的问答错位一行。**
`fsm.submit()` 会往 `st.turns` 追加新轮次, 循环开始后 `st.current` 已经变了。必须在循环**之前**取。症状是只在触发追问时错位 —— 极难发现。

**坑 6: 测试直接 import 应用, 污染了真实数据库。**
跑一遍测试, 你的历史记录里多出十几场假面试。修法是在 `conftest.py` **模块顶层**设 `DB_PATH` / `CHROMA_DIR` 到临时目录 —— 放 fixture 里就晚了, 因为 `settings` 是 import 时构造的。

**坑 7: 测试之间通过数据库互相污染, 通过率取决于执行顺序。**
`client_authed` 和 `client_limited` 用同一个 `test-key`, 也就是同一个 `user_key`, 于是 `usage_log` 的计数会串。修法是 `client_limited` 里先 `DELETE FROM usage_log`。**共享状态的测试必须自己清理前置数据。**

**坑 8: 先 SELECT 再 UPDATE/INSERT, 两个请求撞主键。**
用 `ON CONFLICT ... DO UPDATE` 一条 SQL 原子完成。同理 `incr_usage` 的自增和读取必须在同一个事务里, 否则两个请求都读到旧值, 配额被突破。

**坑 9: `database is locked`。**
SQLite 的写锁是全库级的。FastAPI 的线程池会并发写。修法是 `check_same_thread=False` + 一把模块级 `threading.Lock` 自己排队。**自己排队比等它抛错好。**

**坑 10: 内存里的活跃会话永久增长。**
"开了面试然后关掉浏览器"的会话永远不会 `finished`。必须有 `evict_finished` 这类清理, 而且要有 `keep` 上限兜底。**注意: 本项目里这个方法还没有自动调用, 是一个已知待办。**

**坑 11: 用 `==` 比较 API Key。**
字符串 `==` 是短路的, 比较耗时泄漏信息。用 `secrets.compare_digest`。**一行成本的防御。**

**坑 12: 明文 Key 落库。**
`user_key` 会写进三张表。存 sha256 前 12 位, 数据库泄漏就不等于凭证泄漏。代价是换 Key 等于换身份, 历史记录对不上。

**坑 13: 限流挂在"提交答案"上。**
用户答到第 4 题额度用完, 前面全废。挂在"开一场"上, 开了就一定能答完。**限流点要选在用户还没投入成本的地方。**

**坑 14: 用户输入直接进 `innerHTML`。**
答案是用户输入的, 题目来自 LLM 生成 —— 两者都不可信。所有插入 DOM 的内容过 `esc()`。**转义是一次性成本, 漏掉是永久风险。**

**坑 15: `/healthz` 只返回 `{"status":"ok"}`。**
进程活着不等于服务能用。忘挂 volume 时进程正常但知识库是空的。必须查向量库条数, 0 就返回 `degraded`。

**坑 16: `offline or settings.offline` 处理布尔默认值。**
`offline=False` 是有意义的值, `or` 会把它当成"没传"。布尔参数必须用 `if x is None` 判断。

---

## 十一、面试视角

**问 1: 你的接口为什么用 SSE? 和 WebSocket 怎么选的?**

因为一次提交答案会连续产生 2 到 3 个事件, 其中判分要调 LLM (2~5 秒), 而选下一题是毫秒级。用普通 POST 就得攒齐了一起返回, 用户盯着转圈 5 秒; 用 SSE 可以判分一出来先推评分, 用户马上看到分数, 下一题随后到达 —— **同样的总耗时, 感知快一倍。**

不选 WebSocket 是因为我们**只需要单向推送**: 请求本身还是普通 POST, 只有响应要分段。SSE 是纯 HTTP, 过 Nginx 只要关 buffering; WebSocket 要 Upgrade 握手、要额外的 Nginx 配置、断线重连要自己写。**为单向需求付双向的运维成本不值得。**

**问 2: SSE 有什么坑?**

三个。第一, 报文末尾必须是空行, 少一个 `\n` 前端永远等不到事件结束 —— 我把格式收口在一个 3 行的 `sse()` 函数里, 只可能写错一次。第二, **Nginx 默认会 buffer**, 本地直连 uvicorn 一切正常, 上了生产流式效果就消失 —— 加 `X-Accel-Buffering: no` 响应头。第三, **流里抛异常只会掐断连接**, 前端拿到的是浏览器层面的 network error, 没有任何信息 —— 必须 `try/except` 转成 error 事件, 而且 `done` 事件要在 try 外面, 否则出错时前端 UI 永远卡在 loading。

前端那边还有一个: **不能逐 chunk 解析**, 一条报文可能跨两个 chunk 到达, 要攒 buffer 只处理完整部分。这个 bug 只在网络慢时出现。

**问 3: 你的分层是怎么设计的? 为什么要有服务层?**

三层: FSM 只管状态流转, 不知道数据库和 HTTP; 服务层负责"FSM 和数据库怎么配合"; 接口层只管 HTTP 契约。

**服务层存在的理由是有两件事既不属于 FSM 也不属于 HTTP**: 一是"一个事件产生了顺手落库", 二是进程内的活跃会话表 (HTTP 无状态, 但面试是多轮有状态的)。

这个分层有一个可验证的好处: **第 7 章那 17 个 FSM 测试跑完 9.5 秒, 零网络零数据库。** 如果 FSM 直接写库, 那些测试就得连数据库。

**问 4: 会话状态放哪里? 为什么?**

进行中的会话放**内存**, 已完成的明细落**SQLite**。判断依据是: "进行中的面试"丢了不算大事 (重开一场就行), "已完成的报告"丢了不可接受。所以进行中的放内存图快 (每次提交不用反序列化整个 `SessionState`), 完成的落库图稳。

代价是**单进程限制** —— 多个 worker 的话请求可能路由到没有这个会话的进程。要突破就把那个 dict 换成 Redis, 而且只改一个类。**我在注释里写清了这个边界。**

另外内存不能无限涨: "开了面试然后关掉浏览器"的会话永远不会 finished, 所以有一个两级淘汰 (先清已完成的, 超过 200 个按插入顺序清最老的)。

**问 5: 鉴权做了什么? 安全上考虑了什么?**

API Key 模式 —— 发 Key 就是发账号, 撤 Key 就是封号, 一行环境变量搞定。三个安全点:

一, **比较 Key 用 `secrets.compare_digest` 不用 `==`**。`==` 是短路的, 比较耗时泄漏了"前几个字符对不对"的信息, 攻击者能逐字符爆破。

二, **落库存的是 sha256 前 12 位指纹, 不是明文 Key**。`user_key` 会进三张表, 存明文的话数据库泄漏 = 所有凭证泄漏。

三, **配额落在 SQLite 而不是内存**, 进程重启不清零 —— 而崩溃重启恰恰最容易发生在被人刷接口的时候。

另外限流挂在"开一场面试"而不是"提交答案"上: 限的是开多少场, 而不是答多少题, 否则一场面试答到一半被掐掉体验极差。成本上两者差不多, 体验差得远。

**问 6: 一个 500 是怎么排查的? 你留了什么线索?**

启动时打三行日志: **知识库几块、判分模型是哪个、重排开没开**。这三行能省掉一半的"为什么线上和本地不一样" —— 知识库 0 块就是忘挂 volume, 模型不对就是 .env 没生效。

`/healthz` 同时是就绪检查和运维面板: 返回 `chunks` (0 就 `degraded`) 和数据库统计。**"进程活着"和"服务能用"是两件事** —— 忘挂 volume 时进程正常启动、首页能开, 但面试第一题就选不出来; 如果 `/healthz` 只返回 `{"status":"ok"}`, 监控会显示一切正常。

---

## 十二、划重点

1. **SSE 的选择依据是"一次请求产生几个事件"。** 一个事件 (开场选题) 就用普通 JSON, 多个事件 (判分 + 下一题) 才用流。**协议选择按事件个数来, 不要图统一。**
2. **SSE 三行格式 + 末尾空行**, 收口在一个 `sse()` 函数里。
3. **三个响应头都不可选**, 尤其 `X-Accel-Buffering: no` —— 它是唯一只在生产环境暴露的那个。
4. **流式响应的错误必须走流。** `error` 事件而不是 4xx; `done` 事件放在 `try` 外面。
5. **前端 SSE 解析必须攒 buffer。** 逐 chunk 解析会偶尔丢事件, 而且只在网络慢时出现。
6. **三层分工: FSM 不知道数据库, 服务层不知道 HTTP, 接口层不知道 `Turn`。** 可验证的好处是 FSM 的 17 个测试零 IO。
7. **活跃会话放内存, 完成明细落库。** 判断依据: 进行中的丢了可接受, 已完成的不可接受。
8. **`turn_before = st.current` 必须在循环之前取。** 否则数据库里问答错位一行。
9. **先落库再推送。** 推送时断线, 数据已经安全了。
10. **UPSERT 一条 SQL 原子完成**, 不要"先 SELECT 再判断"。配额的自增和读取要在同一个事务里。
11. **SQLite 并发写要 `check_same_thread=False` + 一把全局锁。** 自己排队比等它抛 `database is locked` 好。
12. **`compare_digest` 而不是 `==`**; **指纹入库而不是明文 Key**; **配额落库而不是内存**。
13. **限流点选在"用户还没投入成本"的地方** —— 开场而不是答题。
14. **`/healthz` 要查真实依赖 (向量库条数), 不是只返回 ok。**
15. **启动打印关键配置。** 成本一行, 回报是一半的部署问题当场定位。
16. **测试环境变量必须在 `conftest.py` 模块顶层设置**, 因为 `settings` 是 import 时构造的。
17. **验证副作用要用另一条路径去读。** `test_full_interview_flow` 最后从 `/report` 再读一遍比对 level。
18. **测限流要测"限之前是通的"。** 只断言第三次 429, "永远拒绝"的实现也能通过。
19. **知道测试测不到什么。** `parse_sse` 测的是格式契约, 测不出跨 chunk 的 buffer bug。
20. **`esc()` 不能省。** 用户答案和 LLM 生成的题目都不可信。

---

## 十三、下一章预告

现在你有一个能用的服务: `curl` 能面试, 浏览器能面试, 数据落库了, 错题本闭环了, 32 个测试 (17 + 15) 守着。

但有一个问题一直没回答: **这个判分器到底准不准?**

第 7 章我们做了双向自测 (满分要高分、胡说要低分) 和单调性测试 —— 那只是**下限**, 保证它不是随机数。而"7.5 分和 6.0 分的区别是否真实存在", 到现在为止**没有任何证据**。

**第 9 章建立三层评估体系**:

1. **检索层** —— 第 6 章已经做完 (Recall@3 = 1.0000, MRR = 0.9667)
2. **判分层** —— 人工标注 21 个 (答案, 分数) 样本, 算 MAE、系统性偏差、顺序一致率、皮尔逊相关。**这一层会给出本项目最重要的一个数字。**
3. **对话层** —— 用 `--dump` 存的两条轨迹做 LLM Judge 成对比较, 而且**要做双向消偏** (交换 A/B 位置再问一遍), 因为裁判模型有位置偏好

第 9 章结束时, 你对"我的判分器有多准"这个问题会有一个可以写进简历的答案 —— 而不是"我感觉挺准的"。

---
