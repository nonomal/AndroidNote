# 07 · 面试官状态机与命令行版

## 一句话总结

这一章做的是整个项目最容易被做错的部分: **不用一个大提示词让模型扮演面试官**, 而是写一个六状态的显式状态机 —— 模型只负责判断"这个要点讲到了没有", 分数由代码按权重算, 追问几层、什么时候降难度、什么时候结束, 全部是 `if` 语句决定的。本章结束时你能在终端跑完一场三题面试并拿到一份复盘报告, 全程不花一分钱。

## 本章你能学到什么

- 为什么"一个提示词扮演面试官"在多轮对话里一定会漂, 具体漂在哪三处
- 六个状态怎么划分, 状态流转图怎么落成代码
- Pydantic 在这里的真实角色: 不是类型标注, 是 **LLM 输出的质检员**
- 分数为什么必须由代码算: `hit_points` + 权重 -> 0~10, 每处错误扣 1.5
- 一个 90 行的离线规则判分器, 让整条链路在没有 API Key 的机器上也能跑
- `scored` 这一个布尔字段如何避免"一次追问把满分题的均分从 10 拽到 5"
- 追问 (followup) 和纠正 (hint) 的区别, 以及为什么追问不能占题目配额
- 难度自适应为什么要"连续两次"而不是"单次"

---

## 一、为什么不能用一个大提示词

### 1.1 提示词版面试官漂在哪

最直觉的做法是这样一段系统提示词:

> 你是一位资深 Android 面试官。请依次向候选人提问 6 道题, 每道题根据回答打 0~10 分, 答得好就追问深一层, 答得差就换简单的题, 最后给出一份复盘报告。

这段提示词在第一轮完美工作, 然后开始漂。三处具体的失效, 都是真实会发生的:

**第一处: 忘记评分。** 到第三轮, 模型开始只回复"很好, 那我们看下一题", 分数没了。因为对话历史越长, 系统提示词在注意力里的占比越低。

**第二处: 重复出题。** 第 5 题问了一遍第 2 题的知识点, 换了个说法。模型没有"已问过的知识点"这个可靠的状态, 它只有对话历史 —— 而它对历史的记忆是概率性的。

**第三处: 追问层次失控。** 你说"答得好就追问深一层", 但"一层"是多深? 模型可能在一个知识点上追五层, 六道题的面试变成了两道题的深挖。

**最要命的第四处: 出了问题无法定位。** 报告里均分算错了, 你不知道是模型忘了某一轮的分数、还是算错了平均、还是把追问轮也算进去了。**你唯一能做的调试动作是改提示词然后再试一次**, 而这不叫调试, 叫许愿。

### 1.2 六个状态

显式状态机把上面每一处都变成了一行可读、可测、可改的代码:

```
SELECT --(选到题)--> ASK --(收到答案)--> GRADE --> BRANCH
BRANCH --followup/hint--> ASK   (同一知识点, depth+1)
BRANCH --switch--> SELECT
BRANCH --end--> REPORT --> DONE
```

| 状态 | 干什么 | 谁在干 |
| --- | --- | --- |
| SELECT | 挑下一道题 | 代码 (选题器) |
| ASK | 把题目推给用户, 等回答 | 代码 |
| GRADE | 判断每个要点命中没有 | **模型** |
| BRANCH | 决定追问 / 提示 / 换题 / 结束 | 代码 |
| REPORT | 生成复盘报告 | 代码 (纯计算) |
| DONE | 结束 | — |

**六个状态里只有一个用到模型。** 这不是为了省钱 (虽然确实省), 而是因为其余五步都是**确定性任务**: 挑题是查表加过滤, 算分是加权求和, 分支是四条 `if`, 报告是分组统计。**把确定性任务交给非确定性组件, 是这类项目最常见的架构错误。**

### 1.3 职责边界怎么划

一句可以直接用在面试里的判断标准:

> **模型只做它做得比代码好的那件事。**

模型做得比代码好的: 判断"候选人这句话和这个要点是不是一个意思" (语义等价判断)。这件事写规则会写死。

代码做得比模型好的: 加权求和、去重、计数、比较、分支。这些事模型会做错, 而且错得没有规律。

第 3 章讲 rubric 时说过"分数完全由它算出来", 本章就是那句话的落地。

---

## 二、数据契约: `app/agent/schemas.py`

### 2.1 Pydantic 在这里的角色

文件开头两行注释说明了一切:

```python
"""面试流程的数据契约。Pydantic 在这里的角色是 LLM 输出的质检员。

分支决策依赖字段完整性, 所以每个模型都给了默认值和取值约束: 模型少写一个字段不至于让整场面试崩掉。
"""
```

**"分支决策依赖字段完整性"** 是关键。`_decide_next` 要读 `grade.depth`, 如果模型这一轮没吐 `depth` 字段, 没有默认值的话就是 `AttributeError`, 整场面试崩在第三轮。有默认值的话就是"按 surface 处理, 继续走"。

**这是本章最重要的一条工程判断: 面对 LLM 的输出, 容错优先于严格。** 严格校验适合内部接口 (对方是你的代码, 错了就是 bug); LLM 是外部不可控输入, 它一定会时不时少个字段。

### 2.2 三个枚举

```python
class Stage(str, Enum):
    SELECT = "select"
    ASK = "ask"
    GRADE = "grade"
    BRANCH = "branch"
    REPORT = "report"
    DONE = "done"


class Depth(str, Enum):
    BLANK = "blank"      # 完全不会 / 交白卷
    SURFACE = "surface"  # 背了结论, 说不出机制
    SOLID = "solid"      # 机制清楚
    EXPERT = "expert"    # 能讲权衡和边界


class NextAction(str, Enum):
    FOLLOWUP = "followup"  # 同一知识点追深
    HINT = "hint"          # 给提示后让他再答一次
    SWITCH = "switch"      # 换知识点
    END = "end"            # 结束面试
```

**为什么继承 `str` 而不只是 `Enum`**: `str, Enum` 的成员本身就是字符串, 所以 `json.dumps` 能直接序列化, Pydantic 的 `model_dump()` 出来是 `"solid"` 而不是 `<Depth.SOLID: 'solid'>`。第 8 章要把这些结构推给前端 SSE, 少了 `str` 就得手写一个 encoder。

**`Depth` 为什么是四档而不是分数**: 因为它是**模型判断的**, 而模型判断连续量 (7 分还是 8 分) 极不稳定, 判断离散档位 (背了结论 vs 讲清机制) 相对稳。**让模型做分类, 不要让模型做回归。** 分数由代码从要点权重算, 和 depth 是两条独立的信息。

**`NextAction` 的四个值是穷尽的**, 而且互斥。这就是为什么 `BRANCH` 状态可以只用四条 `if` 实现。

### 2.3 `GradeResult`: 三层防御

```python
class GradeResult(BaseModel):
    score: float = 0.0
    hit_points: list[str] = Field(default_factory=list)
    missed_points: list[str] = Field(default_factory=list)
    wrong_points: list[str] = Field(default_factory=list)
    depth: Depth = Depth.SURFACE
    next_action: NextAction = NextAction.SWITCH
    followup: str | None = None
    comment: str = ""
    scored: bool = True
```

**第一层防御: 每个字段都有默认值。** 模型吐一个 `{}` 也能构造出对象。

**第二层防御: `mode="before"` 的钳位校验器。**

```python
    # mode="before" 很关键: 校验前先夹回区间。
    # 如果写成默认的 after 校验, 模型吐个 12.7 会直接抛 ValidationError 让整轮崩掉,
    # 而我们要的是"把不讲理的分数掰正", 不是"因为一个分数放弃这次面试"。
    @field_validator("score", mode="before")
    @classmethod
    def _clamp(cls, v: object) -> float:
        try:
            f = float(v)  # type: ignore[arg-type]
        except (TypeError, ValueError):
            return 0.0
        return max(0.0, min(10.0, round(f, 2)))
```

`mode="before"` 和默认的 `mode="after"` 差别很大:

| | `after` (默认) | `before` (本项目) |
| --- | --- | --- |
| 执行时机 | Pydantic 先做类型校验, 再跑你的函数 | 你的函数先跑, 再做类型校验 |
| 模型吐 `12.7` | 通过类型校验 (是 float), 你的函数可以钳位 | 一样能钳位 |
| 模型吐 `"八分"` | **类型校验直接抛 ValidationError** | 你的 `try/except` 兜住, 返回 0.0 |

**所以 `before` 的真正价值是能兜住类型错误, 不只是范围错误。** 模型偶尔会吐 `"score": "8/10"` 这种东西。

**注意 `except (TypeError, ValueError)` 而不是裸 `except`**: 裸 `except` 会吞掉 `KeyboardInterrupt`, 而且会掩盖真正的 bug。只捕获你预期的两种。

**第三层防御: 非法枚举不兜, 直接拦。** `depth` 字段模型如果吐了 `"牛逼"`, Pydantic 会抛 `ValidationError`。**为什么这一个反而不容错?** 因为枚举只有四个合法值, 吐出第五个说明模型没在按指令工作 —— 这时候"猜一个默认值继续跑"比报错危险。而 `score` 越界只是数值不讲理, 语义还在。

**容错的边界在于: 你能不能合理地推断出它本来想说什么。** 12.7 分明显想说"满分", `"牛逼"` 推不出来。

### 2.4 ✅ Checkpoint 1: 三种脏输入

`schemas.py` 底部有一段 `__main__` 自测, 专门喂三种脏数据:

```python
if __name__ == "__main__":
    # 缺字段 / 越界 / 非法枚举, 三种脏输入都得被兜住或拦下
    g = GradeResult.model_validate({"score": 12.7, "hit_points": ["a"]})
    print(f"score 越界被夹回: {g.score}, depth 默认值: {g.depth}, action 默认值: {g.next_action}")
    g2 = GradeResult.model_validate({})
    print(f"全空也能构造: score={g2.score} has_error={g2.has_error}")
    try:
        GradeResult.model_validate({"score": 5, "depth": "牛逼"})
    except Exception as e:
        print(f"非法枚举被拦下: {type(e).__name__}")
    q = Question(qid="t1", topic="Kotlin/协程", question="协程和线程的区别?",
                 rubric=[{"point": "挂起不阻塞线程", "weight": 4}, {"point": "一线程多协程", "weight": 3}])
    print(f"rubric 总权重 = {q.total_weight}")
```

```bash
uv run python -m app.agent.schemas
```

真实输出:

```
score 越界被夹回: 10.0, depth 默认值: Depth.SURFACE, action 默认值: NextAction.SWITCH
全空也能构造: score=0.0 has_error=False
非法枚举被拦下: ValidationError
rubric 总权重 = 7
```

**最后一行是故意的**: 这里的 rubric 权重是 4+3=7, 不是 10。`total_weight` 返回 7 说明**判分公式用的是实际权重和, 不是硬编码的 10**:

```python
    @property
    def total_weight(self) -> int:
        return sum(p.weight for p in self.rubric) or 1
```

**`or 1` 是防除零。** 第 3 章要求评测集的权重和必须是 10, 但那是**对人的约束** (`check_golden.py` 强制); 而 LLM 生成的题目 (4.3 节) 的 rubric 权重和可能不是 10, 代码必须能算。**约束靠校验脚本, 代码要能容忍。**

### 2.5 `scored`: 一个布尔字段防住的坑

```python
    # 追问轮没有人工 rubric, 分数没有可比的基准, 所以只记 depth 不计入均分。
    # 少了这个开关, 一次追问就能把一道满分题的均分从 10 拽到 5。
    scored: bool = True
```

