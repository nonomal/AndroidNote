# 04 · RAG 第一步: 切分与入库

## 一句话总结

这一章把 7 篇 Markdown 变成 25 个可检索的块, 灌进 Chroma 向量库。核心是三个决定: **按标题切而不是按固定字数切**、**重叠按整句回带而不是按字符截断**、**每块前面拼上"知识点 + 标题路径"再去做向量化**。做完这一章, 知识库第一次真正存在了 —— 用 `collection.count()` 能看到 25。

## 本章你能学到什么

- 为什么切块是 RAG 里最不起眼却最影响效果的一步
- 三种切分策略的对比, 以及为什么本项目选标题感知切分
- 重叠 (overlap) 到底在解决什么问题, 为什么必须按整句
- 为什么送去向量化的文本 (`text`) 和拼进提示词的文本 (`raw`) 要分成两个字段
- Chroma 怎么用: `PersistentClient`、`hnsw:space`、为什么要关掉它自带的 embedding function
- 元数据为什么必须跟着块一起落库, 而不是查完再回表

---

## 一、切块: 为什么这一步最关键

### 1.1 检索的最小单位就是块

**是什么。** 切块 (chunking) 就是把一篇长文档切成若干小段, 每段单独算一个向量、单独进索引。检索时返回的是块, 不是文档。

**为什么不能整篇入库。** 三个原因:

1. **向量会被稀释。** 一篇语料讲了四件事 (七个回调、跳转顺序、返回顺序、onSaveInstanceState), 整篇算一个向量, 这个向量是四个主题的"平均", 结果是问哪个都不太像。
2. **提示词装不下。** 检索出来的内容要拼进提示词让模型出题。整篇 852 字乘 top_k=5 就是四千多字, 而其中大部分和当前问题无关 —— 无关内容会**干扰模型**, 不只是浪费 token。
3. **没法定位。** 报告里想告诉候选人"你漏的这一点在哪儿", 块级能给到"Activity 生命周期 > onSaveInstanceState 的时机", 文档级只能给到"去看那篇 852 字的文档"。

**为什么不能切太碎。** 切成一句一块的话, "onStop 表示界面完全不可见"这句话脱离上下文, 检索到它也没法用来出题。**块的理想大小是"一个完整的、能独立成立的知识单元"** —— 恰好就是语料里一个 `##` 节的大小。这不是巧合, 是第 2 章写语料时就设计好的。

### 1.2 三种切分策略

| 策略 | 怎么做 | 问题 |
| --- | --- | --- |
| 定长切分 | 每 300 字切一刀 | 会把一句话劈成两半, 会把两个无关主题拼在一块 |
| 递归字符切分 | 按 `\n\n` → `\n` → `。` 依次尝试 | 比定长好, 但仍然不知道"这段属于哪个小节" |
| **标题感知切分 (本项目)** | 先按 `#` 分节, 节内再按句子凑到目标长度 | 需要语料有规范的标题层级 |

第三种的**前提条件是第 2 章那条规则**: "`##` 分节, 每节只讲一件事"。语料写规范了, 切块就几乎是免费的; 语料是爬来的乱七八糟的 HTML, 就只能退回第二种。**这是"上游数据规范化省下游工程量"的典型例子。**

---

## 二、`app/rag/chunker.py`

### 2.1 数据结构: 为什么有 `text` 和 `raw` 两个字段

```python
"""切块: 标题感知 + 句子边界 + 整句重叠。"""
from __future__ import annotations

import re
from dataclasses import dataclass, field
from typing import Any

from .loader import Doc

SENT_END = "。!?;！？；\n"


@dataclass
class Chunk:
    chunk_id: str
    text: str          # 带 heading path 前缀, 送去 embedding 的就是这个
    raw: str           # 不带前缀的正文, 拼进提示词用这个
    doc_id: str
    topic: str
    difficulty: int
    tags: list[str] = field(default_factory=list)
    heading: str = ""
    source: str = ""
```

**这两个字段的区分是本章最值得学的设计。**

`text` 长这样 (前面拼了知识点和标题路径):

```
[四大组件/Activity生命周期] Activity 生命周期 > onSaveInstanceState 的时机
onSaveInstanceState 在 onStop 之前调用, 用于保存临时 UI 状态。...
```

`raw` 就是纯正文, 没有前缀。

**为什么向量化要用带前缀的 `text`**: 块内容可能通篇没提"生命周期"这个词 (它只说 onStop、onCreate), 而用户的提问是"生命周期回调顺序"。把 `[四大组件/Activity生命周期]` 拼进去, 等于给这个块补上了它所属的上下文, **检索命中率明显提高**。这个技巧有时叫 contextual chunking。

**为什么拼提示词要用不带前缀的 `raw`**: 提示词里已经会另外说明知识点是什么, 再带一遍前缀是重复的噪声, 而且 `>` 这种符号容易被模型误读成格式标记。

