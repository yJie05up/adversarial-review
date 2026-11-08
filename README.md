# adversarial-review

> The author cannot audit their own work. This is not a matter of skill — it is a matter of position.
> An author knows what they *meant* to build, so they cannot see what they *actually* built.
> **The only fix is to change position.**

A skill that dispatches an independent subagent to tear apart a finished artifact — code, document, plan, analysis, system, copy, data — and returns a **verdict**, not an opinion.

<p>
  <a href="#english"><b>English</b></a> · <a href="#chinese"><b>简体中文</b></a>
</p>

<p>
  <img alt="license" src="https://img.shields.io/badge/license-MIT-2ea44f">
  <img alt="format" src="https://img.shields.io/badge/format-SKILL.md-1f6feb">
  <img alt="invocation" src="https://img.shields.io/badge/invocation-%2Fadversarial--review-d97706">
</p>

---

<a id="english"></a>

## English

### The problem it solves

Ask a model to check its own work and you get a review that was written by the party under review. It is not dishonest — it is structurally blind. The same context that decided *"this is done"* is the context that decides *"this is correct."*

Checklists do not fix this either. A checklist of 40 items gets skimmed, and the item nobody thought to write is the one that bites.

This skill fixes it in two ways:

1. **Change the reviewer, not the rubric.** A fresh subagent with no access to the author's reasoning gets the raw requirement, the artifact, and the author's claim list. Nothing else. Forking the author's context is explicitly forbidden — a forked context inherits *"I already finished it"* and the adversarial edge drops to zero.
2. **Close the review space.** The critique axes are not a list to be extended; they are a set with a proof of exhaustiveness.

### What you get

```
VERDICT: BLOCK
CONFIDENCE: high
BLOCKING: 2 | DEBT: 1 | WITHDRAWN: 1

[BLOCKING]
B1 · D3 Silent Failure · src/export.ts:88
  Claim:    CSV export handles names that contain commas.
  Evidence: fields.join(',') — no quoting, no escaping.
  Trigger:  name = "Kim, Jae-won" → column shift → the row parses with the wrong
            owner, and nothing throws, logs, or turns red.
  Cross-exam: Q: is there a downstream validator that rejects shifted rows?
              → name it, and this drops to DEBT.
              → otherwise it stays BLOCK.
  Fix:      quote fields per RFC 4180 ("…" with inner quotes doubled).

[DEBT]
T1 · D5 Unclaimed Value · export formatter
  Cost: ~30 lines. Value: the same fix also unbreaks the import path.

[WITHDRAWN]
W1 · originally flagged "no progress bar" · withdrawn: the export runs in <50 ms on
     the reference dataset. The value is not there, so the finding was noise.

[Only fix one thing] B1 — silent data corruption outranks everything else here.
```

The `[WITHDRAWN]` block is not politeness. It is an anti-forgery device: a review that cannot withdraw a single finding was not aiming, it was spraying.

### Quick start

```
/adversarial-review  review the API client you just finished
```

That is the only entry point. The skill ships with `disable-model-invocation: true`, which removes it from the model's skill catalog entirely — the model cannot see it, so it cannot quietly stamp its own work as done. Only a human hand at the `/` menu can open this gate.

If you point it at nothing, it reviews the most recent delivery and labels the report:

```
SCOPE: <what it identified> (inferred — not specified)
```

### What happens under the hood

| Step | Name | What actually happens |
|---|---|---|
| 1 | **Docket** | The delivery is compressed into falsifiable claims, each with an evidence pointer (file:line, command output, UI element) or the word `none`. The user's original request is copied **verbatim** — it is the only ammunition D2 will ever get. |
| 2 | **Dispatch** | A fresh `subagent` is briefed. The author's justifications, design intent, and difficulties are forbidden from the brief — they are contamination. |
| 3 | **Adjudicate** | Spot-check ≥2 findings against reality. Forge one evidence pointer and the whole batch is void. Then grade: BLOCK / PATCH / DEBT / PASS. The reviewer proposes; the gate decides. |
| 4 | **Land** | Every finding resolves to exactly one of `FIX` (with location or diff), `ACCEPT-DEBT` (with residual risk and who carries it), or `REBUT` (with evidence, not with "I feel"). A fourth option does not exist. |

### The five axes — and why the set is closed

