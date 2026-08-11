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

## 节点 Comment 规范

| 节点 | Comment 格式 |
|------|-------------|
| Planner | `📋 新建 Issue #[number]，背景：xxx` |
| Researcher | `🔍 研究完成：结论是 xxx，建议方案：yyy` |
| Executor | `👋 开始处理，预计 [时间]` / `✅ 已提交 PR #[number]` |
| Reviewer | `👍 通过` / `🚫 打回：具体问题 xxx` |
| Integrator | `🔗 已合并 PR #[a] 和 #[b]，集成测试通过` |
| Human | `✅ 已确认：xxx` / `❌ 不同意，理由：xxx` |
| 任意节点 | `🚧 阻塞原因：xxx，需要 @xxx 确认` |

## 标签体系

| 标签 | 含义 | 自动流转 |
|------|------|----------|
| `backlog` | 待规划 | → `ready` |
| `ready` | 待分配 | → `in_progress` |
| `in_progress` | 实施中 | → `needs-review` / `blocked` |
| `needs-research` | 需研究 | → `ready` |
| `needs-review` | 待审核 | → `done` / `in_progress` |
| `blocked` | 阻塞 | → `ready`（解除后） |
| `needs-lead` | 需总负责人裁决（规格/优先级/方案分歧） | → `ready` / `in_progress` |
| `needs-human` | 需真人决定（花钱/对外承诺/法律权限/业务方向） | → `ready`（决策后） |
| `done` | 已完成 | 终态 |

> `needs-lead` 与 `needs-human` 不可合并成一个：前者总负责人自己拍板，后者必须到真人。混用会导致本该一句话解决的分歧堆着等人，或 AI 替真人做了它不该做的决定。

> 打 `needs-review` 的前提是 Issue 下已有 `✅ 已提交 PR #N` 的 comment，且该 PR body 写了 `Closes #<issue>`。
> 没有关联 PR 的 `needs-review` 视为无效，Reviewer 应直接退回 `in_progress`。

## 交付与审核

Issue 之后的半程（Executor 交付什么、谁来审、打回怎么走、为什么 Reviewer 必须是独立 session）见 `pr-review-protocol.md`。
