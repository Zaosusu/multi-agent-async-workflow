# Issue 协议规范

为了让多节点异步协作，Issue 本身需要承载足够信息。

## Issue 模板（Planner 填写）

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

## 当前 artifact 身份
- repo / branch / commit：...
- build 版本 / digest：...
- 证据生成时间：...

## 优先级
P0 / P1 / P2

## 标签
`enhancement` `area:xxx` `draft`

## 指定执行者
Executor-A / Executor-B / 由总负责人认领

## 审核负责人
Reviewer-A / 总负责人

## 集成负责人
Integrator-A / 总负责人

## 依赖
- #123 (必须先完成)
- #456 (可并行)

## 累计 inherited gates
| Gate ID | 来源 | 适用范围 | 验证方式 | 当前 artifact 证据 | 状态 |
|---------|------|----------|----------|---------------------|------|
| `G-123-example` | #123 / PR #789 | xxx | `command` | URL / path + SHA | pending |

## 依赖传播矩阵
| 传播源 | 目标 | 依赖路径 | 受影响组件 | 阻断前状态 | 当前 artifact | 关联 open PR | inherited gate | 证据 | 同步状态 |
|--------|------|----------|------------|------------|-----------------|--------------|----------------|------|----------|
| `[#source / PR]` | `[#target / PR / regression Issue]` | `[#source -> #target]` | `[component]` | `[ready/in_progress/needs-review/approved/未阻断]` | `[branch@sha / build digest]` | `[PR #... / 无]` | `[G-id / not-applicable]` | `[URL/path + SHA / pending]` | `pending/synced/unknown/not-applicable` |

## 非目标 / 禁止事项
- [本 Issue 明确不做的内容；发现相关工作时新开 Issue，不在本 PR 顺手扩展]
```

## 节点 Comment 规范

| 节点 | Comment 格式 |
|------|-------------|
| Planner | `📋 新建 Issue #[number]，背景：xxx` |
| Researcher | `🔍 研究完成：结论是 xxx，建议方案：yyy` |
| Executor | `👋 开始处理，预计 [时间]` / `✅ 已提交 PR #[number]` |
| Reviewer | `👍 审核通过，可以合并` / `🚫 打回：具体问题 xxx` |
| Integrator | `🔗 已合并 PR #[a]，main 验证通过` |
| Human | `✅ 已确认：xxx` / `❌ 不同意，理由：xxx` |
| 任意节点 | `🚧 阻塞原因：xxx，需要 @xxx 确认` |

## 标签体系

| 标签 | 含义 | 自动流转 |
|------|------|----------|
| `backlog` | 待规划 | → `ready` |
| `ready` | 待分配 | → `in_progress` |
| `in_progress` | 实施中 | → `needs-review` / `blocked` |
| `needs-research` | 需研究 | → `ready` |
| `needs-review` | 待审核 | → `approved` / `in_progress` |
| `approved` | 审核通过，待集成负责人合并和 main 验证 | → `done` / `in_progress` |
| `blocked` | 阻塞；传播阻断进入前必须在矩阵记录唯一的阻断前状态 | → 传播阻断恢复记录状态（缺失、非法或已过期则 → `needs-lead`）；其他阻塞解除后 → `ready` |
| `needs-lead` | 需总负责人裁决（规格/优先级/方案分歧） | → `ready` / `in_progress` |
| `needs-human` | 需真人决定（花钱/对外承诺/法律权限/业务方向） | → `ready`（决策后） |
| `done` | 已完成 | 终态 |

> `needs-lead` 与 `needs-human` 不可合并成一个：前者总负责人自己拍板，后者必须到真人。混用会导致本该一句话解决的分歧堆着等人，或 AI 替真人做了它不该做的决定。

> 打 `needs-review` 的前提是 Issue 下已有 `✅ 已提交 PR #N` 的 comment，且该 PR body 写了 `Closes #<issue>`。
> 没有关联 PR 的 `needs-review` 视为无效，Reviewer 应直接退回 `in_progress`。
> `done` 只能由集成负责人在 PR 合并且 `main` 验证通过后设置。GitHub 因 `Closes #N` 自动关闭 Issue 时，也必须补齐标签和验证 comment，不能把自动关闭当作验收完成。
> `指定执行者`、`审核负责人`、`集成负责人`记录的是用户任命的 Agent 实例，不依赖 GitHub 账号是否不同。
> 认领以 Issue 中 append-only 的 claim lifecycle comment 为身份事实源，assignee/标签只负责路由。状态机是 `pending -> active`，pending 或 active 可进入终态 `failed/abandoned`；只有最新合法事件为 active 且 lease 未过期的 claim 才表示已认领。未过期 pending 及其 activation grace 是合法激活窗口，scanner 不得清理；竞争、派发失败、过期回收与接管执行 `dependency-propagation.md` 的双重重读规则，旧 comment 不删除。
> shared invariant 变化时，从 open、closed、`done` Issue 枚举直接/间接依赖。源 PR 在影响面矩阵验证后可先合并；仅未同步的活动下游记录阻断前状态后置 `blocked`，受影响的已交付物另开回归 Issue。完整协议见 `dependency-propagation.md`。
> inherited gates 必须在当前 artifact 上产生证据。旧 commit、旧构建或旧配置的证据标为 `historical`，不能填 `pass`。

## 交付与审核

Issue 之后的半程（Executor 交付什么、谁来审、打回怎么走、为什么 Reviewer 必须是独立 session）见 `pr-review-protocol.md`。
