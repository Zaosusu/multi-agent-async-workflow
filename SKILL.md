---
name: multi-agent-async-workflow
description: 多 Agent 异步协同工作流。两类场景使用：(1) 你被指派为流水线中的某个节点（Planner / Researcher / Executor / Reviewer / Integrator）去干活——扫 Issue 队列、认领任务、处理、流转状态；(2) 用户要建立多 Agent 异步协作流水线、用 GitHub Issues 作为任务总线、做规划-研究-实施-审核的角色解耦。当出现「看一下队列里有没有我的任务」「按 Issue 实施并提 PR」「review 这个 PR 是否满足 Issue 验收标准」「把需求拆成可执行 Issue」等请求，或用户提到任务总线 / 节点 / 打回 / needs-review 等本协议术语时，加载此 skill。
agent_created: true
---

# Multi-Agent Async Workflow

用 GitHub Issues 作为任务总线，多个独立节点异步协作。**你是其中一个节点。**

## 先确定你的入口

| 情况 | 去哪 |
|------|------|
| 你是这条流水线的总负责人 / 总指挥 | 继续往下：开工前检查 → 通用硬规则 → 「你是总负责人」 |
| 你被指派为某个具体节点（Executor / Reviewer / Researcher / Integrator） | 继续往下：开工前检查 → 通用硬规则 → 你的角色章节 |
| 用户要从零搭这套流水线 | 读 `references/setup.md` |
| 用户想理解/评估这套架构 | 读 `references/rationale.md` |

## 开工前检查（每轮必做）

1. **确认总线 repo**：用户指定的 repo，或当前 repo。不确定就问，不要猜。
2. **确认你是哪个节点**：用户明确指派的角色。**没有指派就问，不要自己挑。** 角色决定你的权限边界，猜错会越权。
3. **确认标签齐全**：`gh label list | grep -E 'ready|in_progress|needs-review|needs-lead|blocked|needs-human'`。缺标签说明总线没配好 → 停下，告诉用户去看 `references/setup.md`。
4. **确认本轮边界**：默认**只做一个 pass**——扫队列、处理你能处理的任务、流转状态、报告、结束。除非用户明确要求常驻，否则不要自己开无限循环。

## 通用硬规则（所有节点）

这些违反了会破坏整条流水线，优先级高于你对任务的判断：

1. **状态即通知。** 每个动作结束必须更新标签。只干活不换标签 = 下游节点扫不到 = 任务静默死亡。这是最常见的致命错误。
2. **不越权。** 只做你这个角色的事。**唯一的硬红线：不能审、不能合并你自己写的代码。** 除此之外，角色是可以兼任的（见下节）。你不确定该不该做某件事时，答案是不做，comment 说明并交给对应角色。
3. **不扩大范围。** 只处理认领的那个 Issue 写明的事。发现别的问题 → 开新 Issue 或 comment，不要顺手改。
4. **卡住不自己决策。** 遇到需求不清、权限不足、方案有分歧：comment 说明 + 打标签，然后停。硬猜下去的代价远大于等一轮。**分清找谁**：分歧在规格/优先级/技术方案 → `needs-lead`（总负责人能裁决）；涉及花钱、对外承诺、法律权限、业务方向 → `needs-human`（必须真人）。
5. **一次只认领一个。** 处理完一个再拿下一个，不要批量认领——你可能中途失败，被你锁住的任务会一起卡死。
6. **所有交接信息写在 Issue / PR 上。** 不要指望「下一个节点知道我刚才想了什么」——它是全新的上下文，只能看见工件。你没写下来的等于不存在。

## 角色兼任与合并权

**review 权和合并权默认都在总负责人手上。** 拆出独立的 Reviewer 不是架构要求，而是总负责人的一次**任命**——任命了就归被任命者判，没任命就自己判。

```
总负责人（默认握有全部权限）
  ├── 未任命 → 自己定规格、自己验收、自己决定合并
  └── 任命了 review 负责人 → 由被任命者判定是否达标，总负责人仍持合并权
```

节点数少于角色数是正常配置，不是妥协。角色可以兼任，只有一条不能破：

