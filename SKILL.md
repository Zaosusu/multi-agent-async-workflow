---
name: multi-agent-async-workflow
description: 多 Agent 异步协同工作流架构方案。当用户需要建立多 Agent / 多模型组成的异步协作流水线、用 GitHub Issues 作为任务总线、实现规划-研究-实施-审核等角色解耦、或将不同能力的模型按拓扑结构编排时，使用此 skill。适用于 AI-native 团队、自动化流水线、或任何需要高吞吐任务分发与异步闭环的场景。
agent_created: true
---

# Multi-Agent Async Workflow

多 Agent 异步协同工作流：用 GitHub Issues 作为任务总线，让不同能力的模型按拓扑结构异步协作，持续流转任务直至闭环。

## 核心模式

这不是 prompt 路由，不是 session 桥连 PRD——而是一个**基于 Issue 的多节点异步协作协议**：

```
┌─────────────────────────────────────────────────────────────┐
│                        任务总线                              │
│                     GitHub Issues                            │
│                                                             │
│  backlog → ready → in_progress → review → done             │
│              ↘         ↗         ↘        ↗                 │
│              阻塞/澄清反馈  审核打回  需人工介入              │
└─────────────────────────────────────────────────────────────┘
```

**关键洞察：Issue 在这里不是 bug tracker，是跨 session 的持久化消息队列。每个节点只关心自己负责的状态转换。**

## 节点类型

### Planner（规划侧）

| 职责 | 具体行为 |
|------|----------|
| 需求翻译 | 把模糊的业务需求拆成可执行的独立 Issue |
| 代码阅读 | 读相关模块，理解现状，找出改动点 |
| 方案设计 | 给出实现思路、边界条件、测试要点 |
| 任务编排 | 设定优先级、依赖关系、标签分类 |
| 持续 push | 源源不断生成/更新 Issue，维护 backlog |

### Researcher（研究侧）

| 职责 | 具体行为 |
|------|----------|
| 信息收集 | 针对 Issue 中的未知领域做调研 |
| 方案产出 | 输出研究结论、可行性分析、竞品对比 |
| 反馈给 Planner | 在 Issue 里 comment 研究结果，或生成新的子 Issue |

### Executor（实施侧）

| 职责 | 具体行为 |
|------|----------|
| 独立实现 | 按 Issue 描述写代码，不改范围 |
| 提交闭环 | 提交 PR，关联 Issue，自测通过后推进状态 |
| 遇到阻塞 | 在 Issue 里 comment 说明卡点，不自己决策 |

### Reviewer（审核侧）

| 职责 | 具体行为 |
|------|----------|
| 代码审查 | 审查 PR，关注正确性、边界条件、测试覆盖 |
| 批准/打回 | 通过则关闭 Issue；打回则 comment 具体问题，状态回退 |
| 质量把关 | 确保代码符合规范，不引入技术债 |

### Integrator（集成侧）

| 职责 | 具体行为 |
|------|----------|
| 合并与验证 | 合并多个相关 PR，确保整体一致 |
| 端到端测试 | 运行集成测试、E2E 测试 |
| 发布协调 | 与部署/发布流程对接 |

### Human（真人员工）

| 职责 | 具体行为 |
|------|----------|
| 决策确认 | 对阻塞/澄清类 Issue 做最终决策 |
| 质量把关 | 对高优 Issue 做人工 review |
| 边界处理 | 处理自动化流程无法覆盖的边缘情况 |

## 交付与审核闭环：Issue → PR → Review

Planner（A）发 Issue，Executor（B）做完了。三个问题：**B 交付什么？谁来审？上下文怎么过去？**

### B 的交付物只能是 PR

PR 同时具备可审核、可挂 CI、可回滚、可关联 Issue、可并发——它是唯一能被下游节点当作输入的交付形式。

**硬性要求：PR body 必须写 `Closes #<issue>`。** 这是寻址，不是礼貌——Reviewer 靠它反查任务契约。

