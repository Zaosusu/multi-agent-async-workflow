# Multi-Agent Async Workflow

多 Agent 异步协同工作流：用 GitHub Issues 作为任务总线，让不同能力的模型按拓扑结构异步协作，持续流转任务直至闭环。

> 主要贡献者：群友zzszmyf

![Architecture Overview](assets/architecture-overview.svg)

## 核心模式

这不是 prompt 路由，不是 session 桥连 PRD——而是一个**基于 Issue 的多节点异步协作协议**：

**关键洞察：Issue 在这里不是 bug tracker，是跨 session 的持久化消息队列。每个节点只关心自己负责的状态转换。**

### 运行方式

这个架构把 Agent 理解为模型的承载工具：真正干活的是背后的模型，而不是 Agent 本身。因此每个节点只需要是一个独立的 Session，背后跑着不同能力的模型。这些 Session 不需要预先编排好顺序，也不需要中心调度器——它们只需要无限循环地做一件事：**去看 Issue 队列，认领自己能处理的 Issue，处理完流转状态，然后继续看。**

这种「看 Issue → 处理 → 流转 → 再看」的无限 loop，就是整个系统的驱动力。

### Issue 状态机

![Issue State Machine](assets/issue-state-machine.svg)

## 节点类型

### Planner（规划侧）

- 需求翻译：把模糊的业务需求拆成可执行的独立 Issue
- 代码阅读：读相关模块，理解现状，找出改动点
- 方案设计：给出实现思路、边界条件、测试要点
- 任务编排：设定优先级、依赖关系、标签分类
- 持续 push：源源不断生成/更新 Issue，维护 backlog

### Researcher（研究侧）

- 信息收集：针对 Issue 中的未知领域做调研
- 方案产出：输出研究结论、可行性分析、竞品对比
- 反馈给 Planner：在 Issue 里 comment 研究结果，或生成新的子 Issue

### Executor（实施侧）

- 独立实现：按 Issue 描述写代码，不改范围
- 提交闭环：提交 PR，关联 Issue，自测通过后推进状态
- 遇到阻塞：在 Issue 里 comment 说明卡点，不自己决策

### Reviewer（审核侧）

- 代码审查：审查 PR，关注正确性、边界条件、测试覆盖
- 批准/打回：通过则关闭 Issue；打回则 comment 具体问题，状态回退
- 质量把关：确保代码符合规范，不引入技术债

### Integrator（集成侧）

- 合并与验证：合并多个相关 PR，确保整体一致
- 端到端测试：运行集成测试、E2E 测试
- 发布协调：与部署/发布流程对接

### Human（真人员工）

- 决策确认：对阻塞/澄清类 Issue 做最终决策
- 质量把关：对高优 Issue 做人工 review
- 边界处理：处理自动化流程无法覆盖的边缘情况

## 交付与审核闭环：Issue → PR → Review

假设 Planner（A）发了一个 Issue，Executor（B）做完了。三个问题：**B 交付什么？谁来审？上下文怎么过去？**

### B 的交付物只能是 PR

不是在 Issue 里贴 diff，不是直接推 main。PR 才同时具备可审核、可挂 CI、可回滚、可关联 Issue、可并发这几件事——它是唯一能被下游节点当作输入的交付形式。

**硬性要求：PR body 必须写 `Closes #<issue>`。** 这一行不是礼貌，是寻址——Reviewer 靠它反查任务契约，缺了它 PR 就是一堆无从判定对错的 diff。

B 完成后要做两件事，缺一不可：开 PR（关联 Issue）+ 回 Issue comment `✅ 已提交 PR #256` 并把标签换成 `needs-review`。只开 PR 不换标签，Reviewer 的 loop 扫不到，任务静默死掉。

### Review 交给独立的 C，不回 A，也不由 B 自审

| 做法 | 判断 | 理由 |
|------|------|------|
| A（Planner）兼任 Reviewer | ✅ **允许，小规模下的默认** | A 没写这段代码，独立性成立；A 写了验收标准反而最有资格判断标准是否被满足。这就是「总负责人」模式 |
| B（Executor）审自己的 PR | ❌ **唯一的硬红线** | 代码作者已被自己的推理链说服，抓不到自己的错 |
| 用完即弃 A | ❌ | A 是常驻 producer，上下文在 Issue 里而不在 A 的会话里 |
| 在 A 的 Issue 里指定要 review 的 PR | ⚠️ 部分 | 作为寻址信息是对的，但审核动作要由实际承担 Reviewer 角色的节点做 |
| 拆出独立的 C（Reviewer） | ✅ 规模化时的优化 | 想跨模型交叉验证，或总负责人成了吞吐瓶颈时拆 |
| 共享 A/B/C 上下文的 agent team，由人 review team | ❌ 作默认方案 | 共享上下文杀死审核独立性；但可作为**单个节点内部**的实现 |

**不要把「不能自审」外推成「规格作者不能审」。** 前者约束的是**代码作者**；后者是不同的事，硬禁的后果是只有 1 个 lead + N 个 Executor 时没人能合并，流水线直接死锁。**合并权归总负责人**：由它决定哪些 PR 能进、按什么顺序进、是否只择取部分内容。

C 只做一件事：**对着 Issue 的验收标准逐条判 PR。** 但打回的判据不是「验收标准里有没有写」，而是**问题在不在这个 diff 里**——diff 内部的正确性问题、回归、安全缺陷，即使验收标准没提也该打回；想让这个 PR 多做一件事（新功能、顺手重构），则回 A 开新 Issue，本 PR 该过就过。**可以拒收坏的实现，不可以扩大要求。**

### 上下文靠工件传递，不靠 session