| 兼任 | 是否允许 |
|------|---------|
| Planner + Reviewer + Integrator（**总负责人**，默认形态） | ✅ 他没写代码，审别人的实现独立性成立；写了规格反而最有资格判断规格是否被满足 |
| Executor + Reviewer，**在同一个 PR 上** | ❌ **唯一的硬红线** |
| Executor + Integrator（合并自己的 PR） | ❌ 同上，属于自批 |
| Executor-A 审 Executor-B 的 PR | ✅ 互审没问题 |

「不能自审」约束的是**代码作者**，不是规格作者。总负责人定了验收标准又去 review 实现，这不叫自证——那是 ownership，人类工程里开 ticket 的 tech lead 来审来合本来就是主流做法。

### 任命的表达方式

任命必须**写在工件上**，否则被任命者下一轮不知道自己有这个权：在 Issue 里写明 `review 负责人：@xxx`，或在总线 repo 的 README / 置顶 Issue 里声明常任分工。口头说过等于没说。

### 合并权归总负责人

**Executor 提 PR，但对合并没有任何决定权。** 即使 review 通过了，也由总负责人决定：

1. **哪些 PR 可以进**——包括「代码本身没错，但基于已废弃的架构，所以不该合」这类判断
2. **按什么顺序进**——多个 PR 文件范围重叠时由总负责人裁决谁先合、谁去 rebase。**不要让 Executor 自行协商**，它们互相看不见彼此的上下文
3. **是否只择取部分内容**——旧 PR 里有可复用的东西时，关掉 PR 并开新 Issue 写明「从 PR #N 摘取 xxx」，而不是硬合

Executor 侧对应的禁令：不合并任何 PR（含自己的）、不推 main、不 force push。

### 全兼任的代价（必须知道自己在换什么）

总负责人同时定规格、验收、合并时，**它自己的误解会系统性地传下去且无人能拦**：

```
误解需求 → 写进验收标准 → Executor 照做 → 用同一个误解去验收 → 通过 → 合并
```

Executor 拦不住（它只负责符合规格），CI 也拦不住（代码没 bug）。这条链上没有任何一环能发现「做出来的东西不是要的东西」。

两个补偿手段：

- **任命 review 负责人时，明确给它质疑规格的权力**（详见 Reviewer 章节）
- **真人分两阶段介入**（见下节）——这是唯一能拦住上面那条链的办法

什么时候值得任命独立 Reviewer：想要**跨模型交叉验证**（换一家的模型审，同模型容易犯同样的错），或总负责人自己成了吞吐瓶颈。

## 真人的位置：只看两样东西

真人不看 PR、不看 diff、不做 code review。只在两个地方投入：

| 阶段 | 真人看什么 | 为什么必须是这里 |
|------|-----------|----------------|
| **① 底层框架期** | 骨架本身：架构选型、目录结构、技术栈、数据模型、接口契约、CI 基线 | 框架的错会被之后**每个任务继承**，且**抽查交付结果发现不了**——每个交付相对于错框架都是"正确"的。这是唯一必须逐项过目的阶段 |
| **② 框架验收后** | 只看交付结果：做出来的东西是不是要的东西 | 框架固定后，单个任务的偏差**会体现在结果上**，抽查就够了。此时看 PR 是浪费——那是总负责人的职责 |

### 框架未验收前的硬约束

**框架通过真人验收之前，Planner 不放新功能 Issue。** 只放骨架相关的任务。

理由是复利方向：框架期的一个错误决定，会以后续每个任务为倍数放大；而这个阶段又恰恰是唯一能廉价改正的时候——一旦上面堆了二十个功能，改框架的成本就从「改一次」变成「改二十一次」。

对应地，如果你接手一个已经跑起来但框架有问题的项目：**先停止派发新功能，把存量 Issue / PR 按新框架重新分类（继续 / 关闭替代 / 已完成 / 待重写），骨架验收通过后再恢复派发。**

## 认领协议

多个同角色节点可能同时扫到同一个 Issue。认领时：

```bash
gh issue edit <n> --add-assignee @me --add-label in_progress --remove-label ready
gh issue view <n> --json assignees   # 立刻重读校验
```

重读后如果 assignee 里有别人且不是你在首位 → **让给对方，去拿下一个**。宁可放弃也不要双份产出。

---

## 你是总负责人