Three review surfaces (completeness, latent risk, headroom) collapse into five atomic axes. A finding may land on exactly one; nothing else is allowed.

| Axis | The single question | Evidence shape |
|---|---|---|
| **D1 Missing Claim** | Where is the evidence for what you claim you finished? | claim → artifact location |
| **D2 Missing Expectation** | What would a reasonable user expect that nobody ever mentioned? | line-by-line reconciliation against the original request |
| **D3 Silent Failure** | Under what input or state does it return a wrong answer *without announcing it*? | trigger condition + wrong result + why nobody notices |
| **D4 Substitution Doubt** | Is there an equivalent path that is simpler, more standard, or cheaper? | executable alternative + what it saves + what it costs |
| **D5 Unclaimed Value** | Given the design is sound, what cheap value is being left on the table? | concrete delta + cost order of magnitude |

**Exhaustiveness.** Take anything that exists. Every defensible criticism of it falls into exactly four families:

- it is **missing** something it promised (D1) or something it owed (D2)
- it is **wrong** in a way that does not announce itself (D3)
- it is **excessive** — some part could be replaced by something cheaper (D4)
- it is **incomplete** — the design works, but leaves value unclaimed (D5)

Missing, wrong, excessive, incomplete. There is no fifth family. When something genuinely will not fit, the skill reports `AXIS-MISS` and proposes a candidate axis rather than quietly widening the definition — unbounded axes are how a review becomes a vibe.

### Socratic — but closed

Open questions make a review feel rigorous and change nothing. "Have you considered…?" invites a paragraph of prose back and the finding dissolves.

So every finding carries **one closed question with two branches**, written into the report:

> Q: *is there a downstream validator that rejects shifted rows?*
> → **Name it** and this drops to DEBT.
> → **Cannot name it** and BLOCK stands.

A question you can answer out of is a real question. A question you can answer *around* is decoration. Depth is capped at two rounds: a cross-examination of a cross-examination must resolve into a verdict, or that axis is abandoned for this review.

### Anti-forgery devices

| Device | Why it exists |
|---|---|
| `WITHDRAWN` must be non-empty | Withdrawing nothing means the review was a shotgun, not a rifle |
| Zero findings requires a list of what was actually inspected | Inventing defects is worse than missing them — but claiming a clean sweep you never ran is worse still |
| `PASS` is only legal with a non-empty `WITHDRAWN` | An all-green report is, ~99% of the time, a report that did not happen |
| `REBUT` requires evidence | Disagreement is allowed. Disagreement by assertion is not |
| `CONFIDENCE: low` caps the verdict at DEBT | Hunches may be recorded. Hunches may not block a release |

### Files

```
adversarial-review/
├── SKILL.md                    gate · iron rules · four steps · five axes · output format · anti-patterns
└── references/
    ├── audit-axes.md           per-axis: the question, evidence shape, escalation threshold, boundary exclusions
    ├── interrogation.md        closed cross-examination templates: two-branch resolution, two-level cap
    └── review-brief.md         dispatch brief skeleton: role, inputs, prohibitions, output contract
```

### Install

Copy the directory into whichever skill root your harness reads. No dependencies, no scripts, no network calls.

| Harness | Path |
|---|---|
| Claude Code | `~/.claude/skills/adversarial-review/` |
| Codex | `~/.codex/skills/adversarial-review/` (or `~/.agents/skills/` if symlinked) |
| DeepSeek Harness | `~/.dsh/skills/adversarial-review/` |

Invoke with `/adversarial-review`.

### Tuning

- **Want it automatic?** Delete `disable-model-invocation: true` from the frontmatter. It becomes model-invocable again — which is exactly the failure mode this skill was built to prevent. Decide deliberately.
- **Too noisy?** Raise the BLOCK threshold: require two independent D3 findings instead of one.
- **Reviews feel soft?** The failure is usually in the brief, not the reviewer. Check that the author's justifications leaked into it.

### Limits

- Needs subagent support. Without it the skill degrades to inline self-review, caps the verdict at PATCH, and marks the report `SELF-REVIEW (degraded)` — an honest downgrade rather than a fake pass.
- It judges an artifact against its own stated goal. A well-executed solution to the wrong problem will pass D1–D4 and only surface under D5.
- It has no taste. It will not tell you the prose is boring or the design is ugly, only that something is unsupported, silent, redundant, or unclaimed.