**如果只用一个字段会怎样**: 用带前缀的去拼提示词, 模型生成的题目里会莫名出现"根据 [四大组件/Activity生命周期]..." 这种话; 用不带前缀的去做向量, 检索命中率下降。**同一段文本, 给机器看和给模型看需要不同的形态** —— 这个意识比这段代码本身重要。

### 2.2 `metadata()`: 迁就 Chroma 的一个限制

```python
    def metadata(self) -> dict[str, Any]:
        """Chroma 的 metadata 只接受标量, 所以 tags 拼成字符串。"""
        return {
            "doc_id": self.doc_id,
            "topic": self.topic,
            "difficulty": self.difficulty,
            "tags": ",".join(self.tags),
            "heading": self.heading,
            "source": self.source,
        }
```

**Chroma 的 metadata 值只能是 str / int / float / bool**, 不能是列表。所以 `tags: ["Activity", "生命周期"]` 要拼成 `"Activity,生命周期"`。取出来时再 split。

这看着是个小妥协, 但要知道它的代价: **拼成字符串后就没法按单个 tag 精确过滤了** (只能用 `contains` 之类的模糊匹配)。如果将来 tag 过滤成为核心功能, 就得换一个支持数组字段的向量库 —— 这正是第 0 章说"换库只改 `index.py`"要兑现的地方。

### 2.3 切句: 为什么自己写而不用 NLTK

```python
def split_sentences(text: str) -> list[str]:
    """按句末标点切句, 标点保留在句尾。"""
    out, buf = [], []
    for ch in text:
        buf.append(ch)
        if ch in SENT_END:
            s = "".join(buf).strip()
            if s:
                out.append(s)
            buf = []
    tail = "".join(buf).strip()
    if tail:
        out.append(tail)
    return out
```

十行代码, 没有任何依赖。

**为什么不用 NLTK / spaCy**: 它们的中文分句要额外下载模型, 而且对中文的效果并不比"遇到句末标点就切"好多少。中文的句末标点是明确的 (`。！？；` 加换行), 不像英文有 `Mr. Smith` 这种缩写歧义。**用不用现成库的判断标准: 这个问题在你的语言/领域里有没有真正的歧义。** 中文分句没有, 所以自己写十行更可控。

**`SENT_END` 里为什么有半角 `!?;`**: 语料里混着半角标点是常态 (输入法状态、复制粘贴)。只判全角会导致一整段不切分, 直接变成一个超长块。

**`\n` 也算句末**: 语料里一行一句, 换行是天然的边界。

### 2.4 按标题分节: 维护一个标题栈

```python
def split_by_heading(body: str) -> list[tuple[str, str]]:
    """按 Markdown 标题分节, 返回 [(heading_path, 该节正文)]。"""
    sections: list[tuple[str, str]] = []
    stack: list[str] = []
    buf: list[str] = []

    def flush() -> None:
        content = "\n".join(buf).strip()
        if content:
            sections.append((" > ".join(stack), content))

    for line in body.splitlines():
        m = re.match(r"^(#{1,6})\s+(.*)$", line)
        if m:
            flush()
            buf = []
            level = len(m.group(1))
            stack = stack[: level - 1]
            while len(stack) < level - 1:
                stack.append("")
            stack.append(m.group(2).strip())
        else:
            buf.append(line)
    flush()
    return sections
```

**这个函数的核心是 `stack`, 它维护"当前在哪一层标题下"。**

碰到 `# Activity 生命周期` (level 1): `stack` 截断到 0 个元素, 然后 append, 变成 `["Activity 生命周期"]`。
碰到 `## 七个回调` (level 2): `stack` 截断到 1 个元素 (保留 level 1), 然后 append, 变成 `["Activity 生命周期", "七个回调"]`。
于是 heading path = `Activity 生命周期 > 七个回调`。

**为什么要完整路径而不只是当前标题**: 因为"七个回调"这四个字单独看毫无信息量。带上父级标题, 检索时就知道是"Activity 生命周期"的七个回调。

**`while len(stack) < level - 1: stack.append("")` 是干什么的**: 处理**跳级**的情况 —— 有人写了 `#` 然后直接写 `###`, 中间少了 `##`。不补空串的话 `stack[: level-1]` 会得到一个比预期短的列表, 后面 append 上去层级就错位了。语料规范里说了不要跳级, 但**代码不能假设人一定守规范**。

**`flush()` 在两处被调用**: 遇到新标题时 (把上一节存起来) 和循环结束后 (存最后一节)。少了第二个 `flush()`, **最后一节会被静默丢掉** —— 这是这类循环最经典的 off-by-one 错误, 而且它不报错, 只是少了一块。

**✅ Checkpoint 1**: 看一篇语料被分成了哪几节。