默认形态：你同时是 Planner、Reviewer、Integrator。按顺序读这三节执行，外加以下只属于你的职责。

**独有职责**

1. **任命与授权**：决定是否把 review 权委派出去。任命必须写在工件上（Issue 里写 `review 负责人：@xxx`，或总线 repo 置顶 Issue 声明常任分工），否则被任命者下一轮不知道自己有这个权。
2. **裁决 `needs-lead`**：分歧在规格、优先级、技术方案上的，你拍板。这是你的活，别往真人那儿推。
3. **持合并权**：即使 review 通过，进不进、什么顺序进、是否只择取部分内容，由你定（见「合并权归总负责人」）。
4. **守框架闸门**：真人验收框架之前，不放新功能 Issue。
5. **认识自己的盲区**：你定规格又验收，误解会无人拦阻地传下去。反复返工、同类打回时**先怀疑自己的规格**，而不是实施质量。

**扫描**
```bash
gh issue list --label needs-lead --state open      # 待你裁决的，优先
gh pr list --state open                            # 待你决定合并的
gh issue list --label needs-review --state open    # 未任命 Reviewer 时你自己审
gh issue list --label backlog --state open         # 然后才是产新任务
```

处理顺序是固定的：**先清裁决和合并，再产新任务。** 反过来会让队列越积越长——你是唯一的出口，堵住出口去灌入口只会更糟。

**禁止**
- 自己去实施你定的 Issue（写了代码就不能再审它）
- 替真人做决定：花钱、对外承诺、法律权限、业务方向 → `needs-human`
- 框架未验收就派发新功能

---

## 你是 Planner

**扫描**
```bash
gh issue list --label backlog --state open
```

**动作**
1. 把模糊需求拆成独立 Issue，粒度控制在 **1-2 小时可完成**（拆不动说明还没想清）
2. 读相关代码，在 Issue 里写清 `涉及范围`（具体文件），这是下游避免冲突的依据
3. 套用 `assets/issue-template.md`，**验收标准必须可判定**——「性能更好」不行，「P99 < 200ms」才行
4. 标注依赖关系和优先级；预分配执行者（填 `指定执行者`）可以彻底避免认领竞争
5. 置 `ready` 交出

**禁止**
- 同时把**文件范围重叠**的多个 Issue 置 `ready`——并行实施必然冲突
- 一次放出超过 N 个 `in_progress`（N = 实施节点数），否则队列积压
- 自己去实施你定的 Issue（写了代码就不能再审它）

> 兼任 Reviewer / Integrator 是允许的，见「角色兼任与合并权」。审别人实现的 PR 不违反自审约束。

---

## 你是 Researcher

**扫描**
```bash
gh issue list --label needs-research --state open
```

**动作**
1. 只调研 Issue 里问的那个问题
2. 结论 comment 回 Issue：`🔍 研究完成：结论是 xxx，建议方案：yyy`
3. **产出必须可落地**——把结论转成具体可执行的 Issue，或补全原 Issue 的实现要点
4. 转 `ready`（可实施）或 `blocked`（发现需求本身不清）

**禁止**
- 交一份没人能接的研究报告就完事
- 顺手把方案实现掉

---

## 你是 Executor

**扫描**（含被打回退回来的）
```bash
gh issue list --assignee @me --label in_progress --state open
gh issue list --label ready --state open        # 无人认领的
```

**动作**
1. 按认领协议锁定 Issue
2. 开分支：`<你的节点名>/<issue号>-<slug>`，例 `executor-a/128-fix-retry-backoff`
3. 按 Issue 实现。**验收标准写不明确 / 前后矛盾 → 不要硬猜**：comment 问题 + 打 `needs-clarification` 退回 `ready` 给 Planner
4. 自测通过后提 PR，套用 `assets/pr-template.md`，**body 必须写 `Closes #<issue号>`**（Reviewer 靠它反查契约，缺了 PR 就是无从判定的 diff）
5. 收尾**两件事缺一不可**：
```bash
gh pr create --title "..." --body "Closes #128\n\n..."
gh issue comment 128 --body "✅ 已提交 PR #256"
gh issue edit 128 --add-label needs-review --remove-label in_progress
```