B 完成后两件事缺一不可：开 PR（关联 Issue）+ 回 Issue comment `✅ 已提交 PR #N` 并把标签换成 `needs-review`。只开 PR 不换标签，Reviewer 的 loop 扫不到，任务静默死掉。

### Review 交给独立的 C，不回 A，也不由 B 自审

| 做法 | 判断 | 理由 |
|------|------|------|
| A 兼任 review | ❌ | A 是验收标准的作者，审自己定的标准等于自证；且规划侧变瓶颈 |
| 用完即弃 A | ❌ | A 是常驻 producer，上下文在 Issue 里而不在 A 的会话里 |
| 在 A 的 Issue 里指定要 review 的 PR | ⚠️ 部分 | 作为寻址信息对，但 review 的执行权不能挂在 A 身上 |
| 独立的 C（Reviewer）审这个 PR | ✅ **采用** | C 独立 session、独立模型，输入只有 PR diff + 反查到的 Issue |
| 共享 A/B/C 上下文的 agent team | ❌ 作默认方案 | 共享上下文杀死审核独立性；可作为单个节点内部的实现 |

C 只做一件事：**对着 Issue 的验收标准逐条判 PR。** 但打回的判据不是「验收标准里有没有写」，而是**问题在不在这个 diff 里**——diff 内部的正确性问题、回归、安全缺陷，即使验收标准没提也该打回；想让 PR 多做一件事（新功能、顺手重构），则回 A 开新 Issue。**可以拒收坏的实现，不可以扩大要求。**

### 上下文靠工件传递，不靠 session

Issue 承载 why / what / 验收标准（契约），PR 承载 how / diff / CI / 自检（交付），两者双向关联。C 的输入 = PR diff + 反查到的 Issue，足够独立判定是否满足验收标准。**契约已外化到工件上，所以不需要共享上下文。**

### 打回环与升级阈值

```
B: PR → C: review ├── 通过 → Issue done
                  └── 打回 → Issue 回 in_progress → B 的 loop 扫到 → 改 → 重新请审
```

B 不需要被通知，它的 loop 本来就在扫「assignee 是我 且 in_progress」——**状态本身就是通知。**

**必须设升级阈值：同一个 PR 被打回 2 次后打 `needs-human`。** 否则 B 和 C 无限对打且不收敛；两次之后说明分歧在标准本身，那是 A 和人的职责。

### 为什么这个场景必须是多智能体架构

**审核的价值来自独立性，而共享上下文恰好摧毁独立性。** 继承了 B 全部上下文的 reviewer 已经被 B 的推理链说服，会把 B 的假设当前提——它抓不到 B 的错，因为错就在它自己的前提里。C 的空上下文不是缺陷，是它唯一的资产。

| 维度 | 共享上下文的 agent team | Issue 总线上的独立节点 |
|------|------------------------|----------------------|
| 审核独立性 | 无——继承实施者的盲区 | 强——只看工件，不看过程 |
| 规模上限 | 一个上下文窗口就是天花板 | 无界 |
| 崩溃恢复 | session 挂了上下文全丢 | Issue/PR 在盘上，换节点接着做 |
| 并发 | 本质是串行对话 | N 个 Executor 真并行 |
| 模型异构 | 通常同一个模型 | 每节点可换模型；换厂商 review 能交叉出同模型的共同错法 |
| 可审计 | 埋在会话记录里 | 每步都是 comment / review，可统计 |
| 人的位置 | 人 review 每份产出 | 人 review 协议和队列健康度 |

**人不该 review 每个 PR，人该 review 这套系统**：吞吐、打回率（太高说明 Issue 写得糙，太低说明 C 在放水）、`blocked` 堆积、升级频次。指标异常时改的是 Issue 模板和节点 prompt，不是某个 PR。