```bash
uv run python -c "
from app.rag.loader import load_corpus
from app.rag.chunker import split_by_heading
docs = {d.doc_id: d for d in load_corpus('corpus')}
for h, body in split_by_heading(docs['lifecycle-activity-basic'].body):
    print(f'{h!r} -> {len(body)}字')
"
```

真实输出:

```
'Activity 生命周期 > 七个回调' -> 239字
'Activity 生命周期 > 页面跳转时两个 Activity 的回调顺序' -> 195字
'Activity 生命周期 > 返回时的顺序' -> 167字
'Activity 生命周期 > onSaveInstanceState 的时机' -> 152字
```

四个 `##` 节, 每节 150~240 字。**注意每节都在 300 字以内** —— 这意味着这篇语料的切块几乎完全由标题决定, `chunk_size` 根本没起作用。这是好事: 说明第 2 章的语料规范写对了。

### 2.5 `chunk_doc`: 节内凑长度 + 整句回带

```python
def chunk_doc(doc: Doc, chunk_size: int = 300, overlap: int = 60) -> list[Chunk]:
    """一篇文档 -> 若干 Chunk。overlap 以整句为单位回带, 不切断句子。"""
    chunks: list[Chunk] = []
    for sec_i, (heading, content) in enumerate(split_by_heading(doc.body)):
        sents = split_sentences(content)
        cur: list[str] = []
        cur_len = 0

        def emit(heading: str = heading, sec_i: int = sec_i) -> None:
            nonlocal cur, cur_len
            raw = "".join(cur).strip()
            if not raw:
                return
            prefix = f"[{doc.topic}] {heading}\n" if heading else f"[{doc.topic}]\n"
            chunks.append(
                Chunk(
                    chunk_id=f"{doc.doc_id}#{sec_i}-{len(chunks)}",
                    text=prefix + raw,
                    raw=raw,
                    doc_id=doc.doc_id,
                    topic=doc.topic,
                    difficulty=doc.difficulty,
                    tags=list(doc.tags),
                    heading=heading,
                    source=doc.source,
                )
            )
            # 整句回带: 从尾部往前凑够 overlap 个字
            back: list[str] = []
            total = 0
            for s in reversed(cur):
                if total >= overlap:
                    break
                back.insert(0, s)
                total += len(s)
            cur = back
            cur_len = total

        for s in sents:
            if cur_len + len(s) > chunk_size and cur:
                emit()
            cur.append(s)
            cur_len += len(s)
        emit()
    return chunks
```

**主循环只有五行**: 逐句累加, 加不下了就 `emit()` 吐出一块。看着简单, 但有四个地方值得停下来。

**第一: `and cur` 这个条件。** 如果某一句本身就超过 `chunk_size` (比如一句 400 字的长句), 不加 `and cur` 的话, `cur` 是空的也会 `emit()`, 而 `emit` 里 `if not raw: return` 会直接返回, 于是死循环 —— 不, 不会死循环, 但会**每句都触发一次无效 emit**。加上 `and cur` 表达的是"只有攒了东西才吐"。**超长单句会形成一个超过 chunk_size 的块, 这是有意的**: 宁可块偏大, 也不切断句子。

**第二: 那个 `default 参数绑定` 的技巧。**

```python
def emit(heading: str = heading, sec_i: int = sec_i) -> None:
```

代码注释里解释了原因:

> heading / sec_i 用默认参数显式绑进来, 而不是靠闭包捕获循环变量。这里闭包其实是安全的 (emit 只在本轮迭代内被调用), 但显式绑定能通过 linter 的 B023 检查, 也能防止以后有人把 emit 挪出循环 —— 那时闭包会读到最后一轮的值。

**这是 Python 一个经典陷阱**: 闭包捕获的是变量本身而不是它当时的值。如果把 `emit` 收集起来最后统一调用, 所有 `emit` 看到的 `heading` 都会是最后一节的标题。第 1 章 ruff 配置里的 `B` 规则组就是查这类问题的 (B023 = function definition does not bind loop variable)。

**第三: 整句回带的实现。** 从 `cur` 的尾部往前取句子, 直到累计长度达到 `overlap`。取到的句子成为下一块的开头。

**为什么必须按整句而不是按字符数**: 假设按字符截断, 上一块的最后 60 个字可能是"...所以 onPause 里不能做耗" —— 下一块开头就是这半句话。这半句话进了向量, 它表达的语义是残缺的; 更糟的是它拼进提示词后, 模型可能照着这半句编出一道错题。

**第四: `chunk_id` 的格式 `{doc_id}#{sec_i}-{len(chunks)}`。** 比如 `perf-memory-leak#1-2` 表示"这篇文档、第 1 节、全篇第 2 块"。带上两个数字是为了**排查时能一眼看出结构**: 同一节切成了两块 (`#1-1` 和 `#1-2`) 就说明这一节超过 300 字了。