**被打回时**：改完 comment `🔄 已按 review 意见修正，PR #N 请复审`，重新置 `needs-review`。

**禁止**
- 直接推 main / force push
- 改 CI 配置、密钥、依赖版本（除非 Issue 明确要求）
- 超出 Issue 范围的「顺手优化」
- **review 或合并自己的 PR**

---

## 你是 Reviewer

**前提**：你必须不是这个 PR 的**代码作者**。是自己写的就停下，交给别人。（定规格的人来审不违反此约束——见「角色兼任与合并权」。）

**扫描**
```bash
gh issue list --label needs-review --state open
gh pr view <n> --json body,files,statusCheckRollup   # 反查关联 Issue、看 CI
```

**动作**
1. PR 没写 `Closes #<issue>` → 直接打回，不进入审查（无契约不可判定）
2. CI 红 → 打回，不浪费一轮人工审查
3. 读 Issue 的验收标准，**逐条对着 diff 判**
4. 通过 → `👍 通过，可以合并`，Issue 置 `done`（有 Integrator 则交给它）
5. 打回 → comment **具体到文件和行**：`🚫 打回（第 N 次）：xxx`，Issue 退回 `in_progress`，PR 保持 open

**打回的判据是「问题在不在这个 diff 里」，不是「验收标准有没有写」**

| 情况 | 能否打回 |
|------|---------|
| diff 内部的正确性问题、回归、安全/资源缺陷 | ✅ 可以，**即使验收标准没提** |
| 想让这个 PR 多做一件事（新功能、顺手重构、覆盖新场景） | ❌ 不可以，回 Planner 开新 Issue，本 PR 该过就过 |

**可以拒收坏的实现，不可以扩大要求。** 只按标准打勾会放过 Planner 没预料到的真 bug，而那正是独立 review 最该抓的；反过来在 review 里加需求会让 PR 永远合不进去。

**你有质疑规格的权力，但方向是向上。** 认为验收标准本身定错了（不是不够详细，是方向错）→ comment 说明 + 打 `needs-lead` 交给总负责人裁决。**不要把这个判断压给 Executor**——它只负责符合规格，改规格不是它的权限。这是「不准扩大要求」的例外：往上提规格问题合法，往下加需求不合法。

**升级：分清是找上级还是找真人**

| 情况 | 打什么标签 | 谁处理 |
|------|-----------|--------|
| 同一 PR 已打回 2 次，分歧在验收标准本身 | `needs-lead` | 总负责人裁决。你和 Executor 谈不出来，继续对打只会烧钱且不收敛 |
| 涉及花钱、对外承诺、法律/权限、业务方向、拿不到的外部信息 | `needs-human` | 必须到真人，总负责人也不该替它决定 |

---

## 你是 Integrator

**扫描**
```bash
gh pr list --state open --label approved
```

**动作**
1. 合并已批准且 CI 绿的 PR；冲突**不要自己硬解**——退回对应 Executor（它有上下文，你没有）
2. 跑集成 / E2E 测试
3. 通过 → 关闭相关 Issue；失败 → `❌ 集成测试失败：xxx`，相关 Issue 退回 `in_progress`
4. 多个 PR 合成一个 Epic 时，确认整体一致而非逐个正确

**禁止**：合并未经 Reviewer 批准的 PR；跳过集成测试。

---

## 报告格式

每轮结束向用户汇报，别只说「做完了」：

```
处理：#128 修复重试退避
动作：提交 PR #256，Issue 转 needs-review
下一步：等 Reviewer 节点
队列：ready 3 个 / in_progress 1 个 / blocked 1 个（#131 等 Human 决策）
```

## 参考

| 文件 | 内容 |
|------|------|
| `assets/issue-template.md` | Issue 模板（Planner） |
| `assets/pr-template.md` | PR 模板（Executor） |
| `assets/comment-protocol.md` | 各节点 comment 话术 |
| `references/issue-protocol.md` | 完整标签体系与状态机 |
| `references/pr-review-protocol.md` | 交付与审核闭环细则 |
| `references/setup.md` | 从零搭总线（建 label、配模板、编排节点） |
| `references/rationale.md` | 为什么这么设计、为什么不用共享上下文的 agent team |
| `references/best-practices.md` | 落地检查清单与 15 个坑 |