这个注释里的"从 10 拽到 5"是真的。追问轮 (`_ask_deeper` 造的 `follow_q`) 的 `rubric` 是空列表, 于是判分器只能给 0 分 —— **不是因为答得差, 是因为没有要点可判**。

**✅ Checkpoint 2**: 亲眼看这个坑。

```bash
uv run python -c "
from app.agent.schemas import GradeResult, Question, SessionState, Turn
q = Question(qid='a', topic='t', question='q', rubric=[{'point':'p','weight':10}])
fq = Question(qid='a#f1', topic='t', question='追问')
st = SessionState(session_id='x')
st.turns.append(Turn(index=0, question=q, grade=GradeResult(score=10.0, scored=True)))
st.turns.append(Turn(index=1, question=fq, grade=GradeResult(score=0.0, scored=False)))
print('scores(只算主问题) =', st.scores, '-> avg =', st.avg_score)
bad = [t.grade.score for t in st.turns if t.grade]
print('如果不区分 scored     =', bad, '-> avg =', round(sum(bad)/len(bad), 2))
"
```

真实输出:

```
scores(只算主问题) = [10.0] -> avg = 10.0
如果不区分 scored     = [10.0, 0.0] -> avg = 5.0
```

**一道满分题, 因为多问了一句追问, 均分变成 5.0。** 而且这个 bug 极其隐蔽: 面试跑得很顺, 报告也生成了, 只是分数莫名其妙低。**候选人答得越好 (触发的追问越多), 分数越低** —— 一个方向完全反了的 bug。

实现只有两个 property:

```python
    @property
    def scores(self) -> list[float]:
        """只统计有 rubric 的主问题, 追问轮 (scored=False) 不计入均分。"""
        return [t.grade.score for t in self.turns if t.grade and t.grade.scored]

    @property
    def avg_score(self) -> float:
        s = self.scores
        return round(sum(s) / len(s), 2) if s else 0.0
```

**`if s else 0.0` 又是一次防除零。** 一场面试可能一题都没答就结束 (用户点开就关掉), 这时候 `avg_score` 必须是 0.0 而不是崩掉。第 6 章的 `dcg` 里也有同样形状的守卫 —— **凡是有除法就要问"分母能不能是 0"**, 这个习惯能省掉一半的线上报错。

### 2.6 `SessionState`: 会话的全部状态

```python
class SessionState(BaseModel):
    session_id: str
    stage: Stage = Stage.SELECT
    difficulty: int = Field(default=2, ge=1, le=3)
    asked_topics: list[str] = Field(default_factory=list)
    asked_qids: list[str] = Field(default_factory=list)
    turns: list[Turn] = Field(default_factory=list)
    finished: bool = False
    # 历史错题本: 上几场面试没答好的知识点, 开场时由 service 层从库里注入。
    # 放在 SessionState 里而不是 FSM 里, 是因为 FSM 不该知道数据库的存在。
    weak_topics: list[str] = Field(default_factory=list)
```

**`asked_qids` 和 `asked_topics` 就是 1.1 节"重复出题"那个问题的解法。** 提示词版面试官靠模型回忆自己问过什么, 这里是两个显式的列表。**状态显式化是状态机的全部价值**, 不是"看起来更专业"。

**`weak_topics` 那两行注释值得读三遍。** 它解释的是一个分层设计: FSM 是纯函数式的, 它读 `st.weak_topics` 但不知道这个列表从哪来。第 8 章的 service 层从 SQLite 查出"这个用户上几场哪些知识点答砸了"填进去。

**如果 FSM 直接查库会怎样**: FSM 的单测就必须准备一个数据库。第八节那 17 个测试能在 9 秒内跑完、不碰任何 IO, 全靠这条边界。

`Turn` 是最小记录单元:

```python
class Turn(BaseModel):
    """一问一答一评。既是会话状态, 也是第 10 篇导出微调数据的原始单元。"""

    index: int
    question: Question
    answer: str = ""
    grade: GradeResult | None = None
    followup_depth: int = 0
    latency_ms: int = 0
    tokens: int = 0
```

**"既是会话状态, 也是微调数据的原始单元"** —— 第 10 章导出训练样本时, 直接遍历 `st.turns` 就有了 (问题, 回答, 评分) 三元组。**不需要另写一套埋点。** 这和第 3 章"一份 golden.yaml 服务四件事"是同一个思路: 让一份数据结构承担多个下游用途, 代价是设计时多想半小时。

---

## 三、判分器: 分数由代码算

### 3.1 公式

`app/agent/grader.py` 里 6 行的函数, 是整个项目可信度的支点:

```python
def _score_from_points(q: Question, hit: list[str], wrong: list[str]) -> float:
    """命中权重占比 -> 0~10 分, 每条错误扣 1.5 分。"""
    hit_set = {h.strip() for h in hit}
    got = sum(p.weight for p in q.rubric if p.point.strip() in hit_set)
    base = 10.0 * got / q.total_weight
    return max(0.0, min(10.0, base - 1.5 * len(wrong)))
```

**✅ Checkpoint 3**: 手算验证。

```bash
uv run python -c "
from app.agent.grader import _score_from_points
from app.agent.schemas import Question
def Q():
    return Question(qid='b', topic='t', question='q',
                    rubric=[{'point':'A','weight':3},{'point':'B','weight':3},{'point':'C','weight':4}])
print('3+3 命中/总10, 无错  ->', _score_from_points(Q(), ['A','B'], []))
print('全命中 + 1 处说错     ->', _score_from_points(Q(), ['A','B','C'], ['x']))
print('全命中 + 3 处说错     ->', _score_from_points(Q(), ['A','B','C'], ['x','y','z']))
print('模型抄错要点文本      ->', _score_from_points(
    Question(qid='b',topic='t',question='q',rubric=[{'point':'A 要点','weight':10}]), ['A要点'], []))
"
```

真实输出:

```
3+3 命中/总10, 无错  -> 6.0
全命中 + 1 处说错     -> 8.5
全命中 + 3 处说错     -> 5.5
模型抄错要点文本      -> 0.0
```

前三行都能手算: `10 × 6/10 = 6.0`; `10.0 - 1.5 = 8.5`; `10.0 - 4.5 = 5.5`。**这就是"可解释"的含义 —— 候选人问"我凭什么 8.5 分", 你能一步步算给他看。**

**第四行是这个设计的软肋, 必须正视。** 要点原文是 `"A 要点"` (中间有空格), 模型抄回来的是 `"A要点"` (没空格), `in hit_set` 匹配失败, 权重 10 全丢, 得 0 分。

`h.strip()` 只能处理首尾空白, 处理不了中间的差异。三种可选的加固:

| 方案 | 做法 | 代价 |
| --- | --- | --- |
| 让模型返回序号 | 提示词改成"返回命中的要点编号" | 模型对文本比对序号更可靠, 但会出现"编号 4"这种越界值 |
| 模糊匹配 | 去掉所有空白后比对, 或算编辑距离 | 引入新的误判 (两个相似要点互相串) |
| 归一化 | 比对前统一去空白、统一全半角 | 最便宜, 但只覆盖已知的差异 |

**本项目选了不加固, 而是把提示词里的要求写死为"原样抄回"**, 并接受偶发的漏判。理由是: 第 9 章会量出判分器和人工标注的 MAE 是 0.833, 这个数字里**已经包含了这类漏判的代价**。**已经量过的缺陷比没量过的加固更让人安心** —— 加固可能引入新的、没量过的误判。

面试时能把这段说清楚, 比说"我做了一个 AI 判分器"有分量得多。

### 3.2 LLMGrader 的提示词: 四条规则各防一件事

```python
GRADE_SYSTEM = """你是一位严格但公正的 Android 技术面试官, 正在给候选人的回答评分。

评分规则:
1. 逐条检查下面给出的每一个评分要点, 判断候选人的回答是否讲到了
2. 只看技术内容是否正确, 不看表达是否流畅, 不因为用词和参考答案不同就判为没讲到
3. 如果候选人说了明确错误的技术内容, 记入 wrong_points, 这比漏讲严重
4. 不要因为候选人多讲了要点之外的正确内容而扣分

只输出一个 JSON 对象, 字段如下:
{
  "hit_points": ["原样抄回命中的要点文本"],
  "missed_points": ["原样抄回漏掉的要点文本"],
  "wrong_points": ["候选人说错的地方, 用你自己的话描述"],
  "depth": "blank | surface | solid | expert",
  "comment": "一句话点评, 不超过 40 字"
}
```

**每条规则都在防一个具体的失效, 不是凑字数:**

- **规则 1 "逐条检查"** 防的是模型整体印象打分。不说"逐条", 模型会读完答案给一个笼统判断, 然后随便挑几个要点填进 `hit_points`。
- **规则 2 "不因为用词不同就判为没讲到"** 防的是字符串匹配式判断。这条是 LLMGrader 存在的**全部理由** —— 如果它只做字面匹配, 那用 RuleGrader 就行了, 不用花 API 钱。
- **规则 3 "这比漏讲严重"** 防的是模型把答错和漏答混为一谈。**漏答是不知道, 答错是知道错的**, 后者在面试里危险得多 (会写出错的代码), 所以要额外扣 1.5 分, 而且触发 HINT 分支去纠正。
- **规则 4 "多讲不扣分"** 防的是模型惩罚啰嗦。候选人紧张时会多讲, 那不是缺点。

**注意 `hit_points` 和 `missed_points` 要求"原样抄回", 而 `wrong_points` 要求"用你自己的话"。** 因为前两个要参与代码的字符串匹配 (3.1 节), 后一个只用来展示和触发分支。**哪个字段要被代码消费, 哪个字段就必须约束格式。**

`depth` 的四档判定标准也写进了提示词:

```
- blank: 完全不会, 或答的和问题无关
- surface: 只背出结论, 讲不出机制和原因
- solid: 机制讲清楚了
- expert: 讲清机制, 还能讲权衡、边界条件或源码层面
```

**给枚举值配上判定标准, 而不是只给值。** 光写 `"depth": "blank | surface | solid | expert"`, 模型会按自己的理解分档, 而且每次理解不一样。

### 3.3 `temperature=0.0`

```python
        data = chat_json(
            [...],
            temperature=0.0,
        )
```

**判分必须是 0 温度。** 同一份答案两次判分给出不同结果, 是候选人最不能接受的事, 也让第 9 章的一致性评测失去意义。

对比 4.3 节的出题: `temperature=0.7`。**出题要多样, 判分要稳定** —— 温度参数是按任务性质定的, 不是全局设一个值了事。

### 3.4 空答案短路

```python
        scored = bool(q.rubric)
        if not answer.strip():
            return GradeResult(
                score=0.0,
                missed_points=[p.point for p in q.rubric],
                depth=Depth.BLANK,
                next_action=NextAction.SWITCH,
                comment="没有作答",
                scored=scored,
            )
```

**空答案不调模型。** 三个理由: 省钱、省时间、结果更对 (模型面对空字符串的行为是不确定的, 可能给 3 分)。

**而且 `missed_points` 填的是全部要点** —— 这样报告里的错题本会记下"这个知识点的所有要点都没答", 而不是一片空白。**交白卷是信息量最大的一种回答, 不要把它的信息丢掉。**

### 3.5 RuleGrader: 让链路能离线跑

```python
class RuleGrader:
    """离线判分器: 把 rubric 要点切成关键词, 按加权覆盖率判命中。

    只用于自测和 CI。它的存在意义是: 让整条链路在没有 API Key 的机器上也能跑通并被测试。
    ASCII 标识符 (onPause、ViewModelStore) 权重给 3, 中文二元组权重给 1 ——
    面试答案里 API 名字对上了才是真懂, 中文措辞每个人都不一样。
    """

    KEY = re.compile(r"[A-Za-z][A-Za-z0-9_.]{2,}|[一-鿿]{2,}")
    STOP = {"可以", "需要", "因为", "所以", "如果", "就是", "这个", "那个", "什么", "之后", "之前", "发生", "调用", "表示", "也就是"}
    W_ASCII = 3
    W_CJK = 1
    HIT_RATIO = 0.5
```