**agent team 什么时候合适**：强耦合、一次性、高频来回的探索（如定位偶发 bug），拆 Issue 成本高于收益。此时它是**某个节点的内部实现**，对总线只暴露一个结论 comment。**团队在节点内，协议在节点间。**

## 流水线拓扑

### 线性流水线

```
Planner → Researcher → Executor → Reviewer → Integrator → Done
```

适用场景：需求明确、流程标准、需要多道质量关卡。

### DAG（有向无环图）

```
Planner
    ├── Issue-A → Executor-A → Reviewer-A
    └── Issue-B → Researcher → Executor-B → Reviewer-B
                ↘
                 Issue-C → Executor-C
```

适用场景：并行任务多、部分任务需要研究、部分可以直接实施。

### 环形反馈

```
Executor → Reviewer → [打回] → Executor
                ↓
            [通过] → Done
```

适用场景：对质量要求高，允许迭代修正。

### 扇入/扇出

```
Planner
    ├── Issue-1 → Executor-A
    ├── Issue-2 → Executor-B
    └── Issue-3 → Executor-C
              ↓
        Integrator（合并）
```

适用场景：一个 Epic 拆成多个子任务，最后需要集成。

### 人工介入点

```
Executor → [阻塞/需确认] → Human → [决策] → Executor
```

适用场景：涉及业务决策、模糊需求、高风险变更。

## Issue 协议规范

为了让多节点稳定协作，Issue 本身需要承载足够信息。

### Issue 模板（Planner 填写）

```markdown
## 背景
[为什么做这个改动]

## 目标
[期望达到的效果]

## 涉及范围
- 文件：`src/xxx.ts`, `src/yyy.ts`
- 模块：xxx module

## 实现要点
1. [关键步骤 1]
2. [关键步骤 2]

## 验收标准
- [ ] 测试用例 A 通过
- [ ] 不影响现有功能 B

## 优先级
P0 / P1 / P2

## 标签
`enhancement` `area:xxx` `draft`

## 指定执行者
@executor-A / @executor-B / @human

## 依赖
- #123 (必须先完成)
- #456 (可并行)
```

### 节点 Comment 规范

| 节点 | Comment 格式 |
|------|-------------|
| Planner | `📋 新建 Issue #[number]，背景：xxx` |
| Researcher | `🔍 研究完成：结论是 xxx，建议方案：yyy` |
| Executor | `👋 开始处理，预计 [时间]` / `✅ 已提交 PR #[number]` |
| Reviewer | `👍 通过` / `🚫 打回：具体问题 xxx` |
| Integrator | `🔗 已合并 PR #[a] 和 #[b]，集成测试通过` |
| Human | `✅ 已确认：xxx` / `❌ 不同意，理由：xxx` |
| 任意节点 | `🚧 阻塞原因：xxx，需要 @xxx 确认` |

## 使用方式

### 1. 基础配置

确定一个 GitHub Repository 作为任务总线：
- 建立 Issue 模板
- 配置 Labels（`enhancement` / `bug` / `draft` / `blocked` / `needs-review` 等）
- 配置 Milestones（可选，用于按版本管理）

### 2. 各节点 Prompt 要点

#### Planner

- 扫描代码库变更和业务需求
- 生成结构化 Issue，包含完整字段
- 控制并发：一次最多 N 个 in_progress
- 参考 `assets/issue-template.md`

#### Researcher

- 扫描带有 `needs-research` 标签的 Issue
- 产出研究结论，更新 Issue 描述或添加 comment
- 如发现需求不清晰，标记 `blocked` 并 @Planner

#### Executor

- 扫描分配给自己的 Issue（含被打回退回 `in_progress` 的）
- 按描述实现，不超出范围
- 提交 PR，body 写 `Closes #<issue>`，套用 `assets/pr-template.md`
- 回 Issue comment `✅ 已提交 PR #N` 并换标签为 `needs-review`

#### Reviewer

