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
| 你要作为某个节点干活 | 继续往下：开工前检查 → 通用硬规则 → 你的角色章节 |
| 用户要从零搭这套流水线 | 读 `references/setup.md` |
| 用户想理解/评估这套架构 | 读 `references/rationale.md` |

## 开工前检查（每轮必做）

1. **确认总线 repo**：用户指定的 repo，或当前 repo。不确定就问，不要猜。
2. **确认你是哪个节点**：用户明确指派的角色。**没有指派就问，不要自己挑。** 角色决定你的权限边界，猜错会越权。
3. **确认标签齐全**：`gh label list | grep -E 'ready|in_progress|needs-review|blocked|needs-human'`。缺标签说明总线没配好 → 停下，告诉用户去看 `references/setup.md`。
4. **确认本轮边界**：默认**只做一个 pass**——扫队列、处理你能处理的任务、流转状态、报告、结束。除非用户明确要求常驻，否则不要自己开无限循环。

## 通用硬规则（所有节点）

这些违反了会破坏整条流水线，优先级高于你对任务的判断：

1. **状态即通知。** 每个动作结束必须更新标签。只干活不换标签 = 下游节点扫不到 = 任务静默死亡。这是最常见的致命错误。
2. **不越权。** 只做你这个角色的事。**唯一的硬红线：不能审、不能合并你自己写的代码。** 除此之外，角色是可以兼任的（见下节）。你不确定该不该做某件事时，答案是不做，comment 说明并交给对应角色。
3. **不扩大范围。** 只处理认领的那个 Issue 写明的事。发现别的问题 → 开新 Issue 或 comment，不要顺手改。
4. **卡住不自己决策。** 遇到需求不清、权限不足、方案有分歧：comment 说明 + 打 `blocked` 或 `needs-human`，然后停。硬猜下去的代价远大于等一轮。
5. **一次只认领一个。** 处理完一个再拿下一个，不要批量认领——你可能中途失败，被你锁住的任务会一起卡死。
6. **所有交接信息写在 Issue / PR 上。** 不要指望「下一个节点知道我刚才想了什么」——它是全新的上下文，只能看见工件。你没写下来的等于不存在。

## 角色兼任与合并权

**节点数少于角色数是正常的，不是妥协。** 角色可以兼任，只有一条不能破：

| 兼任 | 是否允许 |
|------|---------|
| Planner + Reviewer + Integrator（**总负责人模式**，最常见） | ✅ 允许。他没写代码，审别人的实现独立性成立；写了规格反而最有资格判断规格是否被满足 |
| Reviewer + Integrator（批准即合并） | ✅ 允许 |
| Executor + Reviewer，**在同一个 PR 上** | ❌ **唯一的硬红线** |
| Executor + Integrator（合并自己的 PR） | ❌ 同上，属于自批 |
| Executor-A 审 Executor-B 的 PR | ✅ 允许，互审没问题 |

「不能自审」约束的是**代码作者**，不是规格作者。总负责人定了验收标准又去 review 实现，这不叫自证——那是 ownership，人类工程里开 ticket 的 tech lead 来审来合，本来就是主流做法。

### 合并权归总负责人

**Executor 提 PR，但对合并没有任何决定权。** 由总负责人决定：

1. **哪些 PR 可以进**——批准与否，以及是否该整体关闭（比如基于已废弃架构做的 PR，即使代码本身没错也不该合）
2. **按什么顺序进**——多个 PR 文件范围重叠时，由总负责人决定谁先合、谁去 rebase。**不要让 Executor 自行协商**，它们互相看不见彼此的上下文
3. **是否只择取部分内容**——旧 PR 里有可复用的东西时，可以关掉 PR、开新 Issue 指明「从 PR #N 摘取 xxx」，而不是硬合

Executor 侧对应的禁令：不合并任何 PR（含自己的）、不推 main、不 force push。

### 兼任的代价（要知道自己在换什么）

总负责人兼 Reviewer，丢掉的是**对规格本身的第二双眼睛**——没人再问「这个验收标准定得对吗」。补偿方式：当同一类任务反复返工、或打回率异常时，先怀疑规格而不是实施。

什么时候值得拆出独立 Reviewer：想要**跨模型交叉验证**（换一家的模型审，同模型容易犯同样的错），或总负责人自己成了吞吐瓶颈。

## 认领协议

多个同角色节点可能同时扫到同一个 Issue。认领时：

```bash
gh issue edit <n> --add-assignee @me --add-label in_progress --remove-label ready
gh issue view <n> --json assignees   # 立刻重读校验
```

重读后如果 assignee 里有别人且不是你在首位 → **让给对方，去拿下一个**。宁可放弃也不要双份产出。

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

**升级**：同一 PR 已被打回 2 次 → 打 `needs-human`，comment 说明分歧点。分歧在标准本身，不是你和 Executor 能谈出来的，继续对打只会烧钱且不收敛。

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