**`W_ASCII = 3` vs `W_CJK = 1` 这个 3 倍差距是本项目的核心判断**: 候选人说得出 `onSaveInstanceState` 这个词, 基本就是真懂; 而"在...之前调用"这种中文措辞, 每个人的说法都不一样, 匹配上了也不能证明什么。

`HIT_RATIO = 0.5` 意思是要点关键词的加权覆盖率过半就算命中。这个 0.5 是拍的, 没有校准 —— 但它不重要, 因为 RuleGrader 不用于生产。

关键词抽取里有一个坑:

```python
    def _keywords(self, text: str) -> dict[str, int]:
        """返回 {关键词: 权重}。点分标识符拆成子词, 不保留复合形式,
        否则 'A.onPause' 和 'onPause' 会被当成两个不同的词, 白白拉低覆盖率。"""
        out: dict[str, int] = {}
        for m in self.KEY.finditer(text):
            w = m.group(0)
            if w.isascii():
                for part in w.split("."):
                    if len(part) > 2:
                        out[part.lower()] = self.W_ASCII
            else:
                for i in range(len(w) - 1):
                    bi = w[i : i + 2]
                    if bi not in self.STOP:
                        out.setdefault(bi, self.W_CJK)
        return out
```

**要点里写的是 `A.onPause`, 候选人说的是 `onPause`。** 不拆点分的话这是两个不同的 token, 覆盖率白掉一截。第 5 章 BM25 的 `_camel_parts` 解决的是同一类问题 (`onSaveInstanceState` 拆成 `on`/`save`/`instance`/`state`), 只是这里换成了按 `.` 拆。

**中文走二元组滑窗** 和第 5 章的 BM25 分词是同一套思路, 也同样不用 jieba。`STOP` 集合里的"调用"、"之前"、"发生"这些词在 Android 语料里到处都是, 留着会让任何答案都能蹭上覆盖率。

`out.setdefault` 而不是 `out[bi] =`: 同一个二元组重复出现时不覆盖, 保持第一次的权重。这里两者结果一样 (都是 `W_CJK`), 但习惯上**权重字典的写入用 setdefault 更安全** —— 将来如果给某些二元组特殊权重, 就不会被后面的普通写入覆盖掉。

RuleGrader 的 depth 是从分数反推的:

```python
        elif score >= 8.5:
            depth = Depth.EXPERT
        elif score >= 6.0:
            depth = Depth.SOLID
        elif score >= 2.0:
            depth = Depth.SURFACE
        else:
            depth = Depth.BLANK
```

**这和 LLMGrader 的方向正好相反** (那边 depth 是模型独立判断的, 和分数是两条信息)。规则判分器没有能力判断"讲清机制了吗", 只能用分数当代理。**这是它的根本局限, 也是为什么它只用于 CI。**

### 3.6 ✅ Checkpoint 4: 判分器双向自测

`grader.py` 底部的自测喂四种答案:

```python
    cases = {
        "满分答案": "跳转时 A.onPause 先于 B.onCreate 执行, 然后 A.onStop 发生在 B.onResume 之后, "
                "也就是 B 完全可见后 A 才停止。返回时 B.onPause 之后 A.onRestart onStart onResume, "
                "最后 B.onStop 和 B.onDestroy。",
        "部分答案": "先走 A 的 onPause, 然后 B 的 onCreate onStart onResume。",
        "空答案": "",
        "胡说答案": "这个跟 Flutter 的 Widget 树 diff 算法有关系, 需要看 Skia 渲染管线。",
    }
```

```bash
uv run python -m app.agent.grader
```

真实输出:

```
判分器双向自测 (满分必须高, 空/胡说必须低):
  满分答案   score=10.00 depth=expert  action=switch   命中3/3
  部分答案   score= 3.00 depth=surface action=followup 命中1/3
  空答案    score= 0.00 depth=blank   action=switch   命中0/3
  胡说答案   score= 0.00 depth=blank   action=switch   命中0/3
```

**"双向"是这个自测的关键词。** 只测"满分答案得高分"是不够的 —— 一个恒定返回 10 分的判分器也能过。**必须同时测"胡说答案得低分"**, 两个方向都对, 才说明它在判断而不是在瞎猜。

**"胡说答案"是刻意设计的**: 它满是技术术语 (Flutter、Widget、diff、Skia、渲染管线), 但和 Android 生命周期毫无关系。**这是在测"术语堆砌能不能骗过判分器"** —— 第 3 章的人工标注规范里也要求专门放这类陷阱样本。

**注意每一行的 `action` 都对上了 3.7 节的分支表**: 部分答案 (surface) 触发 `followup` 去追问, 胡说 (blank) 直接 `switch` 换题。

### 3.7 分支决策: 四条 `if`

```python
def _decide_next(
    grade_depth: Depth, wrong: list[str], followup_depth: int, max_depth: int, has_followup: bool
) -> NextAction:
    """分支决策放在代码里, 不交给模型。规则少而确定, 才调得动、测得了。"""
    if wrong:
        return NextAction.HINT           # 说错了要先纠正, 不能带着错认知往下走
    if grade_depth == Depth.BLANK:
        return NextAction.SWITCH         # 完全不会就别追了, 换个方向
    if followup_depth >= max_depth or not has_followup:
        return NextAction.SWITCH         # 追问深度到顶, 防止在一个点上钻死
    if grade_depth in (Depth.SURFACE, Depth.SOLID):
        return NextAction.FOLLOWUP       # 有基础但没到底, 正是该追的时候
    return NextAction.SWITCH             # expert 已经问到底了, 换题
```

**这九行就是 1.1 节"追问层次失控"的解法。** `followup_depth >= max_depth` 是硬上限, 默认 2 层。

**顺序有意义, 不能调换:**

- `wrong` 排第一, 因为**答错比答得深浅更紧急**。一个人讲得很深但有个关键点说反了, 必须先纠正。
- `BLANK` 排第二, 在追问判断之前。对完全不会的人追问是折磨, 而且拿不到任何信息。
- 深度上限排第三, 保证无论如何都不会追超过 `max_depth` 层。

**`纯函数` 是它最值得学的地方**: 五个入参, 一个返回值, 不读全局状态、不改任何东西。所以能像下面这样一次性枚举全部分支。

**✅ Checkpoint 5**: 六种情形一次看完。

```bash
uv run python -c "
from app.agent.grader import _decide_next
from app.agent.schemas import Depth
cases = [
    ('说错了 + solid',      Depth.SOLID,   ['把 onStop 说成 onPause'], 0, True),
    ('完全不会',            Depth.BLANK,   [], 0, True),
    ('surface + 有追问',    Depth.SURFACE, [], 0, True),
    ('solid + 追问到顶',    Depth.SOLID,   [], 2, True),
    ('solid + 题目没追问',  Depth.SOLID,   [], 0, False),
    ('expert',              Depth.EXPERT,  [], 0, True),
]
for name, d, wrong, fd, hasf in cases:
    print(f'  {name:20s} -> {_decide_next(d, wrong, fd, 2, hasf).value}')
"
```

真实输出:

```
  说错了 + solid          -> hint
  完全不会                 -> switch
  surface + 有追问        -> followup
  solid + 追问到顶         -> switch
  solid + 题目没追问        -> switch
  expert               -> switch
```

**第五行印证了第 3 章那句话**: "没有 followups 的题会怎样 —— 状态机发现追问池空了, 就直接换下一题, 不会崩。" 现在有实测了。

**只有一种情形会追问** (`surface`/`solid` + 有追问池 + 没到顶), 其余五种都换题或纠正。**这是有意的保守设计** —— 追问是好东西但成本高 (多一次 API 调用、多占一轮), 宁可少追。

---

## 四、选题器: 优先用人写的题

### 4.1 为什么优先黄金集

```python
"""出题: 优先用黄金集的题 (带人工 rubric), 不够了再让 LLM 基于检索到的语料生成。

为什么优先黄金集: 人工写的 rubric 质量远高于 LLM 生成的, 而 rubric 直接决定评分可信度。
LLM 生成只用于扩大覆盖面。
"""
```

**这条排序直接来自第 3 章踩坑 4: "不要让模型帮你写 rubric"。** 模型生成的要点看着齐整, 但它是从自己的知识里生成的, 不保证语料里有 —— 于是判分标准和知识库脱节。

所以架构是: **黄金集 15 题打头, 用完了才让 LLM 生成。** 而第 1 章把 `MAX_QUESTIONS` 默认设成 6, 意味着**正常一场面试根本用不到 LLM 生成** —— 生成分支是给"连续面试三场"这种场景兜底的。

### 4.2 三档优先级

```python
    def pick(
        self,
        asked_qids: Iterable[str],
        asked_topics: Iterable[str],
        difficulty: int,
        weak_topics: Iterable[str] = (),
    ) -> Question | None:
        asked_qids, asked_topics = set(asked_qids), set(asked_topics)
        weak = [t for t in weak_topics if t]

        pool = [q for q in self.golden if q.qid not in asked_qids]
        if pool:
            # 三档优先级: 错题本里的知识点 > 没考过的知识点 > 其他
            for cand in (
                [q for q in pool if q.topic in weak and q.difficulty <= difficulty],
                [q for q in pool if q.topic not in asked_topics and q.difficulty == difficulty],
                [q for q in pool if q.topic not in asked_topics],
                [q for q in pool if q.difficulty == difficulty],
                pool,
            ):
                if cand:
                    return self.rng.choice(cand)
```

**这个"元组套列表推导 + 第一个非空就用"的写法值得学。** 五档候选从严到宽排列, 最后一档是 `pool` 本身 (只要还有没问过的题就一定能选出来)。**等价的 `if/elif` 写法要 15 行, 而且加一档优先级要动三处。**

五档的顺序含义:

| 档 | 条件 | 为什么排这个位置 |
| --- | --- | --- |
| 1 | 错题本里的知识点, 且难度不超过当前 | 上次没答好的优先重考; 但**不要用更难的题去考弱项** |
| 2 | 没考过的知识点, 难度正好 | 常规情况走这一档 |
| 3 | 没考过的知识点 (难度不限) | 覆盖面优先于难度贴合 |
| 4 | 难度正好 (知识点可能重复) | 宁可重复知识点也别跳难度 |
| 5 | 剩下的任何题 | 兜底, 保证不返回 None |

**第 1 档 `q.difficulty <= difficulty` 那个 `<=` 是刻意的**: 重考弱项时用等难度或更简单的题。用更难的题考一个已知的弱项, 只会再得一个 0 分, 拿不到新信息。**评估的目的是定位边界, 不是反复确认失败。**

**第 3 档和第 4 档的先后是一个价值判断**: 覆盖面 (问到没问过的知识点) 比难度贴合更重要。因为报告要按 topic 分组给出薄弱项, 一个 topic 都没问到就没法给建议。

**`self.rng.choice(cand)` 而不是 `cand[0]`**: 随机选一个, 否则每场面试的题目顺序完全一样。`self.rng = random.Random(seed)` 是**独立的随机数实例**, 不用全局 `random` —— 这样测试传 `seed=42` 能稳定复现, 而且不会因为别的代码调了 `random.seed()` 而被干扰。

**✅ Checkpoint 6**: 四种情形选出四道不同的题。