- 扫描带有 `needs-review` 标签的 Issue，反查其关联 PR
- 逐条对着 Issue 的验收标准判 PR，不加新需求（要加就回 Planner 开 Issue）
- 通过则推进 `done`；打回则在 PR 上留具体到文件行的 comment，Issue 退回 `in_progress`
- 同一 PR 打回 2 次后打 `needs-human` 升级

#### Integrator

- 扫描所有已合并的 PR，识别需要集成的任务
- 合并相关分支，运行集成测试
- 验证通过后关闭 Epic Issue

#### Human

- 扫描带有 `blocked` 或 `needs-human` 标签的 Issue
- 做决策、澄清需求、处理例外

### 3. 会话桥连方案

#### 方案 A：独立 Session + 定时轮询（推荐起步）

```
Planner Session:  每 N 小时 → 扫描 → 生成 Issue
Researcher Session: 每 M 小时 → 扫描 needs-research → 研究
Executor Session:  每 M 小时 → 扫描 assigned to me → 实现
Reviewer Session:  每 M 小时 → 扫描 needs-review → 审查
```

- 优点：完全解耦，各自独立运行
- 缺点：延迟取决于轮询间隔

#### 方案 B：Webhook 触发

```
GitHub Webhook → Issue labeled / PR opened → 触发对应 Session
```

- 优点：实时响应
- 缺点：需要部署 webhook receiver

#### 方案 C：人工 / Agent 调度器

```
调度器 Agent → 检查 Issue/PR 状态 → 决定唤醒哪个节点 Session
```

- 优点：集中控制，可以做更复杂的编排
- 缺点：多一层调度复杂度

## 多角色协同扩展

### 多对多（Mesh）

```
Planner-A ─┐
Planner-B ─┤→ Issue Bus ←┬── Executor-A
Planner-C ─┘              ├── Executor-B
                          ├── Reviewer-A
                          └── Reviewer-B
```

做法：用标签/里程碑/assignee 区分来源和职责，各节点按规则过滤处理。

### 按领域拆分

```
Planner（技术债） → Executor（后端） → Reviewer（后端）
Planner（业务需求） → Executor（前端） → Reviewer（前端）
Planner（探索） → Researcher → Planner（转成 Issue）
```

做法：不同领域用不同 milestone 或 repo 隔离，避免交叉干扰。

### 质量门控

```
Executor → [自动化 CI] → [通过] → Reviewer → [通过] → Integrator
                ↓                    ↓
             [失败] → Executor    [打回] → Executor
```

做法：CI 状态作为第一个关卡，减少 Reviewer 负担。

## 为什么 Issue 比 PRD 更适合做资产管理

| 维度 | PRD | GitHub Issues |
|------|-----|---------------|
| 结构化程度 | 自由文本，格式不统一 | 模板化，字段固定 |
| 可追踪性 | 文档更新历史，不显式 | 每个 Issue 有独立生命周期 |
| 异步边界 | 弱，需要人解读 | 强，标签/状态/comment 自解释 |
| 并发安全 | 多人编辑冲突 | 天然支持多 Issue 并行 |
| 集成度 | 离代码远 | 原生关联 PR、Commit、CI |
| 搜索/筛选 | 全文搜索 | 标签、milestone、assignee 多维筛选 |
| 成本 | 高（需要维护文档） | 低（Issue 本身就是资产） |

**核心优势：Issue 是代码和需求之间的「标准化接口」。**

## 资源文件

- `references/issue-protocol.md`：Issue 模板与 Comment 规范
- `references/pr-review-protocol.md`：交付与审核闭环（Issue → PR → Review）
- `references/best-practices.md`：落地检查清单与常见坑
- `assets/issue-template.md`：可直接复制使用的 Issue 模板
- `assets/pr-template.md`：可直接复制使用的 PR 模板
- `assets/comment-protocol.md`：各节点 Comment 规范