### License

MIT © YJie

---

<a id="chinese"></a>

## 简体中文

### 它解决什么问题

让模型检查自己的作品，得到的是一份由被审查方撰写的审查报告。这不涉及诚实与否，而是位置问题：决定「这个做完了」的那段上下文，和决定「这个是对的」的那段上下文，是同一段。

列清单同样无效。四十条清单会被扫读，而真正咬人的那一条，恰恰是没人想到要写上去的。

这个 skill 从两处入手：

1. **换掉审查者，而不是换掉标准。** 一个全新上下文的 subagent，拿到的只有：原始诉求、交付物、作者的断言清单。别的都不给。明确禁止 fork 作者的上下文——fork 会继承「我已经做完了」的叙事，对抗力度归零。
2. **把审查空间关起来。** 批评的轴不是一张可以无限追加的清单，而是一个带穷尽性证明的集合。

### 它会产出什么

```
VERDICT: BLOCK
CONFIDENCE: high
BLOCKING: 2 | DEBT: 1 | WITHDRAWN: 1

[BLOCKING]
B1 · D3 失效之径 · src/export.ts:88
  断言: CSV 导出能处理含逗号的姓名。
  证据: fields.join(',') —— 没有引号包裹，没有转义。
  触发: name = "Kim, Jae-won" → 列错位 → 该行按错误的归属人被解析，
        不抛异常、不打日志、不变红。
  反诘: Q: 有没有一个下游校验器会拒绝错位的行？
        → 说得出名字，降为 DEBT。
        → 说不出，维持 BLOCK。
  修法: 按 RFC 4180 给字段加引号（内部引号双写）。

[DEBT]
T1 · D5 未取之值 · 导出格式化器
  成本: 约 30 行。价值: 同一个修复顺带让导入路径也不再出错。

[WITHDRAWN]
W1 · 原判「缺少进度条」· 撤回理由: 参考数据集上导出耗时 <50 ms，价值不存在，
     这条属于乱开枪。

[只修一件事] B1 —— 静默的数据损坏，优先级高于清单上其余所有条目。
```

`[WITHDRAWN]` 不是谦辞，是防伪装置：一条都撤不回的审查，说明它在乱开枪。

### 怎么用

```
/adversarial-review  审一下你刚写完的 API 客户端
```

**这是唯一入口。** frontmatter 里 `disable-model-invocation: true`，这个 skill 从模型的技能清单里被整体剔除——模型看不见它，就不会在写「已完成」的时候顺手给自己盖个章。这扇门只能由人的手在 `/` 菜单里推开。

不指定对象时，默认审最近一次交付，并在报告头部标注：

```
SCOPE: <识别到的对象> (inferred — 未显式指定)
```

### 内部发生了什么

| 步骤 | 名字 | 具体动作 |
|---|---|---|
| 1 | **立卷** | 把交付物压成可证伪的断言清单，每条挂一个证据落点（文件:行 / 命令回显 / 页面元素），落不了的写「无」。同时**逐字**抄录用户原始诉求——那是 D2 唯一的弹药。 |
| 2 | **派单** | 起一个全新上下文的 `subagent`。brief 里禁止出现作者的辩护、设计意图、难处——那是污染源。 |
| 3 | **裁决** | 抽查 ≥2 条 finding 的证据落点是否真的存在。捏造一条，整批作废。然后定级 BLOCK / PATCH / DEBT / PASS。reviewer 只负责提议，过闸的是你。 |
| 4 | **落地** | 每条 finding 必须落到 `FIX`（附位置或 diff）、`ACCEPT-DEBT`（写清残余风险与承担者）、`REBUT`（附证据，不是「我觉得」）三者之一。第四种选项不存在。 |

### 五个轴，以及为什么这个集合是闭合的

三个审查面（完成度 / 隐患 / 优化空间）拆成五个原子轴。一条 finding 只能落在其中一个轴上。