```bash
uv run python -c "
from app.agent.selector import QuestionSelector
from app.rag.index import KnowledgeIndex
from app.rag.retriever import Retriever
s = QuestionSelector(Retriever(KnowledgeIndex()), 'eval/golden.yaml', seed=42)
print('无约束        ->', s.pick([], [], 2).qid)
print('错题本=Kotlin ->', s.pick([], [], 2, weak_topics=['Kotlin/协程']).qid)
print('难度3        ->', s.pick([], [], 3).qid)
asked = ['android-lifecycle-001','android-lifecycle-002','android-lifecycle-003']
q = s.pick(asked, ['四大组件/Activity生命周期'], 2)
print('已问过生命周期 ->', q.qid, '|', q.topic)
"
```

真实输出:

```
无约束        -> android-lifecycle-002
错题本=Kotlin -> android-kotlin-001
难度3        -> android-perf-002
已问过生命周期 -> android-jetpack-001 | Jetpack/ViewModel
```

四行分别验证了: 默认走第 2 档、错题本命中第 1 档、难度过滤生效、知识点去重生效。**第四行最关键** —— 三道生命周期题都问过之后, 它自动换到了另一个 topic, 而不是在同一个知识点上继续打转。这正是第 1 章列的"漂移 2: 重复问同一个知识点"被代码消灭的那一刻。

### 4.3 LLM 生成: 用完黄金集之后的兜底

```python
GEN_SYSTEM = """你是资深 Android 面试官。基于给定的参考资料出一道面试题。

硬性要求:
1. 题目必须能用参考资料回答, 不要问资料里没有的内容。
2. 必须给出 2-4 条评分要点 (rubric), 每条带权重, 权重合计正好 10。
3. 每条评分要点写成一句可判定的陈述, 不要写"理解了xx"这种没法判定的话。
4. 再给 1-2 个追问 (followup), 用来考察更深一层。

只输出 JSON, 不要任何解释文字:
{"topic": "...", "question": "...", "difficulty": 2,
 "rubric": [{"point": "...", "weight": 5}, {"point": "...", "weight": 5}],
 "followups": ["...", "..."]}"""
```

**四条硬性要求各对应一个失败模式**, 和第 3 章写判分提示词时是同一个套路:

| 要求 | 防的是什么 |
| --- | --- |
| 必须能用参考资料回答 | 模型凭自己的知识出题, 结果判分标准和知识库脱节 |
| 权重合计正好 10 | 分数没法跨题比较 (第 3 章的 `check_golden.py` 也检这一条) |
| 要点写成可判定的陈述 | "理解了生命周期"这种要点, 判分时没法判断命中与否 |
| 给出 followup | 没有 followup 时 `_decide_next` 只能 SWITCH, 追问分支形同虚设 |

```python
    def generate(self, topic: str, difficulty: int, asked_qids: Iterable[str]) -> Question | None:
        hits = self.retriever.search(topic, top_k=3, fetch_k=20, topic=topic, threshold=None)
        if not hits:
            return None
        ctx = build_context(hits, max_chars=1600)
        data = chat_json(
            [
                {"role": "system", "content": GEN_SYSTEM},
                {"role": "user", "content": f"知识点: {topic}\n难度: {difficulty}\n\n参考资料:\n{ctx}"},
            ],
            temperature=0.7,
        )
        slug = re.sub(r"[^a-z0-9]+", "-", topic.lower()).strip("-") or "gen"
        data["qid"] = f"gen-{slug}-{len(list(asked_qids)) + 1}"
        data["from_golden"] = False
        return Question.model_validate(data)
```

三个细节:

**`threshold=None`**: 出题时不设相似度阈值。第 5 章的检索默认会过滤低分结果, 但出题场景宁可拿到几条不太相关的资料, 也不要因为一条都没过阈值而直接返回 `None` —— 那样面试就断了。

**`temperature=0.7`**: 和判分的 `0.0` 形成对照。出题要多样性, 同一个 topic 连问两次不该出一样的题; 判分要确定性, 同一份答案两次必须同分。**温度是"每次调用"的参数, 不是全局配置** —— 这是把 LLM 调用收口在 `app/core/llm.py` 一个文件里带来的好处: 每个调用点自己决定温度。

**`qid = f"gen-{slug}-{n}"`**: 生成题的 qid 带 `gen-` 前缀。这样报告里、导出的微调样本里, 一眼能看出这道题是黄金集的还是生成的 —— 第 10 章筛微调数据时会用到这个前缀。`re.sub` 把 `"四大组件/Activity生命周期"` 这种带斜杠和中文的 topic 压成安全的 slug, `or "gen"` 兜住"整个 topic 被过滤空了"的情况 (纯中文 topic 就会这样)。

---

## 五、状态机本体: `app/agent/fsm.py`

### 5.1 Event: 状态机不 print, 只 yield

```python
@dataclass
class Event:
    """一次状态推进产生的对外事件。

    type: question | grade | followup | hint | report | done | error
    """
    type: str
    data: dict
```

**状态机不打印任何东西, 只产出 `Event`。** 这一条决定了同一个 FSM 能同时服务三个前端:

| 消费方 | 怎么用 Event |
| --- | --- |
| 第 7 章的 CLI | `render_event(e)` 打到终端 |
| 第 8 章的 Web | `yield f"data: {json.dumps(e.data)}\n\n"` 走 SSE |
| 第 8 章的测试 | `kinds = [e.type for e in fsm.submit(st, ans)]` 断言事件序列 |

如果 FSM 里写了 `print`, 上面第二和第三种用法都不成立。**"业务逻辑不做 IO"不是洁癖, 是复用的前提。**

`Event` 用 `@dataclass` 而不是 Pydantic: 它是纯内部结构, 不来自模型输出, 不需要校验。**该校验的地方 (模型输出) 用 Pydantic, 不该校验的地方别加负担。**

### 5.2 初始化: 参数从配置来, 但可以覆盖

```python
class InterviewFSM:
    def __init__(
        self,
        selector: QuestionSelector,
        grader: Grader,
        max_questions: int | None = None,
        max_followup_depth: int | None = None,
    ) -> None:
        self.selector = selector
        self.grader = grader
        self.max_questions = max_questions or settings.max_questions
        self.max_followup_depth = max_followup_depth or settings.max_followup_depth
```

`selector` 和 `grader` 是**构造函数注入**的, FSM 自己不 `import` 具体实现。所以测试可以塞一个 `_StubSelector` (按序发题, 不碰检索) 和 `RuleGrader` (不碰 LLM), 整套 FSM 测试跑完不需要网络、不需要 API Key、不需要向量库。这就是 5.8 节那 17 个测试能在 10 秒内跑完的原因。

`max_questions or settings.max_questions`: 默认从 `.env` 读, 显式传参覆盖。CLI 的 `--questions 3` 和测试的 `max_q=2` 都走这条路。

### 5.3 start 与 weak_topics

```python
    def start(self, difficulty: int = 2, weak_topics: Iterable[str] = ()) -> SessionState:
        return SessionState(
            session_id=uuid.uuid4().hex[:12],
            difficulty=difficulty,
            weak_topics=[t for t in weak_topics if t],
        )

    def weak_topics(self, st: SessionState) -> list[str]:
        """本场答得差的知识点 + 外部注入的历史薄弱点, 去重保序。"""
        cur = [t.question.topic for t in st.turns if t.grade and t.grade.scored and t.grade.score < 6.0]
        out: list[str] = []
        for t in [*cur, *st.weak_topics]:
            if t not in out:
                out.append(t)
        return out
```

**`weak_topics` 是一个分层的关键点。** FSM 需要知道"这个人哪些知识点弱", 但它**不知道数据库存在**。历史薄弱点由第 8 章的服务层从 SQLite 查出来, 通过 `start(weak_topics=[...])` 塞进 `SessionState`。

`cur` 排在 `st.weak_topics` 前面: **本场刚答错的比历史记录更重要**, 因为它更新。

去重用"遍历 + `not in out`"而不是 `set`: 需要保序。`set` 会把优先级顺序打乱, 而这个列表马上要喂给 `selector.pick()` 的第 1 档。

### 5.4 next_question 与 main_count

```python
    @staticmethod
    def main_count(st: SessionState) -> int:
        """已问的主问题数 —— 追问不占配额, 所以不能直接用 len(turns) 当题号。"""
        return sum(1 for t in st.turns if t.followup_depth == 0)
```

**这个 4 行的静态方法是整个 FSM 里最容易被忽略、又最容易出错的地方。** 如果用 `len(st.turns)` 当题号:

- 显示会错: 答得好触发两次追问, 第二道主问题会显示成"第 4 题"
- 结束条件会错: `max_questions=6` 的面试可能只问了 3 道主问题就结束了
- 而且**答得越好越早结束** —— 又是一个符号搞反的 bug

```python
    def next_question(self, st: SessionState) -> Event:
        if self.main_count(st) >= self.max_questions:
            return self.finish(st)
        st.stage = Stage.SELECT
        q = self.selector.pick(st.asked_qids, st.asked_topics, st.difficulty, self.weak_topics(st))
        if q is None:
            q = self.selector.generate(...)   # 黄金集用完才生成
        if q is None:
            return self.finish(st)
        st.asked_qids.append(q.qid)
        st.asked_topics.append(q.topic)
        st.turns.append(Turn(index=len(st.turns), question=q))
        st.stage = Stage.ASK
        return Event("question", {
            "index": self.main_count(st), "total": self.max_questions,
            "qid": q.qid, "topic": q.topic, "difficulty": q.difficulty, "question": q.question,
        })
```

注意 `q is None` 检查了两次: 选题器返回 None 才生成, 生成也返回 None (比如没配 API Key、或者检索一条都没命中) 就直接结束面试。**兜底链条要有终点**, 否则就是死循环。

### 5.5 submit: 一个 generator, 和一个不能省的 try/except

```python
    def submit(self, st: SessionState, answer: str) -> Iterator[Event]:
        turn = st.current
        if turn is None or st.finished:
            yield Event("error", {"message": "当前没有待回答的问题"})
            return

        st.stage = Stage.GRADE
        t0 = time.perf_counter()
        try:
            g = self.grader.grade(turn.question, answer, followup_depth=turn.followup_depth,
                                  max_followup_depth=self.max_followup_depth)
        except Exception as e:
            g = GradeResult(score=0.0, depth=Depth.SURFACE, next_action=NextAction.SWITCH,
                            comment=f"判分失败: {e}", scored=turn.followup_depth == 0)
            yield Event("error", {"message": f"判分失败, 本题记 0 分: {e}"})
        turn.answer = answer
        turn.grade = g
        turn.latency_ms = int((time.perf_counter() - t0) * 1000)
        yield Event("grade", {...})

        st.stage = Stage.BRANCH
        self._adapt_difficulty(st)
        if g.next_action in (NextAction.FOLLOWUP, NextAction.HINT):
            yield self._ask_deeper(st, turn, is_hint=g.next_action is NextAction.HINT)
            return
        ev = self.next_question(st)
        yield ev
```

**`submit` 返回 generator 而不是 list, 是为了第 8 章的 SSE。** 一次提交会产出 2 到 3 个事件 (grade → followup, 或 grade → question, 或 grade → report), Web 端要**边算边推**: 判分结果先显示出来, 下一道题稍后再来。如果返回 list, 用户要等所有事件都算完才能看到第一个。

**那个 `try/except Exception` 是这个文件里最重要的 8 行。** 判分是唯一调 LLM 的环节, 也就是唯一会超时、会限流、会返回垃圾的环节。没有这段兜底, 一次网络抖动就让整场面试崩掉, 用户前面答的 5 道题全丢。

