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

- 扫描分配给自己的 Issue
- 按描述实现，不超出范围
- 提交 PR 并关联 Issue，遵循 `assets/comment-protocol.md`

#### Reviewer

- 扫描带有 `needs-review` 标签的 PR
- 审查通过则关闭 Issue；打回则 comment 具体问题
- 关注：正确性、边界条件、测试覆盖、规范

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
- `references/best-practices.md`：落地检查清单与常见坑
- `assets/issue-template.md`：可直接复制使用的 Issue 模板
- `assets/comment-protocol.md`：各节点 Comment 规范