### 2.6 看一眼重叠的实际效果

`perf-memory-leak` 那篇的"五个高频泄漏场景"一节有 364 字, 超过 300, 所以被切成了两块。

**✅ Checkpoint 2**: 看这两块的接缝。

```bash
uv run python -c "
from app.rag.loader import load_corpus
from app.rag.chunker import chunk_doc
docs = {d.doc_id: d for d in load_corpus('corpus')}
cs = chunk_doc(docs['perf-memory-leak'])
a, b = cs[1], cs[2]
print('块A 尾部:', repr(a.raw[-100:]))
print()
print('块B 开头:', repr(b.raw[:100]))
"
```

真实输出:

```
块A 尾部: '是静态变量持有 Context, 应当改持 Application Context 或用弱引用。第三是注册了监听器没有反注册, 比如 BroadcastReceiver、EventBus、传感器回调。'

块B 开头: '第二是静态变量持有 Context, 应当改持 Application Context 或用弱引用。第三是注册了监听器没有反注册, 比如 BroadcastReceiver、EventBus、传感器回'
```

块 B 的开头把块 A 结尾的两句话完整重复了一遍。**重叠在解决什么问题**: 如果一个知识点正好横跨接缝 (比如"第三是注册了监听器没有反注册"这句在 A 的末尾), 没有重叠的话, 检索"监听器泄漏"只能命中 A, 而 A 的主体讲的是 Handler 和静态变量 —— 排名会偏低。有了重叠, B 也包含这句, 两块都有机会命中。

**代价**: 存储和向量化成本增加约 20%, 而且**同一个知识点会出现在两个块里**, 检索结果可能出现内容重复的两条。第 5 章的 RRF 融合和 top_k 截断会缓解, 但不能完全消除。

### 2.7 每块的最终形态

**✅ Checkpoint 3**: 看一个块的完整字段。

```bash
uv run python -c "
from app.rag.loader import load_corpus
from app.rag.chunker import chunk_doc
docs = {d.doc_id: d for d in load_corpus('corpus')}
c = chunk_doc(docs['lifecycle-activity-basic'])[3]
print('chunk_id =', c.chunk_id)
print('text 前 90 字 =', repr(c.text[:90]))
print('metadata =', c.metadata())
"
```

真实输出:

```
chunk_id = lifecycle-activity-basic#3-3
text 前 90 字 = '[四大组件/Activity生命周期] Activity 生命周期 > onSaveInstanceState 的时机\nonSaveInstanceState 在 onStop 之'
metadata = {'doc_id': 'lifecycle-activity-basic', 'topic': '四大组件/Activity生命周期', 'difficulty': 2, 'tags': 'Activity,生命周期,onSaveInstanceState', 'heading': 'Activity 生命周期 > onSaveInstanceState 的时机', 'source': 'developer.android.com/guide/components/activities/activity-lifecycle'}
```

第 3 章标注的 `gold_chunks: [lifecycle-activity-basic]` 就是靠**前缀匹配**这个 `chunk_id` 判命中的 —— 现在你能看到为什么标文档级前缀是对的: `#3-3` 这个后缀会随切分参数变。

### 2.8 全量切块

```python
def chunk_corpus(docs: list[Doc], chunk_size: int = 300, overlap: int = 60) -> list[Chunk]:
    out: list[Chunk] = []
    for d in docs:
        out.extend(chunk_doc(d, chunk_size, overlap))
    return out
```

**✅ Checkpoint 4**: 切全部语料。

```bash
uv run python -m app.rag.chunker corpus
```

真实输出:

```
7 篇 -> 25 块
块长: min=55 max=291 avg=152
------------------------------------------------------------
jetpack-viewmodel#0-0  heading='ViewModel > 它解决什么问题'
  ViewModel 让数据在配置变更时存活。屏幕旋转会销毁并重建 Activity, 普通成员变量全部丢失。ViewModel 由 View...
jetpack-viewmodel#1-1  heading='ViewModel > 它不能解决什么'
  ViewModel 挡不住进程被杀。系统内存不足杀掉进程后, ViewModel 一起消失。跨进程死亡的状态恢复要靠 SavedStateH...
jetpack-viewmodel#2-2  heading='ViewModel > 为什么不能持有 View 或 Context'
  ViewModel 的生命周期比 Activity 长, 持有 View 或 Activity Context 会导致泄漏。需要 Conte...
```

**`7 篇 -> 25 块` 这个数字后面会反复出现**: 第 6 章的评测输出里、`/healthz` 接口的返回里、Docker 启动日志里。它是"知识库是否完整"的一个速查指标 —— 哪天看到 `18 块`, 说明有语料没被加载。

每篇的块数:

```
jetpack-viewmodel          4 块
kotlin-coroutine-basic     4 块
lifecycle-activity-basic   4 块
lifecycle-fragment         3 块
perf-anr                   3 块
perf-memory-leak           4 块
lifecycle-service          3 块
```

`min=55` 那个最短的块是某个只有一两句话的小节。**块长差异大 (55~291) 是标题感知切分的正常现象**, 因为块边界由语义 (标题) 决定, 而不是由长度决定。定长切分能给你整齐的 300/300/300, 但那种整齐是没有意义的整齐。

### 2.9 `chunk_size` 改了会怎样: 实测

很多教程会说"chunk_size 是要调的超参"。对本项目, 我们量一下:

```bash
uv run python -c "
from app.rag.loader import load_corpus
from app.rag.chunker import chunk_corpus
docs = load_corpus('corpus')
for cs in (150, 300, 500, 900):
    for ov in (0, 60):
        ch = chunk_corpus(docs, cs, ov)
        L = [len(c.raw) for c in ch]
        print(f'{cs:>6} {ov:>4} -> {len(ch):>3} 块  min={min(L)} max={max(L)} avg={sum(L)//len(L)}')
"
```

真实输出:

```
chunk_size  overlap    块数   min   max   avg
       150        0    34    43   150   109
       150       60    40    55   208   132
       300        0    25    55   291   148
       300       60    25    55   291   152
       500        0    24    55   358   154
       500       60    24    55   358   154
       900        0    24    55   358   154
       900       60    24    55   358   154
```

**结论: `chunk_size` 从 300 加到 900, 块数只从 25 变到 24。** 因为语料的 `##` 节本来就都在 300 字左右, `chunk_size` 基本不起作用 —— 真正决定切块的是标题。

**这个测量有两个价值。** 一是省事: 不用花时间调一个不起作用的参数。二是它揭示了标题感知切分的本质 —— **切分质量的上游是语料的结构质量**。想让块变小, 应该去语料里多加几个 `##`, 而不是调 `chunk_size`。

注意 `chunk_size=150` 那两行: 块数从 25 涨到 34/40, 因为这时长度限制开始生效, 强行把节切开了。`overlap=60` 让块数从 34 变成 40 —— 重叠回带的内容本身又占了长度, 于是需要更多块装同样的内容。**overlap 会放大块数, 这是它成本的一部分。**

---

## 三、入库: `app/rag/index.py`

### 3.1 Chroma 的三行初始化

```python
COLLECTION = "android_kb"
DEFAULT_DB = os.getenv("CHROMA_DIR", "data/chroma")


class KnowledgeIndex:
    """一个知识库 = Chroma collection + 内存 BM25。"""

    def __init__(self, db_dir: str = DEFAULT_DB, embedder: Embedder | None = None):
        import chromadb

        pathlib.Path(db_dir).mkdir(parents=True, exist_ok=True)
        self.client = chromadb.PersistentClient(path=db_dir)
        # 关掉 Chroma 自带的 embedding function, 向量一律由我们自己算
        self.col = self.client.get_or_create_collection(
            name=COLLECTION, metadata={"hnsw:space": "cosine"}, embedding_function=None
        )
        self.embedder = embedder or get_embedder()
        self._bm25: BM25 | None = None
        self._chunks: list[IndexedChunk] = []
        self._pos: dict[str, int] = {}
```

四个决定:

**`import chromadb` 写在函数里而不是文件顶部。** chromadb 的 import 要一秒多 (它会加载 onnxruntime 等一堆东西)。写在 `__init__` 里, 那些只用到 `chunker` 或 `loader` 的脚本 (比如上面的 Checkpoint) 就不用付这个代价。**延迟导入重依赖, 是让 CLI 工具保持敏捷的实用技巧。**

**`hnsw:space: "cosine"`** 指定用余弦距离建 HNSW 索引。默认是 L2 (欧氏距离)。因为我们的向量已经归一化了 (`normalize_embeddings=True`), 归一化向量的余弦相似度和点积等价, 而 L2 距离和余弦在归一化后单调相关 —— 理论上都能用, 但显式写 cosine 让"返回的 distance 怎么转成相似度"这件事有唯一答案 (`1 - distance`)。**不写这一行, 换台机器 Chroma 换个默认值, 你的阈值 0.35 就全错了。**

**`embedding_function=None` 是最重要的一行。** Chroma 默认会自己下载一个模型来算向量 (`all-MiniLM-L6-v2`, 英文模型)。如果不关掉:
- 你会在 `col.add()` 时被迫下载一个你不需要的模型
- 更糟的是**查询和入库用的模型可能不一致**, 检索结果全是噪声, 而不报任何错

关掉它, 向量全部由我们自己的 `BGEEmbedder` 算, 入库和查询用的是同一个模型。

**`embedder: Embedder | None = None` 支持注入。** 测试时可以传一个 `HashEmbedder` 进来, 不用真加载 100MB 模型。第 12 章 CI 全程离线就是靠这个。