有了它, 行为变成: **这一轮记 0 分, 附上错误信息, 流程继续往下走。** 注意兜底的 `GradeResult` 里 `next_action=SWITCH` —— 判分都失败了就别追问了, 换题。

`scored=turn.followup_depth == 0` 这一句容易漏: 兜底结果也要正确设置 `scored`, 否则一个失败的追问轮会被计入均分, 把分数拉低。

**`latency_ms` 在 except 之后才算**: 失败也要记耗时。第 9 章分析判分器性能时, "超时那次花了多久"恰恰是最有价值的数据点。

**✅ Checkpoint 7**: 判分器抛异常, 会话不崩。

```bash
uv run pytest tests/test_interview.py::test_grader_exception_does_not_kill_session -q
```

对应的测试是这样构造的 —— 一个 `grade()` 方法直接 `raise` 的假判分器:

```python
class Boom:
    def grade(self, *a, **k):
        raise RuntimeError("模拟 LLM 超时")

fsm = InterviewFSM(selector=_StubSelector([...]), grader=Boom(), max_questions=2)
st = fsm.start()
fsm.next_question(st)
kinds = [e.type for e in fsm.submit(st, "答案")]
assert "error" in kinds
# 关键点不是"会话是否结束", 而是"这一轮被记成了 0 分且流程还在往下走"
assert st.turns[0].grade is not None
assert st.turns[0].grade.score == 0.0
```

**最后两行断言的写法值得注意。** 一个直觉的断言是 `assert not st.finished` (会话没结束), 但那是脆的 —— `max_questions=2` 的面试第二轮就该结束了, 断言"没结束"会随参数变化而失败。真正要验证的不变量是: **异常被转成了一个 0 分的 turn, 数据结构完整。** 断言要盯不变量, 不要盯副作用。

### 5.6 _adapt_difficulty: 为什么要看"连续两次"

```python
    def _adapt_difficulty(self, st: SessionState) -> None:
        recent = [t for t in st.turns if t.grade and t.grade.scored][-2:]
        if len(recent) < 2:
            return
        depths = {t.grade.depth for t in recent if t.grade}
        if depths <= {Depth.SOLID, Depth.EXPERT} and st.difficulty < 3:
            st.difficulty += 1
        elif depths <= {Depth.BLANK, Depth.SURFACE} and st.difficulty > 1:
            st.difficulty -= 1
```

**`[-2:]` 取最近两道主问题, 而不是只看这一道。** 单轮调档的问题是抖动: 一道题恰好撞在熟悉的知识点上就升到难度 3, 下一题不熟又掉回来, 难度在 2 和 3 之间来回跳, 报告里的"水平判定"变成随机数。

**用集合的 `<=` 做子集判断**, 而不是 `all(...)` 循环: `depths <= {SOLID, EXPERT}` 读作"最近两轮的掌握程度都落在 solid 或 expert 里"。两行覆盖了升档和降档两种情况。

注意升降档都只走**一档**, 而且卡在 1~3 之间。难度只有三档, 一次跳两档等于直接从最简单到最难。

**✅ Checkpoint 8**: 连续 3 次满分 + 连续 3 次白卷, 观察难度曲线。

```bash
uv run python -c "
from app.agent.fsm import InterviewFSM
from app.agent.grader import RuleGrader
from app.agent.selector import load_golden_questions

class Stub:
    def __init__(s, qs): s.qs = qs
    def pick(s, asked, topics, diff, weak=()): 
        a = set(asked)
        return next((q for q in s.qs if q.qid not in a), None)

qs = load_golden_questions('eval/golden.yaml')
fsm = InterviewFSM(selector=Stub(qs), grader=RuleGrader(), max_questions=8, max_followup_depth=0)
st = fsm.start()
fsm.next_question(st)
for i in range(6):
    cur = st.current
    ans = '。'.join(p.point for p in cur.question.rubric) if i < 3 else ''
    before = st.difficulty
    for e in fsm.submit(st, ans): pass
    d = st.turns[i].grade.depth.value
    print(f'  第{i+1}轮 depth={d:<9s} 难度 {before} -> {st.difficulty}')
"
```

真实输出:

```
  第1轮 depth=expert   难度 2 -> 2
  第2轮 depth=expert   难度 2 -> 3
  第3轮 depth=expert   难度 3 -> 3
  第4轮 depth=blank    难度 3 -> 3
  第5轮 depth=blank    难度 3 -> 2
  第6轮 depth=blank    难度 2 -> 1
```

**逐行读这个输出, 能看出三件事:**

- 第 1 轮不动: `len(recent) < 2`, 只有一个数据点不调档
- 第 3 轮不动: 已经是难度 3, 撞到上限
- 第 4 轮不动: 最近两轮是 `{expert, blank}`, 既不是全强也不是全弱 —— **换手的第一轮是缓冲**, 这正是"连续两次"想要的效果

### 5.7 _ask_deeper: 追问和纠错走同一个函数

```python
    def _ask_deeper(self, st: SessionState, turn: Turn, is_hint: bool) -> Event:
        depth = turn.followup_depth + 1
        if is_hint:
            wrong = "; ".join(turn.grade.wrong_points) if turn.grade else ""
            text = f"先纠正一下: {wrong}。在这个前提下, 你再答一次这道题。"
        else:
            text = turn.grade.followup or "能再深入讲讲吗?"

        follow_q = turn.question.model_copy(update={
            "qid": f"{turn.question.qid}#f{depth}",
            "question": text,
            # 追问没有独立的 rubric, 所以不计分; 提示则保留原 rubric, 因为要重答同一道题
            "rubric": turn.question.rubric if is_hint else [],
            "followups": [],
        })
        st.turns.append(Turn(index=len(st.turns), question=follow_q,
                             followup_depth=depth, parent_qid=turn.question.qid))
        st.stage = Stage.ASK
        return Event("hint" if is_hint else "followup", {
            "qid": follow_q.qid, "question": text, "followup_depth": depth,
        })
```

**`rubric` 那一行是整个函数的核心区别:**

| | rubric | 计分吗 | 为什么 |
| --- | --- | --- | --- |
| followup (追问) | 空 | 不计分 | 问的是**新东西**, 原 rubric 不适用; 而追问本身没有标准答案 |
| hint (纠错重答) | 保留原题的 | 计分 | 问的还是**同一道题**, 判分标准当然一样 |

追问的 rubric 为空, 判分时 `total_weight` 会走 `or 1` 兜底, `_score_from_points` 算出 0 分 —— 所以 `scored=False` 必须设置, 这正是第 2.5 节 `scored` 字段存在的原因。链条是: **追问没有 rubric → 会得 0 分 → 所以要标记不计分。**

**`qid=f"{原qid}#f{depth}"`** 让追问轮的 id 可追溯: `android-lifecycle-003#f1` 一眼看出是哪道题的第 1 层追问。第 10 章导出微调数据时, 靠 `#f` 就能把追问样本和主问题样本分开。

**`followups=[]`** 清空: 否则追问轮自己又能触发追问, 层数虽然有 `max_followup_depth` 兜着, 但语义上第二层追问该由第一层追问的判分结果决定, 不该继承原题的 followup 列表。

**`model_copy(update=...)` 而不是 `Question(...)` 重新构造**: 原题的 `topic`、`difficulty`、`source_chunks` 全部自动带过来, 加字段时不用改这里。

**✅ Checkpoint 9**: 部分回答触发追问。

```bash
uv run python -c "
from app.agent.fsm import InterviewFSM
from app.agent.grader import RuleGrader
from app.agent.selector import QuestionSelector
from app.rag.index import KnowledgeIndex
from app.rag.retriever import Retriever
fsm = InterviewFSM(
    selector=QuestionSelector(Retriever(KnowledgeIndex()), 'eval/golden.yaml', seed=7),
    grader=RuleGrader(), max_questions=3, max_followup_depth=2)
st = fsm.start()
e = fsm.next_question(st)
print('Q1', e.type, '|', e.data['qid'], '|', e.data['question'][:30])
pt = st.current.question.rubric[0].point
print('输入(只讲要点1):', pt)
for ev in fsm.submit(st, pt):
    d = {k: ev.data[k] for k in ('score','depth','qid','question','followup_depth') if k in ev.data}
    print(' ->', ev.type, '|', d)
print('stage=', st.stage.value, 'difficulty=', st.difficulty, 'turns=', len(st.turns), 'main_count=', fsm.main_count(st))
"
```

真实输出:

```
Q1 question | android-lifecycle-003 | Fragment 里 observe LiveData 为什么要用
输入(只讲要点1): Fragment 本身生命周期比它的 View 长
 -> grade | {'score': 3.0, 'depth': 'surface'}
 -> followup | {'qid': 'android-lifecycle-003#f1', 'question': 'onDestroyView 和 onDestroy 为什么要分开?', 'followup_depth': 1}
stage= ask difficulty= 2 turns= 2 main_count= 1
```

**`turns=2` 但 `main_count=1`** —— 这一行就是 `main_count` 存在的全部理由: 数据结构里有两轮, 但配额只消耗了一道题。

`difficulty` 还是 2: 只有一个计分轮, `_adapt_difficulty` 直接 return 了。

**✅ Checkpoint 10**: 答错触发 hint, 且 rubric 被保留。

```bash
uv run python -c "
from app.agent.fsm import InterviewFSM
from app.agent.grader import RuleGrader
from app.agent.schemas import Depth, NextAction
from app.agent.selector import QuestionSelector
from app.rag.index import KnowledgeIndex
from app.rag.retriever import Retriever

class WrongGrader(RuleGrader):
    def grade(self, q, answer, followup_depth=0, max_followup_depth=2):
        g = super().grade(q, answer, followup_depth, max_followup_depth)
        g.wrong_points = ['把 onSaveInstanceState 说成在 onDestroy 之后调用']
        g.score, g.depth, g.next_action = 0.0, Depth.SURFACE, NextAction.HINT
        return g

fsm = InterviewFSM(
    selector=QuestionSelector(Retriever(KnowledgeIndex()), 'eval/golden.yaml', seed=3),
    grader=WrongGrader(), max_questions=3, max_followup_depth=2)
st = fsm.start()
e = fsm.next_question(st)
print('主问题:', e.data['qid'])
for ev in fsm.submit(st, '随便答一句'):
    if ev.type == 'grade': print('  grade:', {'score': ev.data['score'], 'wrong': ev.data['wrong']})
    if ev.type == 'hint':  print('  hint:', {k: ev.data[k] for k in ('qid','question','followup_depth')})
print('第二轮 rubric 条数 =', len(st.current.question.rubric), '(hint 保留原 rubric)')
print('主问题计数 =', fsm.main_count(st), '总轮数 =', len(st.turns))
"
```

真实输出:

```
主问题: android-jetpack-001
  grade: {'score': 0.0, 'wrong': ['把 onSaveInstanceState 说成在 onDestroy 之后调用']}
  hint: {'qid': 'android-jetpack-001#f1', 'question': '先纠正一下: 把 onSaveInstanceState 说成在 onDestroy 之后调用。在这个前提下, 你再答一次这道题。', 'followup_depth': 1}
第二轮 rubric 条数 = 3 (hint 保留原 rubric)
主问题计数 = 1 总轮数 = 2
```

**这个 Checkpoint 用了一个技巧: 继承 `RuleGrader` 覆盖 `grade` 的返回值。** 想验证 hint 分支, 就得让判分器报告"答错了"; 但离线规则判分器只会说"漏了", 不会说"错了" (它没有判错的能力, 见 3.5 节)。**与其等一个真的答错的样本, 不如构造一个** —— 这也是 5.5 节那个 `Boom` 判分器的同一套思路: **状态机的测试不该依赖判分器的能力。**

