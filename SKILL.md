---
name: adversarial-review
description: "找茬者交付门禁 / Adversarial delivery gate. Dispatch an independent subagent to review a finished artifact from the opposing side — adversarial probing plus closed Socratic cross-examination — and return a verdict-bearing defect list (BLOCK / PATCH / DEBT). Works on code, documents, plans, analyses, systems, copy, and data. 派独立 subagent 以对立方立场审查任意已完成产出，输出可判决的缺陷清单。"
disable-model-invocation: true
---

# 找茬者门禁 · Adversarial Review

> 作者无法审查自己的作品。这不是能力问题，是位置问题：他知道自己想做什么，所以看不见自己实际做了什么。**唯一的解法是换位置。**
>
> An author cannot audit their own work. It is not a matter of skill but of position: they know what they meant to build, so they cannot see what they actually built. **The only fix is to change position.**

本 skill 的产出物不是「评价」，是**起诉书**。没有判决的审查等于没审。

---

## 0. 何时启动

**唯一的门是 `/`。** frontmatter 里的 `disable-model-invocation: true` 让本 skill 从模型的技能目录里整体消失——模型看不见它，也就无法自作主张地「顺手审一下」。

启动条件：用户输入 `/adversarial-review`。没有别的入口。

**不启动**：草稿阶段、方案尚未成形、纯事实查询、用户只要建议不要判决。

⚠️ 一旦被召唤，Step 1–4 全量执行。**不得因为「这次看起来简单」而简化——豁免权只在用户手里。**

---

## 1. 铁律

违反任一条，本次审查作废，必须重来。

| # | 铁律 | 为什么 |
|---|---|---|
| 1 | **必须用 `subagent`（全新上下文）。禁止 `subagent_fork`、禁止自审替代。** | fork 继承「我已经做完了」的叙事，找茬力度归零。位置不换，审查无效。 |
| 2 | **举证责任倒置**：作者的「已完成」在见到可指认证据之前，一律视为未完成。 | 否则审查退化为作者自述的复读。 |
| 3 | **一 finding 一反诘，且反诘必须闭合**：写明「答什么就降级，答不出就升级」。 | 开放式提问 = 走过场。 |
| 4 | **默认不给 PASS**。零 finding 才允许 PASS，且必须附 `WITHDRAWN` 记录证明审查真发生过。 | 全绿的审查报告 99% 是没审。 |
| 5 | **落地三选一，禁止静默丢弃**：FIX / ACCEPT-DEBT / REBUT+证据。 | 未落地的 finding 等于不存在。 |

---

## 2. 四步流程

### Step 1 · 立卷

把交付物压成**可证伪的断言清单**。不是摘要，是主张。

```
断言 A1: <我声称做到了什么>  → 证据落点: <文件:行 / 命令回显 / 页面元素 / 无>
断言 A2: ...
```

规则：
- 一条断言只包含一个可判真假的命题。「完成了 X 并优化了 Y」= 两条。
- 证据落点写「无」的，直接进 D1，不必等 reviewer。
- 同时抄录**用户原始诉求全文**（不是你的转述）——这是 D2 的唯一弹药。

### Step 2 · 派单

调用 `subagent`（`context: fresh` 语义，不 fork）。`description` 写 `adversarial review`。

brief 必须使用 [references/review-brief.md](references/review-brief.md) 的完整模板，逐字填槽，不得压缩。**brief 里禁止出现你自己的辩护、你的设计意图、你的难处**——那是污染源。只给：原始诉求、交付物位置、你的断言清单、验收标准。

同时把 [references/audit-axes.md](references/audit-axes.md) 与 [references/interrogation.md](references/interrogation.md) 的路径交给它，要求逐条照做。

### Step 3 · 裁决

reviewer 返回后，你做三件事，**顺序不可换**：

1. **验真**：抽查 ≥ 2 条 finding 的证据落点是否真实存在。捏造的 finding 整批作废。
2. **定级**：按下表把 finding 归入 BLOCK / PATCH / DEBT。定级权在你，不在 reviewer。
3. **过闸**：BLOCK > 0 → 不得交付，先修或先改方案。

| 级别 | 判定条件 |
|---|---|
| **BLOCK** | ≥1 条 D3 静默错误（错而不报警）；或 ≥1 条 D1/D2 命中用户核心诉求 |
| **PATCH** | 无 BLOCK，但 ≥1 条可当场修的 defect |
| **DEBT** | 仅剩 D5 级，或已明示接受的取舍 |
| **PASS** | 0 finding，且 `WITHDRAWN` 非空 |

`CONFIDENCE`：high = 全部 finding 有可指认证据；med = 部分依赖推断；low = 主要靠常识推断 → **此时一律降为 DEBT，禁止 BLOCK**。

### Step 4 · 落地

对每条 finding 三选一，并**写在最终回复里**（不是心里）：

- `FIX` — 已改，附位置或 diff 摘要
- `ACCEPT-DEBT` — 不改，写清理由、残余风险、谁来承担
- `REBUT` — 不同意，附**证据**（不是「我觉得」）。反驳失败则回到 FIX。

第四种选项不存在。

---

## 3. 五轴（封闭集）

三个审查面（完成度 / 隐患 / 优化空间）拆成 5 个原子轴。**只能落一个轴，落不进去报 `AXIS-MISS`**。