### 3.2 `rebuild`: 为什么全量重建

```python
    def rebuild(self, corpus_dir: str, chunk_size: int = 300, overlap: int = 60) -> int:
        """全量重建。教程阶段用这个: 语料一改就重跑, 不用操心增量的边界情况。"""
        docs = load_corpus(corpus_dir)
        chunks = chunk_corpus(docs, chunk_size, overlap)
        with contextlib.suppress(Exception):
            # 集合不存在时 delete 会抛异常, 这里就是要"有则删无则算"
            self.client.delete_collection(COLLECTION)
        self.col = self.client.get_or_create_collection(
            name=COLLECTION, metadata={"hnsw:space": "cosine"}, embedding_function=None
        )
        self.add(chunks)
        return len(chunks)
```

**为什么不做增量更新。** 增量要处理一堆边界情况: 语料改了一句话, 是哪几个块变了? 一篇语料删掉了, 它的块怎么找出来删? `chunk_id` 里带序号, 改了一节内容后面所有块的编号都变了 —— 增量更新会留下一堆孤儿块。

25 个块全量重建只要几秒。**在数据量小的时候, "重建"永远比"增量"正确**, 因为它没有状态残留。什么时候该做增量? 当重建时间长到影响你迭代节奏时 (比如超过一分钟)。在那之前做增量是纯粹的过早优化, 而且引入的 bug 极难发现 —— 你不会注意到库里多了 3 个过期块。

**`contextlib.suppress(Exception)` 而不是 try/except pass**: 语义完全一样, 但 `suppress` 一眼能看出"这里就是要忽略异常", 而 `except: pass` 常常是"忘了处理"。这是 ruff `SIM105` 规则会提示的写法。

### 3.3 `add`: 一次批量写

```python
    def add(self, chunks: list[Chunk]) -> None:
        if not chunks:
            return
        vecs = self.embedder.encode_docs([c.text for c in chunks])
        self.col.add(
            ids=[c.chunk_id for c in chunks],
            documents=[c.raw for c in chunks],
            embeddings=[v.tolist() for v in vecs],
            metadatas=[c.metadata() for c in chunks],
        )
        self._bm25 = None  # 索引脏了, 下次查询时重建
```

**注意 `text` 和 `raw` 的分工在这里落地了**: `embeddings` 用 `c.text` 算 (带前缀), `documents` 存 `c.raw` (不带前缀)。检索时返回的 `documents` 就是干净正文, 可以直接拼提示词。

**`encode_docs` 传的是整个列表, 不是循环调用。** 模型批量编码 25 条比逐条调用快好几倍 (`batch_size=32` 一批就搞完)。逐条调用是新手最常见的性能问题, 而且在 25 条时感觉不出来, 到 25000 条时慢一百倍。

**`self._bm25 = None` 是一个失效标记。** BM25 索引是内存结构, 数据变了就不能用旧的。设成 None 表示"脏了", 下次访问 `self.bm25` 时会重新构建。这叫 lazy invalidation —— 不立刻重建 (可能后面还有更多 `add`), 而是标记为脏, 等真要用时再算。

### 3.4 BM25 为什么不落库

文件顶部的注释就是答案:

> 为什么 BM25 不落库: 它是纯内存结构, 启动时用 Chroma 里已存的文本重建即可 (毫秒级), 比自己维护第二套持久化简单得多。

```python
    def _load_all(self) -> None:
        got = self.col.get(include=["documents", "metadatas"])
        self._chunks = []
        for cid, doc, meta in zip(got["ids"], got["documents"], got["metadatas"], strict=True):
            m = meta or {}
            topic = str(m.get("topic", ""))
            heading = str(m.get("heading", ""))
            self._chunks.append(
                IndexedChunk(
                    chunk_id=cid,
                    text=f"[{topic}] {heading}\n{doc}",
                    raw=doc,
                    topic=topic,
                    difficulty=int(m.get("difficulty", 0)),
                    tags=str(m.get("tags", "")),
                    heading=heading,
                    doc_id=str(m.get("doc_id", "")),
                    source=str(m.get("source", "")),
                )
            )
        self._pos = {c.chunk_id: i for i, c in enumerate(self._chunks)}
        self._bm25 = BM25([tokenize(c.text) for c in self._chunks])
```

**`text=f"[{topic}] {heading}\n{doc}"` 这一行是在还原前缀。** 入库时只存了 `raw`, 但 BM25 要用带前缀的 `text` 建索引 (和向量侧保持一致)。前缀能从 metadata 里的 `topic` 和 `heading` 重新拼出来 —— 所以不用额外存一份。**能推导出来的数据就不要存, 存两份就会有不一致的那天。**