### 5.8 finish

```python
    def finish(self, st: SessionState) -> Event:
        st.stage = Stage.REPORT
        rep = build_report(st)
        st.finished = True
        st.stage = Stage.DONE
        return Event("report", rep.model_dump())
```

`st.finished = True` 放在 `build_report` **之后**: 报告生成失败时, 会话不该被标记成已完成, 否则数据既没报告又没法重试。**状态位在工作成功之后再翻转**, 这是一条通用规则。

---

## 六、报告: `app/agent/report.py` —— 刻意不用 LLM

### 6.1 为什么报告不让模型写

```python
LEVEL_RULES = [(8.0, "高级"), (6.0, "中级"), (4.0, "初级")]


def _level(avg: float) -> str:
    for lo, name in LEVEL_RULES:
        if avg >= lo:
            return name
    return "待加强"
```

**分数、薄弱知识点、漏掉的要点, 全都是已经算出来的确定数据。让模型再改写一遍, 唯一的效果是引入不一致** —— 它可能把 3.33 分说成"表现尚可", 可能漏掉一个薄弱项, 也可能凭空补一条你没测过的建议。

而且报告是**用户唯一会认真读的输出**。前面判分错一道题, 用户可能没注意; 报告里说错了水平, 用户立刻就知道这个产品不靠谱。**越靠近用户的环节, 越应该用确定性代码。**

`LEVEL_RULES` 写成"阈值降序的列表 + 第一个命中就返回", 是 4.2 节那个五档选题的同一个模式: 规则数据化, 加一档只改数据不改逻辑。

### 6.2 build_report: 计分只算主问题, 错题本收全部

```python
def build_report(st: SessionState) -> InterviewReport:
    by_topic: dict[str, list[Turn]] = {}
    for t in st.turns:
        if t.grade:
            by_topic.setdefault(t.question.topic, []).append(t)

    topics: list[TopicReport] = []
    for name, ts in by_topic.items():
        scored = [t for t in ts if t.grade and t.grade.scored]
        avg = round(sum(t.grade.score for t in scored) / len(scored), 2) if scored else 0.0
        gaps: list[str] = []
        for t in ts:                      # 注意: 这里遍历 ts, 不是 scored
            if t.grade:
                gaps.extend(t.grade.missed_points)
                gaps.extend(f"(答错) {w}" for w in t.grade.wrong_points)
        topics.append(TopicReport(topic=name, avg_score=avg, depth=_depth_of(avg),
                                  gaps=list(dict.fromkeys(gaps))[:5]))
    topics.sort(key=lambda x: x.avg_score)
```

**均分只算 `scored` 的轮, 但错题本 (`gaps`) 收所有 graded 的轮 —— 包括追问。** 这个不对称是刻意的:

- 追问答不上来**不该扣分** (它超出了原题的考察范围)
- 但追问答不上来**确实说明这里有知识缺口**, 该进错题本

`dict.fromkeys(gaps)` 去重保序 (Python 3.7+ 字典有序), 比 `set` 好在顺序稳定 —— 报告两次生成的文字要一样, 否则用户会以为系统在瞎输出。

`topics.sort(key=lambda x: x.avg_score)`: **最差的排最前面。** 报告是给人改进用的, 不是给人夸奖用的。

```python
    strengths = [t.topic for t in topics if t.avg_score >= 7.0]
    weaknesses = [t.topic for t in topics if t.avg_score < 6.0]

    plan: list[str] = []
    for t in topics:
        if t.avg_score < 7.0 and t.gaps:
            plan.extend(f"{t.topic}: {g}" for g in t.gaps[:2])
    if not plan:
        plan = ["整体掌握不错, 建议往源码和实际项目场景深入。"]
    return InterviewReport(..., study_plan=plan[:8])
```

**7.0 和 6.0 之间那个 1.0 分的空档是刻意留的。** 6.0~7.0 分的知识点既不算强项也不算薄弱项, 但**会进学习计划** (`< 7.0` 那个条件)。三条线各管一件事: 6.0 以下是要警告的, 7.0 以下是要练的, 7.0 以上是可以拿去说的。

`if not plan` 那个兜底不能省: 全部满分时 `plan` 是空的, 报告里出现一个空的"学习建议"区块很难看。**每个可能为空的列表都要想一遍空的时候长什么样。**

`plan[:8]`: 上限 8 条。给 20 条建议等于没给建议。

### 6.3 render_markdown

`build_report` 返回 Pydantic 对象, `render_markdown` 把它转成 Markdown 文本。**分成两个函数, 因为两个消费方要的东西不同**: CLI 要文本, 第 8 章的 Web API 要 JSON (`rep.model_dump()`)。如果只写一个返回 Markdown 的函数, Web 端就得去解析 Markdown。

**✅ Checkpoint 11**: 空会话也能出报告。

```bash
uv run pytest tests/test_interview.py::test_report_empty_session tests/test_interview.py::test_report_markdown_renders -q
```

对应的测试:

```python
def test_report_empty_session() -> None:
    """一题没答就结束也不能崩。"""
    r = build_report(SessionState(session_id="empty"))
    assert r.question_count == 0
    assert r.avg_score == 0.0
    assert render_markdown(r)
```

**"用户一题没答就关掉页面"是必然会发生的**, 而且第 8 章的 Web 端还会因为超时自动结束会话。这个测试守的就是那条路径。

---

## 七、CLI: `app/cli.py` —— 先在终端把链路跑通

### 7.1 为什么先写 CLI 而不是直接写 Web

| | CLI | Web |
| --- | --- | --- |
| 调一次要多久 | 改完直接跑 | 重启服务、刷页面、点按钮 |
| 出错时看到什么 | 完整堆栈 | 浏览器里一个 500 |
| 能不能自动跑完 | `--auto` 一条命令 | 要写前端脚本 |

**FSM 的所有逻辑在 CLI 上验证完, 第 8 章的 Web 层就只剩"把 Event 转成 SSE"这一件事。** 反过来, 先写 Web 的话, 每个 FSM 的 bug 都要穿过 HTTP、鉴权、序列化三层才能看到。

### 7.2 render_event: Event 到终端

```python
def render_event(e: Event) -> None:
    d = e.data
    if e.type == "question":
        print(f"\n{'=' * 68}")
        print(f"[第 {d['index']}/{d['total']} 题] {d['topic']} (难度 {d['difficulty']})")
        print(f"面试官: {d['question']}")
    elif e.type == "followup":
        print(f"\n面试官追问 (第 {d['followup_depth']} 层): {d['question']}")
    elif e.type == "hint":
        print(f"\n面试官提示: {d['question']}")
    elif e.type == "grade":
        print(f"\n  得分 {d['score']}/10  掌握程度 {d['depth']}  ({d['latency_ms']} ms)")
        if d["hit"]:    print(f"  讲到了: {'; '.join(d['hit'])}")
        if d["missed"]: print(f"  漏掉了: {'; '.join(d['missed'])}")
        if d["wrong"]:  print(f"  说错了: {'; '.join(d['wrong'])}")
        if d["comment"]: print(f"  点评: {d['comment']}")
    elif e.type == "error":
        print(f"\n  [错误] {d['message']}")
```

**所有 `print` 集中在这一个函数里。** 这是 5.1 节"FSM 只 yield 不 print"的另一半 —— 表现层和逻辑层各占一个文件, 换终端样式不碰 FSM, 改 FSM 不碰输出格式。

`d['depth']` 能直接拼进 f-string 而不是打印成 `<Depth.EXPERT: 'expert'>`, 靠的是 2.2 节 `class Depth(str, Enum)` 那个 `str` 继承。

四个 `if d[...]` 都做了空判断: 满分答案没有 `missed`, 空答案没有 `hit`, 打印空行只会让输出变脏。

### 7.3 fake_answer: 假答案要从 rubric 里拼

```python
def fake_answer(q: Question, turn_i: int) -> str:
    """自动模式的假答案: 好 / 中 / 空 三档轮换, 用来验证三条分支都走得通。

    好答案直接由本题的 rubric 拼出来 —— 这样不管抽到哪道题都能拿高分,
    否则固定文本碰上别的知识点会一律 0 分, 演示不出 followup 分支。
    """
    kind = turn_i % 3
    if kind == 0 and q.rubric:
        return "。".join(p.point for p in q.rubric) + "。"
    if kind == 1:
        return "大概知道一点, 好像和生命周期有关系。"
    return ""
```

**这 8 行踩过一个坑**: 最早的版本是三段写死的文本。结果选题器随机抽到 Kotlin 协程的题, 那段写死的"生命周期"答案一律 0 分, `--auto` 跑十次有九次全是 blank, 追问分支永远走不到。

**从本题自己的 rubric 拼答案, 就和抽到哪道题解耦了。** 这是"测试数据要由被测对象自己提供"的一个实例 —— 和 3.6 节判分器双向自测里 `"。".join(p.point for p in q.rubric)` 是同一个手法。

### 7.4 main: 参数与主循环

```python
def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--offline", action="store_true", help="规则判分, 不调 LLM")
    ap.add_argument("--auto", action="store_true", help="用内置假答案自动跑完, 不等输入")
    ap.add_argument("--questions", type=int, default=3)
    ap.add_argument("--difficulty", type=int, default=2, choices=[1, 2, 3])
    ap.add_argument("--seed", type=int, default=42)
    ap.add_argument("--golden", default="eval/golden.yaml")
    ap.add_argument("--dump", default=None, help="把整场轨迹存成 json, 给 eval_dialog.py 当输入")
    args = ap.parse_args()

    idx = KnowledgeIndex()
    if idx.count() == 0:
        print(f"知识库是空的, 先跑: python -m app.rag.index {settings.corpus_dir}")
        return
```

**那个 `idx.count() == 0` 的检查, 是把"最常见的新手错误"翻译成"能照着做的一句话"。** 没有它, 用户会看到一个空的检索结果、一个 `None` 的题目, 然后是"面试结束, 题数 0", 完全猜不到问题在哪。**能预料到的失败, 就该给出下一步命令。**

```python
    report: InterviewReport | None = None
    ev = fsm.next_question(st)
    render_event(ev)
    turn_i = 0

    while not st.finished:
        cur = st.current
        if args.auto:
            ans = fake_answer(cur.question, turn_i) if cur else ""
            print(f"\n你: {ans or '(交白卷)'}")
        else:
            try:
                ans = input("\n你: ").strip()
            except (EOFError, KeyboardInterrupt):
                print("\n(提前结束)")
                break
            if ans in ("quit", "exit", "q"):
                break
        turn_i += 1

        for e in fsm.submit(st, ans):
            if e.type == "report":
                report = InterviewReport.model_validate(e.data)
            else:
                render_event(e)

    if report is None:
        report = InterviewReport.model_validate(fsm.finish(st).data)
```

三处细节:

**`except (EOFError, KeyboardInterrupt)`**: 用户按 Ctrl+C 或 Ctrl+D 时, 不要抛一屏堆栈。捕获之后 `break` 出去, 后面的 `if report is None` 会补一份报告 —— **提前退出也能拿到当前进度的报告**, 这对"答了两道题就没耐心了"的用户很重要。

**`report` 从 Event 里取, 而不是自己调 `build_report`**: 报告在 FSM 里已经生成过一次了 (`finish()` 里), CLI 只是接住。如果 CLI 自己再算一遍, Web 端和 CLI 端就有两条生成路径, 迟早不一致。

**`InterviewReport.model_validate(e.data)` 又转回对象**: `Event.data` 是 dict, 但 `render_markdown` 要的是对象。这一步转换看着多余, 实际是**在 CLI 边界上再校验一次** —— 如果 FSM 哪天改了报告字段, 这里会立刻 `ValidationError`, 而不是在 Markdown 里渲染出一个 `None`。