| 轴 | 唯一判定问句 | 证据形态 |
|---|---|---|
| **D1 断言之缺** | 作者声称完成的事，证据在哪？ | 断言 → 产物位置的映射 |
| **D2 期待之缺** | 用户没被明说、但按常识会期待的东西，在哪？ | 原始诉求逐条对账表 |
| **D3 失效之径** | 在什么输入/状态下，它会给出错误结果且不自曝？ | 触发条件 + 错误结果 + 是否可被察觉 |
| **D4 替代之疑** | 有没有等价但更简单/更标准/更便宜的做法？ | 可执行替代方案 + 省掉什么 + 它的代价 |
| **D5 未取之值** | 方案成立的前提下，有哪些低代价价值被漏掉？ | 具体增量 + 成本量级 |

穷尽性来源：**缺（D1/D2）· 错（D3）· 冗（D4）· 增（D5）**——对「已完成物」的全部合理批评，只能落在这四类里。

轴定义、边界、升级门槛、互斥证明 → [references/audit-axes.md](references/audit-axes.md)。**逐条照做，不要凭印象简化。**

---

## 4. 输出格式

```
VERDICT: BLOCK | PATCH | DEBT | PASS
CONFIDENCE: high | med | low
BLOCKING: n | DEBT: n | WITHDRAWN: n

[BLOCKING]
B1 · D3 失效之径 · <文件:行 / 段落 / 命令回显>
  断言: ...
  证据: ...（可复现的最小片段）
  触发: <具体输入或状态> → <具体错误结果> → <为何使用者察觉不到>
  反诘: Q: <一句闭合问句>
        → 若答「<具体答复形态>」则降为 DEBT；答不出或答非所问则维持 BLOCK
  修法: <最小改动，一句话>

[DEBT]
T1 · D5 未取之值 · <定位>
  ... 成本估计: <量级>

[WITHDRAWN]  ← 必填，证明审查有校准、不是散弹枪
W1 · 原判 <...> · 撤回理由: <...>

[AXIS-MISS]  ← 仅当出现五轴装不下的问题时；同时提出新轴候选
M1 · ...

[只修一件事] <一句话，指向收益/成本比最高的那一条>
```

`[WITHDRAWN]` 不是谦虚，是**防伪装置**：一条都撤不回的审查，说明它在乱开枪。

---

## 5. 反模式（出现即判审查失败）

| 反模式 | 症状 |
|---|---|
| **三明治反馈** | 「整体不错，但是…」——先夸后批。对抗立场下第一句就是缺陷 |
| **软化词** | 「可能」「建议考虑」「或许可以」「酌情」→ 要么成立要么撤回 |
| **复读作者** | 把作者的自我描述当证据 |
| **幽灵 finding** | 无法指认到具体位置的问题 |
| **重写建议** | 「整个方案应该推翻重来」→ 属 D4，必须给等价替代 + 代价，否则不是 finding |
| **散弹枪** | 20+ 条 finding，全是低价值边角 → 只保留 BLOCKING + top 5 DEBT |
| **全绿** | 零 finding 且零 WITHDRAWN |
| **自我豁免** | 以「这次简单」为由跳过 → 只有用户能豁免 |

---

## 6. 失败模式

| 情况 | 处理 |
|---|---|
| `subagent` 不可用 | 内联自审，`VERDICT` 上限 PATCH，报告首行标 `SELF-REVIEW (degraded)` |
| 交付物超出可审范围 | 按「改动面 + 最高风险面」抽样，报告里声明抽样率与被排除区域 |
| reviewer 返回空或全是赞美 | 判为审查失败。重派一次，brief 首行加：`上一轮被判为审查不足，本轮必须至少给出 3 条可指认的缺陷，或明确证明其为不可能。` 二次失败 → 降级自审并标记 |
| reviewer 返回 20+ 条 | 只保留 BLOCKING + top 5 DEBT，其余压成一行统计 |
| finding 证据落点抽查不存在 | 该批 finding 作废，重派 |
| 5 轴全部装不下 | 报 `AXIS-MISS`，提出新轴候选；**不擅自扩轴** |

---

## 7. 调用方式 · Invocation

**唯一入口：对话栏打 `/`，选 `adversarial-review`。**

```
/adversarial-review  审一下刚交付的 {对象}
```

被这样召唤时，本文件全文会作为一条用户消息直接注入上下文——你读到这里，说明门已经开了。

**为什么没有别的入口**：frontmatter 的 `disable-model-invocation: true` 让本 skill 从模型的技能清单里彻底剔除。模型看不见它，就不会在写「已完成」的时候顺手给自己盖个章。这条路是单向的：只有用户的手能推开。

> 想让它自动生效？删掉这一行即可，代价是它会重新变成模型可以「顺手审一下」的东西——那正是这个 skill 要防的。

召唤后若未指明对象，**默认审最近一次交付**，并在报告头部写明 `SCOPE: <识别到的对象> (inferred — 未显式指定)`；猜错成本由用户一句话纠正，不要反问。

审查器本身也受审：用户说「审得不够狠 / 漏了 X」时，不要辩解，直接按 §6 重派一轮，并在报告 `ROUND: 2` 标注。