```
Issue  ──承载──→  why / what / 验收标准     = 契约
  ↕ 双向关联
 PR    ──承载──→  how / diff / CI / 自检     = 交付
```

C 的输入 = PR diff + 反查到的 Issue，足够独立判定「是否满足验收标准」。**契约已经外化到工件上了，所以不需要共享上下文。**

### 打回环与升级阈值

```
B: PR #256 → C: review
                ├── 通过 → Issue done
                └── 打回 → Issue 回 in_progress
                             └→ B 的 loop 扫到「我的 PR 被打回」→ 改 → 重新请审
```

B 不需要被通知。它的 loop 本来就在扫「assignee 是我 且 `in_progress`」，被打回的任务自然回到视野里——**状态本身就是通知。**

**必须设升级阈值：同一个 PR 被打回 2 次后打 `needs-human`。** 否则 B 和 C 会在「我觉得可以了 / 我觉得还不行」之间无限对打且不收敛。两次之后说明分歧在标准本身，而标准是 A 和人的职责。

## 为什么这个场景必须是多智能体架构

既然 A、B、C 要协作，为什么不开一个共享上下文的 agent team，让人来 review 整个 team？

因为**审核的价值来自独立性，而共享上下文恰好摧毁独立性。**

一个继承了 B 全部上下文的 reviewer，已经读过 B 的推理链并被它说服了。它会把 B 的假设当前提、把 B 的取舍当既定事实——**它抓不到 B 的错，因为 B 的错就在它自己的前提里。** C 的空上下文不是缺陷，是它唯一的资产：C 只能看见 PR 里真实存在的东西，而不是 B 声称它做了什么。

| 维度 | 共享上下文的 agent team | Issue 总线上的独立节点 |
|------|------------------------|----------------------|
| 审核独立性 | 无——reviewer 继承实施者的盲区 | 强——只看工件，不看过程 |
| 规模上限 | 一个上下文窗口就是天花板 | 无界，Issue 数量不受窗口限制 |
| 崩溃恢复 | session 挂了上下文全丢 | Issue/PR 都在盘上，换个节点接着做 |
| 并发 | 内部本质是串行对话 | N 个 Executor 真并行 |
| 模型异构 | 通常同一个模型 | 每个节点可用不同模型；换厂商做 review 能交叉出同模型的共同错法 |
| 可审计 | 埋在会话记录里 | 每步都是 Issue comment / PR review，可回溯可统计 |
| 人的位置 | 人 review 每一份产出 | 人 review 这套协议和队列健康度 |

最后一行是真正的收益：**人不该 review 每个 PR，人该 review 这套系统。** 看吞吐（每天流转多少 Issue）、打回率（太高说明 A 的 Issue 写得糙，太低说明 C 在放水）、`blocked` 堆积（哪里断流）、升级频次（哪类分歧反复出现）。指标异常时改的是 Issue 模板和节点 prompt，不是某个 PR。

**agent team 什么时候反而合适**：强耦合、一次性、需要高频来回的探索（比如「这个偶发 bug 到底在哪」），拆成 Issue 的成本高于收益。这时它的正确位置是**某个节点的内部实现**——Researcher 内部开 team 去定位，对总线仍只暴露一个结论 comment。**团队在节点内，协议在节点间。**

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

### 拓扑总览

![Topology Patterns](assets/topology-patterns.svg)

## 多角色协同扩展

### 多对多（Mesh）

```
Planner-A ─┐
Planner-B ─┤→ Issue Bus ←┬── Executor-A
Planner-C ┘              ├── Executor-B
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

## 项目结构

```
multi-agent-workflow/
├── README.md                          # 项目说明（你正在看的这个）
├── SKILL.md                           # WorkBuddy Skill 定义
├── LICENSE                            # MIT License
├── assets/
│   ├── architecture-overview.svg      # 核心架构图
│   ├── issue-state-machine.svg        # Issue 状态机图
│   ├── topology-patterns.svg          # 流水线拓扑图
│   ├── issue-template.md             # Issue 模板（Planner 填写）
│   ├── pr-template.md                # PR 模板（Executor 填写）
│   └── comment-protocol.md           # 各节点 Comment 规范
└── references/
    ├── setup.md                     # 从零搭总线（建 label、配模板、编排节点）
    ├── rationale.md                 # 设计理由与适用边界
    ├── issue-protocol.md            # 标签体系与状态机
    ├── pr-review-protocol.md        # 交付与审核闭环细则
    └── best-practices.md            # 落地检查清单与常见坑
```

## 快速开始

**作为 skill 用（让 agent 照这套模式干活）**

1. 把本仓库放进你的 skill 目录，agent 加载 `SKILL.md`
2. 指派角色：告诉 agent「你是这条流水线的 Executor，总线是 `<repo>`」
3. agent 会自己做开工前检查（确认角色、校验标签齐全），然后扫队列干活

**搭一条流水线**

按 `references/setup.md` 走：建标签 → 放模板到 `.github/` → 配 branch protection 强制审核独立性 → 编排节点 → 放 5 个 Issue 跑一轮看它在哪断。

## 资源

- `SKILL.md`：可加载的技能定义——agent 加载后按角色章节直接干活
- `references/setup.md`：从零搭总线
- `references/rationale.md`：设计理由与适用边界
- `references/issue-protocol.md`：标签体系与状态机
- `references/pr-review-protocol.md`：交付与审核闭环细则
- `references/best-practices.md`：落地检查清单与常见坑
- `assets/issue-template.md`：可直接复制使用的 Issue 模板
- `assets/pr-template.md`：可直接复制使用的 PR 模板
- `assets/comment-protocol.md`：各节点 Comment 规范

## License

MIT