**✅ Checkpoint 12**: 全离线跑完一场三题面试。

```bash
uv run python -m app.cli --offline --auto --questions 3 --seed 42
```

真实输出 (会话 id 每次不同):

```
面试开始 (会话 95fd72e6cf5e, 离线规则判分)

====================================================================
[第 1/3 题] 四大组件/Activity生命周期 (难度 2)
面试官: onSaveInstanceState 在什么时机被调用? 用户按返回键退出会调用吗?

你: 在 onStop 之前调用。只在系统主动销毁时调用, 用户主动按返回键退出不调用。恢复可在 onCreate 的 savedInstanceState 或 onRestoreInstanceState 读。

  得分 10.0/10  掌握程度 expert  (0 ms)
  讲到了: 在 onStop 之前调用; 只在系统主动销毁时调用, 用户主动按返回键退出不调用; 恢复可在 onCreate 的 savedInstanceState 或 onRestoreInstanceState 读
  点评: [规则判分] 命中 3/3 个要点

====================================================================
[第 2/3 题] Kotlin/协程 (难度 2)
面试官: 协程的挂起和线程的阻塞有什么本质区别?

你: 大概知道一点, 好像和生命周期有关系。

  得分 0.0/10  掌握程度 blank  (0 ms)
  漏掉了: 挂起不阻塞线程, 挂起时线程被释放去执行其他任务; 阻塞是线程占着资源什么都不做; 协程是线程之上的调度单位, 一个线程可承载大量协程
  点评: [规则判分] 命中 0/3 个要点

====================================================================
[第 3/3 题] Kotlin/协程 (难度 2)
面试官: 为什么说协程比线程省资源? 它省在哪?

你: (交白卷)

  得分 0.0/10  掌握程度 blank  (0 ms)
  漏掉了: 协程是线程之上的调度单位, 一个线程可承载大量协程; 挂起时把线程让出去, 而不是占着线程等待; 挂起恢复靠 CPS 变换出的状态机, 不需要额外线程栈
  点评: 没有作答
```

报告部分:

```
# 面试复盘报告

- 会话: `95fd72e6cf5e`
- 题数: 3
- 均分: **3.33 / 10**
- 水平判定: **待加强**
- 总耗时: 0 ms

## 分知识点表现

| 知识点 | 均分 | 掌握程度 |
|---|---|---|
| Kotlin/协程 | 0.0 | blank |
| 四大组件/Activity生命周期 | 10.0 | expert |

## 强项

- 四大组件/Activity生命周期 (均分 10.0)

## 薄弱项

- Kotlin/协程 (均分 0.0)

## 下一步学习建议

1. [Kotlin/协程] 补: 挂起不阻塞线程, 挂起时线程被释放去执行其他任务
2. [Kotlin/协程] 补: 阻塞是线程占着资源什么都不做
```

**这一屏输出值得逐项对照前面的设计:**

| 观察到的现象 | 对应哪一节的设计 |
| --- | --- |
| 三道题都是难度 2, 没有升降档 | 5.6: 最近两轮是 `{expert, blank}` 混合, 不调档 |
| 第 2 题答"大概知道一点"得 0 分而不是 3 分 | 3.5: RuleGrader 的关键词必须真的出现, 说套话没用 |
| 均分 3.33 = (10+0+0)/3 | 6.2: 只算 3 个 scored 轮 |
| topic 表里薄弱的排在前面 | 6.2: `topics.sort(key=lambda x: x.avg_score)` |
| 建议只有 2 条而不是 6 条 | 6.2: 每个 topic 最多取 2 条 gap |
| `0 ms` | 离线规则判分不调网络; 换成 LLM 判分这里是 2000~5000 ms |
| 一次追问都没触发 | 3.7: `blank` 直接 SWITCH, `expert` 也 SWITCH —— 三道题恰好都不在追问区间 |

**最后一行是个有意思的观察**: 三道题分别是满分和两次白卷, 都不触发追问。追问只在"答得半懂" (`surface`/`solid`) 时才有意义, 而 `--auto` 的三档轮换里只有第 2 档 (`turn_i % 3 == 1`) 有机会落进去 —— 这次它得了 0 分, 掉进了 `blank`。想稳定看到追问分支, 用 5.7 节 Checkpoint 9 那个只讲一个要点的构造。

### 7.5 --dump: 为第 9 章埋的钩子

```python
    if args.dump:
        # 轨迹落盘: 第 09 篇的 LLM Judge 要拿两次不同配置的轨迹做成对比较。
        # 存 model_dump 而不是自己拼字典 —— Pydantic 已经保证了结构一致。
        p = pathlib.Path(args.dump)
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(json.dumps({
            "session_id": st.session_id,
            "offline": args.offline,
            "model": "rule" if args.offline else settings.llm_model,
            "avg_score": report.avg_score,
            "level": report.level,
            "turns": [t.model_dump() for t in st.turns],
            "report": report.model_dump(),
        }, ensure_ascii=False, indent=2), encoding="utf-8")
        print(f"\n轨迹已存到 {p} ({len(st.turns)} 轮)")
```

**`--dump` 这 15 行现在看着没用, 第 9 章会用它做对话级评估。** 逻辑是: 用两种配置各跑一场面试, 把两条轨迹交给一个更强的模型做成对比较 ("哪一场的面试官表现更好"), 从而评估**整个 Agent** 而不只是判分器。

写进 JSON 的 `model` 字段是关键: 没有它, 两个 dump 文件放在一起就分不清哪个是哪个配置跑的。**产出评测数据时, 把配置和结果存在同一个文件里** —— 否则三天后你会有五个 json 文件和零个记忆。

`ensure_ascii=False`: 不加这个, 中文会存成 `中文`, 文件没法直接看。

`[t.model_dump() for t in st.turns]` 而不是手写字典: `Turn` 加字段时这里自动跟上。这也是 2.6 节说的"`Turn` 一份数据两用"—— 会话状态和评测样本是同一个结构。

---

## 八、测试: 17 个测试, 9.54 秒, 零网络

### 8.1 _StubSelector: 让 FSM 的测试和 RAG 解耦

```python
class _StubSelector:
    """假选题器: 不碰检索和 LLM, 只按序发题。让 FSM 的测试和 RAG 解耦。"""

    def __init__(self, questions: list[Question]):
        self.questions = questions

    def pick(self, asked_qids, asked_topics, difficulty, weak_topics=()):
        asked = set(asked_qids)
        for x in self.questions:
            if x.qid not in asked:
                return x
        return None
```

**11 行, 没有 `generate` 方法。** 因为测试给的题永远够用, 走不到生成分支 —— **假实现只需要实现被用到的部分**, 补齐整个接口是浪费。

```python
def _clone(q: Question, i: int) -> Question:
    return q.model_copy(update={"qid": f"{q.qid}-{i}", "topic": f"{q.topic}-{i}"})
```

`topic` 也改掉了: 否则 5 道题同一个 topic, 选题器的知识点去重会干扰测试。**造测试数据时, 每个会影响逻辑的字段都要造得不一样。**

### 8.2 三个最重要的测试

**`test_grader_two_way` —— 判分器可信的下限:**

```python
def test_grader_two_way(q: Question) -> None:
    """双向自测: 满分答案必须高分, 空答案和胡说必须低分。这是判分器可信的下限。"""
    g = RuleGrader()
    perfect = "。".join(p.point for p in q.rubric)
    assert g.grade(q, perfect).score >= 9.0
    assert g.grade(q, "").score == 0.0
    assert g.grade(q, "这是 Flutter 的 Skia 渲染管线问题, 要看 Widget diff。").score <= 2.0
```

第三行那个"胡说答案"是刻意选的: 它**技术词密度很高** (Flutter/Skia/Widget), 一个只数关键词个数的判分器会给它高分。**测边界要用"看着像对的错答案", 不是用 "asdfasdf"。**

**`test_score_monotonic` —— 分数必须有序:**

```python
def test_score_monotonic(q: Question) -> None:
    """答得越全分越高。顺序必须严格单调, 否则分数没有意义。"""
    g = RuleGrader()
    s1 = g.grade(q, q.rubric[0].point).score
    s2 = g.grade(q, f"{q.rubric[0].point}。{q.rubric[1].point}").score
    s3 = g.grade(q, "。".join(p.point for p in q.rubric)).score
    assert 0 < s1 < s2 < s3
```

**这个测试比"满分答案得 10 分"更有价值。** 分数的用途是排序和比较 (报告分档、topic 排序、第 9 章的顺序一致率), 只要单调性成立, 绝对值偏一点都不影响结论; 单调性一破, 后面所有对照实验都是噪声。

**`test_fsm_followup_not_counted` —— 追问不污染分数:**

```python
def test_fsm_followup_not_counted(q: Question) -> None:
    """部分回答会触发 followup, 但追问轮不能计入均分, 也不能占题目配额。"""
    fsm = _make_fsm([_clone(q, i) for i in range(5)], max_q=2)
    ...
    assert "followup" in kinds, "solid 水平应该触发追问"
    assert fsm.main_count(st) == 2
    scored = [t for t in st.turns if t.grade and t.grade.scored]
    assert len(scored) == 2, "只有主问题计分"
    assert len(st.turns) > 2, "追问轮确实存在"
```

**四条断言缺一不可, 尤其是最后一条。** 前三条在"追问功能完全失效"时也全部通过 —— 没有追问, `main_count` 当然是 2, `scored` 当然是 2。第四条 `len(st.turns) > 2` 才排除了这个假通过。**测"该发生的事发生了", 也要测"不该被影响的没被影响"。**

`assert` 后面都带了中文消息: 测试失败时终端直接告诉你哪条不变量破了, 不用回来读代码。

### 8.3 六个分支的枚举测试

```python
def test_wrong_points_trigger_hint(q):
    assert _decide_next(Depth.SOLID, ["A.onStop 在 B.onCreate 之前"], 0, 2, True) is NextAction.HINT

def test_followup_depth_capped(q):
    assert _decide_next(Depth.SOLID, [], 0, 2, True) is NextAction.FOLLOWUP
    assert _decide_next(Depth.SOLID, [], 2, 2, True) is NextAction.SWITCH
    assert _decide_next(Depth.SOLID, [], 0, 2, False) is NextAction.SWITCH

def test_blank_does_not_followup(q):
    assert _decide_next(Depth.BLANK, [], 0, 2, True) is NextAction.SWITCH
```

**5 个断言就把分支逻辑测全了, 因为 `_decide_next` 是纯函数** (3.7 节)。如果这段逻辑写在 `submit` 里面, 要测同样的 5 种情况就得构造 5 个完整的 FSM 会话、5 套假判分器。**把决策逻辑抽成纯函数, 测试成本降一个数量级。**

用 `is` 而不是 `==` 比较枚举: 枚举是单例, `is` 比较更严格, 而且能挡住"函数返回了字符串 `"hint"` 而不是 `NextAction.HINT`"这种错误 —— 因为 `NextAction.HINT == "hint"` 在 `str, Enum` 下是 True, `is` 才是 False。

### 8.4 Pydantic 兜底的测试

```python
def test_grade_result_tolerates_dirty_llm_output() -> None:
    assert GradeResult.model_validate({"score": 99}).score == 10.0
    assert GradeResult.model_validate({"score": -5}).score == 0.0
    assert GradeResult.model_validate({"score": "abc"}).score == 0.0
    assert GradeResult.model_validate({}).depth is Depth.SURFACE
```

**四行分别对应模型的四种真实脏输出**: 越界、负数、给了个字符串、整个字段没给。这个测试是 2.3 节三层防御的验收单。