| 轴 | 唯一判定问句 | 证据形态 |
|---|---|---|
| **D1 断言之缺** | 作者声称完成的事，证据在哪？ | 断言 → 产物位置的映射 |
| **D2 期待之缺** | 用户没明说、但按常识理应期待的东西，在哪？ | 原始诉求逐条对账 |
| **D3 失效之径** | 在什么输入或状态下，它会给出错误结果且不自曝？ | 触发条件 + 错误结果 + 为何无人察觉 |
| **D4 替代之疑** | 有没有等价但更简单、更标准、更便宜的做法？ | 可执行替代 + 省掉什么 + 它的代价 |
| **D5 未取之值** | 方案成立的前提下，有哪些低代价价值被漏掉？ | 具体增量 + 成本量级 |

**穷尽性。** 拿任何一个已经存在的东西来看，对它的全部合理批评只落在四类里：

- **缺** —— 它承诺过的（D1）、或它本就欠着的（D2）
- **错** —— 而且是不会自曝的那种错（D3）
- **冗** —— 某一部分可以被更便宜的东西替换（D4）
- **增** —— 设计成立，但有价值没被取走（D5）

缺、错、冗、增。没有第五类。真遇到装不下的东西，报 `AXIS-MISS` 并给出新轴候选，而不是偷偷把定义改宽——无边界正是审查退化成「感觉」的方式。

### 苏格拉底式，但是闭合的

开放式提问让审查显得严谨，却不改变任何东西。「你有考虑过……吗？」换来一段散文式的解释，finding 随即溶解。

所以每条 finding 都挂**一个闭合问句、两条分支**，直接写进报告：

> Q: *有没有一个下游校验器会拒绝错位的行？*
> → **说得出名字**，降为 DEBT。
> → **说不出**，BLOCK 维持。

能答出去的问题才是真问题；能绕着答的问题只是装饰。层级上限两层：反诘的反诘必须落地为判决，否则该轴本次作废。

### 防伪装置

| 装置 | 为什么要有 |
|---|---|
| `WITHDRAWN` 必须非空 | 什么都撤不回，说明这是散弹枪不是步枪 |
| 零 finding 必须列出实际检查的位置 | 编造缺陷比漏报更严重；但声称扫过却没扫，更严重 |
| `PASS` 只有 `WITHDRAWN` 非空时才合法 | 全绿的审查报告，99% 是没审 |
| `REBUT` 必须附证据 | 允许不同意。不允许靠断言不同意 |
| `CONFIDENCE: low` 时判决上限压到 DEBT | 直觉可以记录，直觉不能拦发布 |

### 文件结构

```
adversarial-review/
├── SKILL.md                    门禁 · 铁律 · 四步 · 五轴 · 输出格式 · 反模式
└── references/
    ├── audit-axes.md           逐轴：判定问句 / 证据形态 / 升级门槛 / 边界排除
    ├── interrogation.md        闭合反诘模板：两分支收束、两层上限
    └── review-brief.md         派单 brief 骨架：角色 / 输入 / 禁止项 / 输出契约
```

### 安装

把目录复制到你所用的 harness 读取的 skill 根目录下即可。无依赖、无脚本、无网络调用。

| 运行时 | 路径 |
|---|---|
| Claude Code | `~/.claude/skills/adversarial-review/` |
| Codex | `~/.codex/skills/adversarial-review/`（或用软链指向的 `~/.agents/skills/`） |
| DeepSeek Harness | `~/.dsh/skills/adversarial-review/` |

调用方式：`/adversarial-review`。

### 调参

- **想让它自动生效？** 删掉 frontmatter 里的 `disable-model-invocation: true`，它就重新变成模型可调用的东西——而「模型顺手给自己盖章」正是这个 skill 要防的东西。请有意为之。
- **太吵？** 抬高 BLOCK 门槛：要求两条独立的 D3，而不是一条。
- **审得太软？** 问题通常出在 brief 而不是 reviewer。检查作者的辩护有没有漏进去。

### 边界

- 需要 subagent 支持。没有时降级为内联自审，判决上限压到 PATCH，并在报告首行标 `SELF-REVIEW (degraded)`——诚实的降级，而不是假装的通过。
- 它拿交付物对照它自己声明的目标。一个执行良好但方向错误的东西会通过 D1–D4，只在 D5 露头。
- 它没有品味。它不会说这段文字无聊、这个设计难看；它只会说某处无凭、某处静默、某处可替、某处可惜。

### 许可

MIT © YJie

---

<p align="center"><sub>Built for the moment you are about to say "done".</sub></p>
