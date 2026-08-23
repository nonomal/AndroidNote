# 14 · 实战手敲四:记忆小镇(多 NPC)

> 手敲系列第四篇,对应**项目三(赛博小镇)的核心机制自研版**。官方项目三的 Godot 游戏画面部分不适合手敲教学(游戏引擎是另一门手艺),但它真正的 Agent 精华——**NPC 人设、好感度系统、持久化记忆、NPC 之间的信息传播**——完全可以用 300 行 Python 在命令行里复刻出来。做完这篇再回第 07 篇看官方项目,你会发现 Godot 只是"皮",你敲的这些才是"魂"。
>
> 全部 5 个文件、约 300 行。预计 1~2 天。
>
> 核心新技能:**让 LLM 稳定扮演角色**、**用 LLM 做量化判断(情感裁判)**、**状态持久化**。

## 0. 成品长什么样

命令行运行 `python town.py`:

```
=== 欢迎来到赛博小镇(命令: /list /talk 名字 /gossip /status /exit) ===
[小镇] > /talk 老王
(你走向了老王)
[老王] > 你好呀,我叫小明!
老王: 哎哟小明来啦!快进来快进来,今天刚出炉的红豆包还热乎着呢!
[老王] > /exit
已存档,再见!
```

- 每个 NPC 有独立人设、独立好感度(0~100)、独立记忆
- 你说的话会被 LLM"情感裁判"评分,好感度实时增减,**说话语气跟着好感度变**
- 关键信息("我叫小明")会进 NPC 的长期记忆,`/exit` 存档、下次启动还记得
- `/gossip` 让两个 NPC 私下交换情报——你告诉老王的事,可能从小林嘴里说出来

## 1. 架构

```
town.py (主循环: 命令解析、存档读档)
   │
   ├── NPC 老王 ── persona + affinity + memories + dialog
   ├── NPC 小林 ──        (每个NPC独立一套状态)
   └── NPC 阿虎 ──
          │
          ├─ talk()            → 拼装: 人设+好感度语气+记忆+近期对话 → LLM
          ├─ _update_affinity() → LLM当情感裁判,只输出-5~5的整数
          ├─ _maybe_remember()  → 触发词命中则写入记忆
          └─ hear_gossip()      → 接收其他NPC传来的情报
```

| 文件 | 职责 |
| --- | --- |
| `requirements.txt` / `.env` | 依赖与密钥(本篇不需要 FastAPI) |
| `llm.py` | 只要 `chat()`(第 11 篇那版) |
| `npc.py` | NPC 类:对话、好感度、记忆 |
| `town.py` | 主循环 + 存档 |

## 2. 建项目(5 分钟)

PyCharm New Project → `memory-town`。`requirements.txt`:

```
openai>=1.40.0
python-dotenv>=1.0.0
```

`.env` 三件套同前。`llm.py` 照第 11 篇第 3 节敲一遍(只需要 `chat`,不需要 `chat_json`)。

## 3. npc.py —— 一个"活人"的全部代码(3 小时,本篇灵魂)