**✅ Checkpoint 13**: 整章的测试全绿。

```bash
uv run pytest tests/test_interview.py
```

真实输出:

```
.................                                                        [100%]
17 passed in 9.54s
```

**9.54 秒里有 9 秒是 import 开销** (pydantic、pyyaml 加载)。17 个测试本身几乎不耗时 —— 因为它们不碰网络、不碰向量库、不碰数据库。这个数字是 5.2 节"依赖注入 + 假实现"的直接回报: **状态机的测试能跑得像单元测试一样快, 你才会真的在每次改动后跑它。**

---

## 九、踩坑与专家提示

**坑 1: 用 `len(turns)` 当题号, 结果答得越好面试越短。**
追问也是一个 turn。答得好触发两次追问, `max_questions=6` 的面试可能只问了 3 道主问题就结束。修法就是 5.4 节那个 4 行的 `main_count`。**这个 bug 的症状很隐蔽 —— 面试确实结束了, 报告也出来了, 只是题数不对。** 判断方法: 在 `--auto` 模式下把 `--questions` 设成 5, 数一下输出里 `[第 x/5 题]` 出现了几次。

**坑 2: 追问轮被计入均分, 分数被系统性拉低。**
追问没有独立 rubric, 判分必然 0 分。一道满分题加一次追问, 均分从 10.0 掉到 5.0。**更糟的是符号搞反了: 答得越好触发的追问越多, 分数越低。** 修法是 `scored: bool` 字段 (2.5 节)。定位这类 bug 的通用方法: **构造一个"应该拿满分"的输入, 看实际拿了多少。**

**坑 3: 让模型直接输出分数, 结果同一份答案两次差 2 分。**
模型对连续数值不稳定。改成让它输出**离散的掌握程度**和**命中了哪几条要点**, 分数由代码算 (2.2 + 3.1 节)。原则: **模型做分类和抽取, 代码做算术。**

**坑 4: 模型把 rubric 要点改写了一个字, 整条权重丢失。**
`hit_set` 是精确字符串匹配, `"A 要点"` 被改写成 `"A要点"` 就命中不了, 3 分变 0 分。**这个缺陷我们知道, 并且刻意没有加固** —— 因为第 9 章测出来的判分 MAE 0.833 已经**包含**了这个缺陷的影响。加固的三个选项 (返回下标 / 模糊匹配 / 归一化) 都会让"已经量过的数字"失效。**已经量过的缺陷, 比没量过的加固更让人安心。** 现在的做法是在提示词里写死"原样抄回, 不要改写一个字"。

**坑 5: `temperature` 写成全局配置。**
判分要 `0.0` (同一份答案两次必须同分), 出题要 `0.7` (同一个知识点两次不该出一样的题)。写成全局值, 两边必有一边是错的。修法: `chat()` 和 `chat_json()` 都接受 `temperature` 参数, 每个调用点自己决定。

**坑 6: 判分器超时, 整场面试崩掉。**
判分是唯一调 LLM 的环节, 也就是唯一会超时的环节。没有 `try/except`, 一次网络抖动让用户前面答的 5 道题全丢。修法是 5.5 节那 8 行兜底: **记 0 分, 附上错误信息, 流程继续。** 注意兜底的 `GradeResult` 里 `scored` 也要正确设置。

**坑 7: `--auto` 的假答案写死文本, 追问分支永远测不到。**
选题器随机抽题, 写死的"生命周期"答案碰上协程的题一律 0 分, 十次有九次全是 blank。修法是从本题自己的 rubric 拼答案 (7.3 节)。

**坑 8: 单轮调难度, 难度在 2 和 3 之间来回跳。**
一道题恰好撞在熟悉的知识点上就升档, 下一题不熟又掉回来。修法是看**最近两个计分轮** (`[-2:]`)。代价是换手时有一轮缓冲期, 这个代价值得付。

**坑 9: 报告让 LLM 生成, 结果 3.33 分被描述成"表现尚可"。**
分数、薄弱点、漏掉的要点全是已算出的确定数据, 让模型改写一遍只会引入不一致。而报告是用户唯一会认真读的输出。**越靠近用户的环节, 越应该用确定性代码。**

**坑 10: 状态机里写 `print`, Web 层没法复用。**
FSM 只 `yield Event`, 表现层 (`render_event` / SSE) 各自消费。"业务逻辑不做 IO"不是洁癖, 是同一个 FSM 能同时服务 CLI、Web、测试三个消费方的前提。

**坑 11: 先写 Web 再调 FSM, 每个 bug 都要穿三层才能看到。**
先把 CLI 跑通, 第 8 章的 Web 层就只剩"Event 转 SSE"一件事。**给自己造一个最短的反馈回路, 是开发效率里最划算的投资。**

**坑 12: 假选题器把整个接口都实现了。**
`_StubSelector` 只有 11 行、只有 `pick` 没有 `generate` —— 因为测试走不到生成分支。**假实现只需要实现被用到的部分。**

---

## 十、面试视角

**问 1: 你说你做了个面试 Agent, 它和"给 GPT 一段提示词让它扮演面试官"有什么区别?**

区别是**哪些事由代码保证, 哪些事交给模型**。纯提示词方案有三个必然的漂移: 忘记打分、重复问同一个知识点、追问没完没了。这三件事在我的实现里分别由 `GradeResult` 的必填结构、`selector` 的知识点去重、`max_followup_depth` 的硬上限保证 —— **它们不可能失败, 因为它们是 if 语句而不是请求。** 六个状态里只有 GRADE 一个调模型。

更关键的是第四点: **纯提示词方案无法调试。** 出了问题只能改提示词再试一次, 那不叫调试, 叫许愿。我的版本里 `_decide_next` 是个 5 参数的纯函数, 六个分支能在一条命令里全部枚举出来。

**问 2: 判分为什么不直接让模型给分数?**

模型对连续数值不稳定, 同一份答案两次可能差 2 分。我的做法是让它输出两个**离散**的东西 —— 命中了哪几条 rubric 要点 (抽取任务)、掌握程度落在四档里的哪一档 (分类任务) —— 然后分数由代码按权重算: `10 * 命中权重和 / 总权重 - 1.5 * 错误条数`。**模型做它擅长的分类和抽取, 算术交给代码。** 附带的好处是分数可解释: 用户能看到"你漏了这两条, 所以扣了 6 分"。

**问 3: 你怎么知道你的判分器是准的?**

三层。第一层是**双向自测**: 满分答案必须 ≥9 分, 空答案必须 0 分, 而且我用的"错答案"是一段技术词密度很高的胡说 (Flutter/Skia/Widget), 专门防住"只数关键词个数"的实现。第二层是**单调性**: 答一条要点 < 答两条 < 答三条, 严格递增 —— 这比绝对值更重要, 因为分数的用途是排序。第三层是第 9 章的**人工标注对照**: 105 个样本对上 MAE 0.833、顺序一致率 0.8、皮尔逊 0.9377。**说得出这三个数字, 比说"我们的判分很准"有用得多。**

**问 4: 追问的深度和分数怎么处理的? 这里有什么坑?**

追问不计入题目配额, 也不计入均分。前者靠 `main_count` 只数 `followup_depth == 0` 的轮; 后者靠 `GradeResult.scored` 标记。**没有第二条会有一个符号搞反的 bug: 答得越好触发的追问越多, 追问都是 0 分, 于是答得越好分数越低。**

还有一个细节: 追问和纠错走同一个函数, 区别只在 rubric —— 追问清空 rubric (问的是新东西, 不计分), 纠错保留原 rubric (问的还是同一道题, 要计分)。

**问 5: 这套东西怎么测? 状态机不是很难测吗?**

17 个测试, 9.54 秒跑完, 零网络零数据库。做到这一点靠两件事: **依赖注入** (selector 和 grader 从构造函数传入, 测试塞 `_StubSelector` 和 `RuleGrader`), 和**把决策逻辑抽成纯函数** (`_decide_next` 5 个参数无副作用, 六个分支用 5 个断言测全)。

测试里最值得说的一条是 `test_fsm_followup_not_counted` 的第四条断言 `len(st.turns) > 2`: 前三条断言在"追问功能完全失效"时也会通过, 第四条才排除了假通过。**测"该发生的发生了", 也要测"不该被影响的没被影响"。**

---

## 十一、划重点

1. **状态机 vs 提示词, 分界线是"哪些事不允许失败"。** 六个状态里只有 GRADE 调模型, 其余全是确定性代码。
2. **模型做分类和抽取, 代码做算术。** `Depth` 是四档离散值, 分数由 `10 * 命中权重 / 总权重` 算出来。
3. **Pydantic 是 LLM 输出的质检员, 三层防御**: 全字段有默认值 (`{}` 能构造) → `mode="before"` 的 validator 夹取值并吞掉类型错误 → 非法枚举**不容错**。容错的边界是: **你能不能合理推断出它本来想说什么。** 12.7 分显然是想给满分, "牛逼"推不出是哪一档。
4. **`class Depth(str, Enum)`** 让 `model_dump()` 出 `"solid"` 而不是 `<Depth.SOLID: 'solid'>`, 第 8 章的 SSE 不用写自定义编码器。
5. **`scored: bool` 是追问机制的地基。** 没有它, 答得越好分数越低。
6. **`main_count` 只数主问题。** 追问不占配额, 所以不能用 `len(turns)` 当题号。
7. **把决策逻辑抽成纯函数**, 测试成本降一个数量级。`_decide_next` 的六个分支 5 个断言测完。
8. **`_decide_next` 的判断顺序有含义**: 答错 (最紧急) → 白卷 (别硬钻) → 深度上限 (别钻死) → 追问。
9. **难度调整看"连续两次"**, 一次好运或一次失手不该改变难度。
10. **判分器的 `try/except` 是必需品**: 记 0 分继续, 不让一次网络抖动毁掉整场面试。
11. **FSM 只 `yield Event`, 不 `print`。** 同一个状态机因此能服务 CLI、SSE、测试三个消费方。
12. **报告刻意不用 LLM。** 越靠近用户的环节越要确定性。
13. **选题的五档优先级用"元组套列表推导 + 第一个非空就用"**, 等价 if/elif 要 15 行。第 1 档的 `<=` 是刻意的: **评估的目的是定位边界, 不是反复确认失败。**
14. **`temperature` 是每次调用的参数**: 判分 0.0, 出题 0.7。
15. **先写 CLI 再写 Web。** 给自己造最短的反馈回路。
16. **假实现只实现被用到的部分。** `_StubSelector` 11 行, 没有 `generate`。
17. **已经量过的缺陷, 比没量过的加固更让人安心。** `hit_set` 精确匹配的软肋留着没动, 因为第 9 章的 MAE 已经包含了它。

---

## 十二、下一章预告

现在整条链路在终端里跑通了: 选题、提问、判分、追问、调难度、出报告, 17 个测试守着。但它只能你自己在终端里用 —— 没有接口, 没有会话持久化, 关掉终端一切归零。

**第 8 章把它变成一个能被前端调用的服务**:

- FastAPI 的三个接口: 开始会话、提交答案、取报告
- **SSE 流式输出**: 判分结果先到, 下一道题稍后到 —— 这正是 `submit` 写成 generator 的兑现
- SQLite 落库: 会话、每一轮、报告都存下来, 于是"上次答错的知识点"能在下一场面试里被优先重考 (`weak_topics` 的注入方)
- API Key 鉴权 + 每日限额: 一个对外的接口不加这两样, 账单会教你做人
- `/healthz`: 一个接口告诉你知识库有几块、用的哪个模型、rerank 开没开

第 8 章结束时, 你会有一个 `curl` 就能面试的服务。