**`strict=True` 是 Python 3.10+ 的 zip 参数**, 表示三个列表长度必须相等, 不等就报错。不加的话 `zip` 会静默地按最短的截断 —— 如果 Chroma 因为某种原因返回的 metadatas 少了一条, 你会**丢块而不知道**。第 1 章 ruff 的 `B` 规则组里 B905 专门查这个。

**`self._pos` 是一个 chunk_id → 下标的字典。** 因为第 5 章的检索会返回一堆 chunk_id, 需要用 id 反查完整对象。用字典是 O(1), 用列表遍历是 O(n) —— 25 个块无所谓, 但这是一行代码就能换来的正确复杂度。

### 3.5 三个 property: 按需加载

```python
    @property
    def chunks(self) -> list[IndexedChunk]:
        if self._bm25 is None:
            self._load_all()
        return self._chunks

    @property
    def bm25(self) -> BM25:
        if self._bm25 is None:
            self._load_all()
        assert self._bm25 is not None
        return self._bm25
```

**`if self._bm25 is None` 作为"未加载"的判断依据**, 三个 property 共用同一个标记。这样第一次访问任意一个都会触发一次完整加载, 后续都命中缓存。

**`assert self._bm25 is not None` 不是运行时检查, 是给类型检查器看的。** `_load_all()` 里确实赋了值, 但静态分析器推不出来, 不加 assert 它会认为返回类型是 `BM25 | None`。

### 3.6 灌库

**✅ Checkpoint 5**: 建索引。

```bash
uv run python -m app.rag.index corpus
```

真实输出:

```
入库 25 块, collection.count()=25
BM25 词表大小=904, 平均块长(token)=69.2
```

**首次运行会慢 20~90 秒**, 因为要下载 100MB 的 bge 模型。第二次就是几秒。如果卡在下载, 检查 `.env` 里有没有 `HF_ENDPOINT=https://hf-mirror.com`。

**`collection.count()=25` 和上面 `7 篇 -> 25 块` 对得上** —— 这个交叉验证是这个 Checkpoint 的全部意义。如果入库数少于切块数, 说明有块因为 id 重复被覆盖了。

**BM25 词表 904** 表示 25 个块里一共出现了 904 个不同的 token (第 5 章会讲这个 token 是怎么切的)。**平均块长 69.2 token** 对应上面 `avg=152` 字 —— 因为一个中文字算一个 token, 而英文 API 名字整个算一个。

顺便确认一下磁盘上真的有东西了:

```bash
ls data/chroma
```

会看到一个 `chroma.sqlite3` 和一个 UUID 命名的目录 (HNSW 索引文件)。**这就是"持久化"的含义**: 进程退出后数据还在, 下次启动不用重新灌。第 11 章 Docker 部署时, 这个目录会挂成一个卷。

### 3.7 顺便: embedding 模块的自测

第 5 章会详细讲 `embedding.py`, 但这里可以先确认模型是活的。

**✅ Checkpoint 6**:

```bash
uv run python -m app.rag.embedding
```

真实输出:

```
embedder=BGEEmbedder dim=512
  doc0 cos=0.5411  A.onPause 先于 B.onCreate,
  doc1 cos=0.4061  ViewModel 由 ViewModelSto
  doc2 cos=0.3433  协程挂起不阻塞线程。
最相关: doc 0
```

问的是"从一个页面跳到另一个页面, 回调顺序是什么", 最相关的是讲跳转回调顺序的 doc0 (0.5411) —— 语义检索在工作。

**注意 doc2 (完全无关的协程) 也有 0.3433。** 这是余弦相似度的一个重要特性: **它没有绝对意义上的"不相关 = 0"**。中文向量模型给任意两段中文的相似度通常都在 0.3 以上。所以第 5 章的阈值定 0.35 而不是 0.1 —— 这个数字是看着这类实测数据定的, 不是拍的。

如果这里打出 `embedder=HashEmbedder dim=256`, 说明模型没加载成功 (降级了)。检查一步: `uv run python -c "import sentence_transformers"` 有没有报错。

---

## 踩坑与专家提示

1. **`embedding_function=None` 千万别漏。** 漏了的话 Chroma 会用它默认的英文小模型算向量, 而你的查询用 bge 算 —— 两套向量空间不兼容, 检索结果是纯噪声, **而且不报错**。这是本章最贵的一个坑。
2. **改了切分参数一定要重建索引。** 库里是旧块、代码是新参数, 检索出来的 `chunk_id` 对不上 `self._pos`, 直接 `KeyError`。习惯是: 动了 `chunker.py` 就跑一遍 `python -m app.rag.index corpus`。
3. **`data/chroma` 不要提交 git。** 第 1 章的 `.gitignore` 里有 `data/`。它是运行产物, 而且是二进制, 提交会让仓库迅速膨胀。
4. **不要在语料里放代码块。** 第 2 章说过, 这里给出机制原因: `split_by_heading` 只认 `#` 开头的行, 而代码块里的注释 `# TODO` 会被当成一级标题, 把一段代码劈成两个"章节"。
5. **`zip(..., strict=True)`。** 处理多个平行列表时永远加上。静默截断导致的数据丢失是最难查的一类 bug。
6. **不要为了"块整齐"去用定长切分。** 块长 55~291 的不均匀是语义边界的正常结果。整齐的块长不是质量指标, 检索指标才是。
7. **`chunk_size` 在语料规范的项目里几乎是个死参数。** 先量再调 —— 上面那张表花了 30 秒, 省下的是"我要不要试试 500"的反复纠结。