```python
"""NPC: 有人设、有好感度、有持久化记忆的小镇居民"""
import re

from llm import chat


class NPC:
    def __init__(self, name: str, persona: str):
        self.name = name
        self.persona = persona
        self.affinity = 50            # 好感度 0~100,初始中立
        self.memories: list[str] = []  # 长期记忆(存档保留)
        self.dialog: list[dict] = []   # 短期对话(本次运行有效)

    # ---------- 对话 ----------
    def _tone(self) -> str:
        """好感度 → 说话语气。这就是'数值驱动人设'——游戏行业的经典做法"""
        if self.affinity >= 80:
            return "你非常喜欢玩家,说话热情亲昵"
        if self.affinity >= 60:
            return "你对玩家有好感,说话友善"
        if self.affinity >= 40:
            return "你对玩家态度中立、礼貌"
        if self.affinity >= 20:
            return "你对玩家有些冷淡,回答简短"
        return "你讨厌玩家,说话带刺但不说脏话"

    def talk(self, player_input: str) -> str:
        memory_text = "\n".join(f"- {m}" for m in self.memories[-8:]) or "- (暂无)"
        messages = [{
            "role": "system",
            "content": (
                f"你在扮演小镇居民「{self.name}」。人设: {self.persona}\n"
                f"当前你对玩家的好感度是 {self.affinity}/100。{self._tone()}。\n"
                f"你记得的事情:\n{memory_text}\n"
                "用口语化中文回复,不超过3句话,永远不要跳出角色、不要提到自己是AI。"
            ),
        }]
        messages += self.dialog[-8:]   # 只带最近对话,防止上下文越滚越大
        messages.append({"role": "user", "content": player_input})

        reply = chat(messages, temperature=0.8)  # 高温度让角色更鲜活
        self.dialog += [
            {"role": "user", "content": player_input},
            {"role": "assistant", "content": reply},
        ]
        self._update_affinity(player_input)
        self._maybe_remember(player_input)
        return reply

    # ---------- 好感度: LLM当情感裁判 ----------
    def _update_affinity(self, player_input: str):
        """让模型只输出一个数。约束越死,输出越稳——这是'LLM做量化判断'的通用套路"""
        raw = chat([
            {"role": "system", "content": "你是情感裁判。评估玩家这句话对听者好感度的影响,只输出一个-5到5之间的整数,不要输出其他任何字符。"},
            {"role": "user", "content": player_input},
        ], temperature=0)
        m = re.search(r"-?\d+", raw)
        delta = max(-5, min(5, int(m.group()))) if m else 0  # 解析失败默认0,系统不能因裁判失灵而崩
        self.affinity = max(0, min(100, self.affinity + delta))

    # ---------- 记忆 ----------
    TRIGGERS = ("我叫", "我是", "我喜欢", "我讨厌", "告诉你", "记住", "秘密")

    def _maybe_remember(self, player_input: str):
        if any(t in player_input for t in self.TRIGGERS):
            self.memories.append(f"玩家说: {player_input}")

    def hear_gossip(self, from_npc: str, content: str):
        """从别的NPC那里听来的情报,也进长期记忆"""
        self.memories.append(f"听{from_npc}说: {content}")

    # ---------- 持久化 ----------
    def to_dict(self) -> dict:
        return {
            "name": self.name,
            "persona": self.persona,
            "affinity": self.affinity,
            "memories": self.memories,
        }

    @classmethod
    def from_dict(cls, d: dict) -> "NPC":
        npc = cls(d["name"], d["persona"])
        npc.affinity = d["affinity"]
        npc.memories = d["memories"]
        return npc


if __name__ == "__main__":
    npc = NPC("老王", "面包师,55岁,热心肠大嗓门,喜欢聊镇上八卦")
    print(npc.talk("你好!我叫小明,我给你带了礼物!"))
    print(f"好感度: {npc.affinity}, 记忆: {npc.memories}")
```

**✅ Checkpoint 1**:Run,老王热情回应,好感度大概率涨到 51~55 之间(送礼物是好话),记忆里出现"玩家说: ...我叫小明..."。

## 4. town.py —— 小镇主循环(1.5 小时)

