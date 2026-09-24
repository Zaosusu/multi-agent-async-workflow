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
- repo：`[owner/name]`
- base：`[branch]@[sha]`
- head：`[branch]@[sha / 未开始]`
- build：`[package/version/digest / not-built]`
- runtime config：`[不含密钥的版本或 digest]`
- 证据生成时间：`[timestamp / pending]`

## 优先级
P0 / P1 / P2

## 来源
[`source:human`（真人需求，人类认领/交付/验收）或 `source:ai`（AI 工程任务，Agent 认领/交叉审核）；若执行到某步必须先找人类拍板，另加 `human-only`]

## 标签
`enhancement` `area:xxx` `draft`

## 指定执行者
[用户任命的 Agent 实例，例如 Executor-A；不是 GitHub 用户名。
 若已在 GitHub 上指派，同时写「账号 X 的实例 Y」——`assignee` 只能填账号，工件里记实例。
 注意：指定 ≠ 认领。指定后保留 `ready`，被指定者仍走 claim 协议；不要因「已指定」就置 `in_progress`。
 `source:human`/`human-only` 下：被指定者是真人 ⇒ 指派即已拍板，可省告知；是纯 Agent 账号 ⇒ 仍须先经人类确认。]

## 审核负责人
[用户任命的独立 Agent 实例；不得是本 PR 的代码作者]

## 集成负责人
[用户任命的 Agent 实例；不得合并自己的实现]

## 依赖
- [必须先完成的 Issue；没有写「无」]

## 累计 inherited gates
| Gate ID | 来源 Issue / PR | 适用范围 | 验证命令 / 操作 | 预期信号 | 当前 artifact 证据 | 状态 |
|---------|-----------------|----------|-------------------|----------|---------------------|------|
| `[G-id]` | `[#issue / PR]` | `[component]` | `[command]` | `[expected]` | `[URL/path + SHA]` | `pending/pass/not-applicable` |

## 依赖传播矩阵
| 传播源 | 目标 | 依赖路径 | 受影响组件 | 阻断前状态 | 当前 artifact | 关联 open PR | inherited gate | 证据 | 同步状态 |
|--------|------|----------|------------|------------|-----------------|--------------|----------------|------|----------|
| `[#source / PR]` | `[#target / PR / regression Issue]` | `[#source -> #target]` | `[component]` | `[ready/in_progress/needs-review/approved/未阻断]` | `[branch@sha / build digest]` | `[PR #... / 无]` | `[G-id / not-applicable]` | `[URL/path + SHA / pending]` | `pending/synced/unknown/not-applicable` |

## 非目标 / 禁止事项
- [本 Issue 明确不做的内容]
- [不得顺手扩展的接口、模型、重构或配置]