---

## 面试视角: 本章高频五问

**Q1: 你的切块策略是什么? 为什么不用固定长度?**

标题感知切分: 先按 Markdown 标题层级分节, 节内再按句子边界凑到目标长度, 超长时按整句回带 60 字重叠。不用固定长度因为它会在语义中间切断 —— 一句话被劈成两半, 两个块的向量都是残缺的。实测显示这个策略下 `chunk_size` 从 300 加到 900 块数只变 1 (25 → 24), 说明切块实际由标题决定 —— 这也反过来说明**切分质量的上游是语料的结构质量**。

**Q2: 重叠 (overlap) 解决什么问题? 代价是什么?**

解决"知识点横跨块边界"的问题: 一句话正好落在块尾, 检索时它所在的块主题不匹配, 排名会偏低。加了重叠, 相邻块都包含这句话, 两块都有机会命中。代价有三个: 存储和向量化成本增加约 20%、检索结果可能出现内容重复的两条、块数增加 (实测 chunk_size=150 时 overlap 让块数从 34 涨到 40)。所以 overlap 不是越大越好, 60 字大约是"一到两句话"。

**Q3: 为什么送去 embedding 的文本和拼进提示词的文本不一样?**

因为用途不同。embedding 用带前缀的版本 (`[知识点] 标题路径\n正文`), 因为块内容可能通篇不出现"生命周期"这个词而用户就是这么问的 —— 前缀补上了它的上下文, 提高命中率。拼提示词用纯正文, 因为提示词里已经另外说明了知识点, 前缀是重复噪声, `>` 之类的符号还可能被模型误读成格式标记。**同一段文本, 给检索引擎看和给生成模型看需要不同形态。**

**Q4: 为什么 BM25 索引不持久化?**

因为它能从已持久化的文本毫秒级重建, 而维护第二套持久化会引入"两份数据不一致"的风险。判断标准是重建成本: 25 个块重建是毫秒级, 那就不该存。同样的道理, 块的 `text` 字段 (带前缀) 也没存 —— 它能从 metadata 里的 topic 和 heading 拼出来。**能推导的不要存, 存两份就会有不一致的那天。**

**Q5: 为什么全量重建而不做增量更新?**

因为增量的边界情况多且难验证: 改一句话是哪几个块变了、删一篇语料怎么找出它的块、`chunk_id` 带序号所以改一节后面全部错位。全量重建 25 块只要几秒, 而且没有状态残留。什么时候该转增量? 当重建时间影响迭代节奏时 (比如超过一分钟)。在那之前做增量是过早优化, 而且它的 bug 特别隐蔽 —— 库里多了三个过期块, 没有任何报错。

---

## 划重点

1. 检索的最小单位是块。整篇入库会稀释向量、塞爆提示词、丢失定位能力; 切太碎会让块脱离上下文不可用。
2. 标题感知切分 = 按 `#` 分节 + 节内按句凑长度 + 整句回带重叠。它的前提是语料有规范的标题层级。
3. `text` (带 `[知识点] 标题路径` 前缀) 用于向量化, `raw` (纯正文) 用于拼提示词。两者必须分开。
4. 重叠必须按整句回带, 不能按字符截断 —— 半句话进向量是残缺语义, 拼进提示词会让模型编错题。
5. Chroma 必须 `embedding_function=None`, 否则它用自己的英文模型算向量, 和查询侧不一致且不报错。
6. `hnsw:space: "cosine"` 要显式写, 否则默认值一变, 你的相似度阈值全部失效。
7. 数据量小时"全量重建"永远比"增量更新"正确, 因为它没有状态残留。
8. `7 篇 -> 25 块 -> count()=25` 这个交叉验证是知识库完整性的速查指标。
9. `chunk_size` 在语料规范的项目里几乎是死参数 —— 实测证明了它, 省下了调参时间。

## 下一章预告

有了 25 个块和它们的向量, 下一章开始真正的检索: 手写一个中文 BM25 (不用 jieba, 只有 60 行)、接上向量召回、用 RRF 把两路结果融合, 再加上 cross-encoder 精排和相似度阈值。这一章会第一次看到"同一个问题, BM25 和向量给出的答案不一样"—— 而这正是混合检索存在的理由。