```python
"""小镇主程序: 命令行交互 + 存档"""
import json
import random
from pathlib import Path

from npc import NPC

SAVE = Path("town_save.json")

DEFAULT_NPCS = [
    ("老王", "面包师,55岁,热心肠大嗓门,喜欢聊镇上八卦,面包手艺是祖传的"),
    ("小林", "图书管理员,26岁,安静博学,说话轻声细语,梦想写一本小说"),
    ("阿虎", "铁匠,38岁,沉默寡言,外冷内热,只对打铁和钓鱼有热情"),
]


def load_town() -> dict:
    if SAVE.exists():
        data = json.loads(SAVE.read_text(encoding="utf-8"))
        print(f"(读取存档: {len(data)}位居民)")
        return {d["name"]: NPC.from_dict(d) for d in data}
    return {name: NPC(name, persona) for name, persona in DEFAULT_NPCS}


def save_town(npcs: dict):
    SAVE.write_text(
        json.dumps([n.to_dict() for n in npcs.values()], ensure_ascii=False, indent=2),
        encoding="utf-8",
    )


def main():
    npcs = load_town()
    current = None
    print("=== 欢迎来到赛博小镇(命令: /list /talk 名字 /gossip /status /exit) ===")
    while True:
        prompt_name = current.name if current else "小镇"
        try:
            line = input(f"[{prompt_name}] > ").strip()
        except (EOFError, KeyboardInterrupt):
            line = "/exit"
        if not line:
            continue

        if line == "/exit":
            save_town(npcs)
            print("已存档,再见!")
            break
        elif line == "/list":
            for n in npcs.values():
                print(f"  {n.name}: {n.persona[:18]}…")
        elif line == "/status":
            for n in npcs.values():
                print(f"  {n.name} 好感度{n.affinity} 记忆{len(n.memories)}条")
        elif line.startswith("/talk"):
            name = line.replace("/talk", "").strip()
            if name in npcs:
                current = npcs[name]
                print(f"(你走向了{name})")
            else:
                print(f"小镇上没有叫「{name}」的人,用 /list 看看都有谁")
        elif line == "/gossip":
            a, b = random.sample(list(npcs.values()), 2)
            if a.memories:
                b.hear_gossip(a.name, a.memories[-1])
                print(f"({a.name}悄悄把一件事告诉了{b.name})")
            else:
                print(f"({a.name}最近没什么新鲜事可讲)")
        elif current is None:
            print("先用 /talk 名字 选择一位居民再说话")
        else:
            print(f"{current.name}: {current.talk(line)}")


if __name__ == "__main__":
    main()
```

**✅ 最终验收**(照着这个剧本玩一遍,全过 = 通关):

- [ ] `/talk 老王` → "我叫小明,告诉你个秘密,我打算送小林一本书" → 老王回应,`/status` 显示老王记忆 +1
- [ ] 连续夸老王三句 → 好感度涨;骂他一句 → 好感度跌,**下一句回复的语气明显变冷**
- [ ] `/gossip` 多敲几次,直到提示"老王悄悄把一件事告诉了小林"
- [ ] `/talk 小林` → "你听说什么了吗?" → 小林可能把送书的事说出来(信息在 NPC 间传播了!)
- [ ] `/exit` 后重新运行 `python town.py` → `/status` 里记忆和好感度都还在(持久化生效)
- [ ] 项目目录里打开 `town_save.json`,亲眼看看"NPC 的灵魂"长什么样

## 5. 为什么这 300 行是官方项目三的"魂"

对照第 07 篇官方项目,一一对应:

| 你手敲的 | 官方项目三 | 差异 |
| --- | --- | --- |
| `_tone()` 数值驱动语气 | 好感度系统 | 官方分了更多档位、影响更多行为 |
| `memories` 列表 + 触发词 | MemoryManager | 官方用 embedding 语义检索 + SQLite,你用的是关键词 + JSON |
| `hear_gossip` | NPC 社交系统 | 官方有定时任务驱动 NPC 自主互动 |
| `town_save.json` | 数据库持久化 | 原理相同:状态 → 序列化 → 落盘 |
| 命令行界面 | Godot 游戏画面 | 纯表现层差异,Agent 逻辑无关 |

**进阶练习**:

1. 把记忆检索升级成第 11 篇 `memory.py` 的字符重合度排序(而不是只取最近 8 条)
2. 加"NPC 自主行动":每输入 5 次,随机让一个 NPC 主动跟你搭话,内容基于它的记忆
3. 给 `_update_affinity` 加缓存:同样的话 10 分钟内不重复请求裁判(省一半 token 钱)

## 6. 面试怎么讲

"我实现了一个多 NPC 记忆系统:每个 NPC 独立维护人设、好感度和长期记忆;好感度用 LLM 做量化情感裁判(强约束输出一个整数,解析失败有默认值兜底),数值再反向驱动角色语气;NPC 间有信息传播机制;全部状态可序列化存档。这套机制和商业 AI 游戏的 NPC 系统同构。"

下一篇:[15 · 模型微调与私有化部署](./15-模型微调与私有化部署.md) | 手敲终点站:[18 · 构建你自己的Agent框架](./18-终极手敲-构建你自己的Agent框架.md) | 返回目录:[README](./README.md)
