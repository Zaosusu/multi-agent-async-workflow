# 依赖传播与累计回归门禁

当 Issue 或 PR 改变 shared invariant、公共接口、协议、共享配置、框架行为或前序阶段基线时，读取并执行本协议。普通的叶子任务不需要建立传播事务。

## 1. 定义

- **shared invariant**：按架构职责属于公共边界、跨任务契约或阶段基线的行为，例如启动流程、认证、序列化格式、公共 SDK 封装、错误处理或阶段基线。是否属于 shared invariant 由其适用范围决定，与当前下游数量无关；即使暂时只有一个已知消费者也按本协议传播。
- **当前 artifact**：本轮实际被审核或测试的精确产物。至少由 repo、branch、commit SHA 标识；有构建物时再记录版本、包名和 digest。
- **inherited gate**：由仍适用的上游 shared invariant 产生、每个下游 artifact 都必须重跑的回归检查。
- **传播事务**：从上游变化开始，到所有受影响活动 Issue / open PR 更新完依赖、artifact 和 gate、受影响的 closed / `done` 交付物建立回归 Issue，并留下可审计证据为止的一次同步。

历史日志、截图或旧 APK 只能证明历史 artifact。只要 commit、依赖、构建配置或运行时配置变化，就必须把旧证据标为 `historical`，不得用于批准当前 artifact。

## 2. 唯一认领与主动派工

GitHub Issue 是认领的唯一事实源。聊天、邮件或 Agent 消息可以唤醒节点，但不能覆盖 Issue。

GitHub 不提供跨 comment、label 和 assignee 的原子事务，采用 append-only 的可恢复两阶段协议：

1. 每次尝试生成唯一 `claim-id=<Agent实例>-<UTC时间>-<随机后缀>`。状态机是 `pending -> active`，pending 或 active 都可进入 `failed/abandoned`；`failed` 和 `abandoned` 都是终态，不可回到 active。禁止编辑或删除 lifecycle comment；同一 claim-id 按 GitHub `createdAt`、comment ID 排序并折叠合法转换，最新合法事件决定当前状态，旧的 pending/active comment 不再单独算有效 claim。终态后的任何事件是协议冲突，不改变终态并转 `needs-lead` 留痕。
2. 每个 pending/active 事件必须写 `actor`、`claim-id`、`state`、`lease-until`；pending 还必须写 `activation-grace-until`。项目未另行规定时，pending lease 上限为 10 分钟、activation grace 上限为 lease 到期后 2 分钟、active lease 上限为 2 小时；声明超过上限时按上限计算。pending 的竞选 lease 到期后不再接受新的阶段一动作，但已胜出的 pending 在 activation grace 结束前仍可完成阶段二，scanner 必须视为受保护。active 执行者须在到期前追加新的 `state=active` comment 续租。
3. **阶段一，竞选**：确认 Issue 是 `ready` 且没有未过期 active claim，追加 `state=pending`，不改 assignee/标签；立即重读全部 lifecycle comment。最早创建的未过期 pending 胜出，同时间戳以 claim-id 字典序裁决。
4. 竞争 loser 只追加自己 claim 的 `state=abandoned; reason=lost-race` 并停止。loser 绝不能移除共享的 `ready` / `in_progress`，也不能移除 winner 的 assignee 或其他路由状态。只有 assignee 属于不同 GitHub 账号、操作记录能证明它由 loser 独占写入且 winner 不需要时，loser 才可移除该 assignee；共享账号或归属不明时不动，由 scanner 按 claim lifecycle 恢复。
5. **阶段二，激活**：winner 在 pending lease 与 activation grace 内设置路由 assignee、添加 `in_progress`、移除 `ready`，再追加 `state=active`；这一窗口内出现 `in_progress + pending + 无 active` 是合法中间态。重读确认该 active 事件、标签和 assignee 均可见后，才允许派发 Agent。任一步明确失败时 winner 追加 `state=failed; reason=<step>`；清理前仍须执行第 7 条的双重重读和归属检查。
6. 派发失败时必须追加 `claim-id=<id>; state=abandoned; reason=dispatch-failed`；不得删除旧 comment。随后按第 7 条双重重读，确认没有其他 active/pending winner 且 Issue 仍匹配该 claim 后，才移除失效路由 assignee / `in_progress` 并恢复 `ready`。因此旧 active comment 仍可审计，但不会再次被视为有效。
7. 总负责人每轮先扫描 `in_progress`，再扫描 `ready`。scanner 绝不终结未过期 pending；`in_progress + pending + 无 active` 在 pending lease 和 activation grace 内保持不动。两者都到期后，scanner 第一次读取并记录候选；紧接任何终结或回滚前第二次完整读取 lifecycle、labels、assignees，并记录 `observed-through-comment=<第二次读取的最大comment ID>`。只有两次都没有该 claim 的 active、没有其他 active/pending winner，且 Issue 仍是该 claim 写入的 `in_progress` 阶段二状态，才追加 `state=failed; reason=activation-interrupted; observed-through-comment=<id>`。写后必须第三次完整读取：若发现 comment ID 晚于该快照的 active（无论它排在 failed 前还是后），或 active 在 pending 已终结后追加，均视为迟到写冲突，转 `needs-lead` 且不清任何 assignee/标签；只有不存在这种 active 且终态可确认时，才移除可归属的失效 assignee / `in_progress` 并恢复 `ready`。若任一读取状态不匹配或无法证明归属，也转 `needs-lead` 而不清理。`ready` 上的 pending 同样等 lease/grace 到期并二次重读后才标 `abandoned; reason=lease-expired`，写后也须检查快照后的 active；failed/abandoned 只保留历史、不阻止新 claim。正常 activation grace 内不会终结 pending。
8. 接管只允许在旧 active lease 已过期、没有更晚的续租/进展、扫描者已追加旧 claim 的 abandonment 并重读确认后进行；然后从新的 pending claim 重新开始。旧 Agent 恢复后在任何写入、push 或状态变更前重读 Issue，发现自己的 claim 不再 active 必须停止并交接已有产物。
9. Agent 收到 Issue 之外的直接派工时，必须先核对 Issue；任务已有未过期 active claim则拒绝重复施工。

## 3. 反向枚举影响面

shared invariant 发生实质变化、修复 PR 更新 head 或合入 main 时，由总负责人主持传播：

1. 记录源 Issue、源 PR、当前 commit / artifact 和不变量变化。
2. 用 `gh issue list --state all` 或等价 API 枚举 open、closed 和 `done` Issue；从它们的 `依赖` 字段及 Issue/PR 对源编号的引用开始，列出直接依赖。不得因 Issue 已关闭而跳过已交付依赖。
3. 以直接依赖为下一层继续反向遍历，直到没有新节点；使用 visited 集合防止环，得到全部间接依赖。
4. 对每个受影响 Issue 反查 `Closes` / `Refs` 关联的 open PR，并读取 PR head、base、changed files 和当前审核状态；对受影响的 closed / `done` Issue 标记其已交付 artifact，准备创建新的回归 Issue。
5. 将上游 changed files / 受影响组件与 open PR 文件交叉核对。若发现代码影响但 Issue 没声明依赖，补关系并纳入传播。

把结果写成传播矩阵：

| 传播源 | 目标 | 依赖路径 | 受影响组件 | 阻断前状态 | 当前 artifact | 关联 open PR | inherited gate | 证据 | 同步状态 |
|--------|------|----------|------------|------------|-----------------|--------------|----------------|------|----------|
| `[#source / PR]` | `[#target / PR / regression Issue]` | `[#source -> #target]` | `[component]` | `[ready/in_progress/needs-review/approved/未阻断]` | `[branch@sha / build digest]` | `[PR #... / 无]` | `[G-id / not-applicable]` | `[URL/path + SHA / pending]` | `pending/synced/unknown/not-applicable` |

若 API 权限不足、依赖字段缺失或引用关系无法可靠枚举：

- 对已知受影响的活动下游写 comment，记录阻断前状态后置 `blocked`；对已知受影响的 closed / `done` 交付物创建回归 Issue；
- 源 Issue 打 `needs-lead`，列出已枚举范围和未知部分；
- 不得把“没有找到”写成“没有依赖”，不得宣称传播完成。

## 4. 传播事务

对传播矩阵中的每个目标执行：

1. 在下游 Issue comment 写明源 Issue / PR、变化摘要、适用范围、当前 artifact 身份和新增 gate。
2. 更新下游 Issue 的依赖关系、inherited gates 和传播矩阵行；需要阻断时，先把当时唯一活动状态写入该行的 `阻断前状态`，再改标签。
3. 有 open PR 时，在 PR 留同样的传播 comment，要求更新分支或实现，并在新的 PR head 上重跑累计 gate。
4. `backlog` / `ready` 且未施工的 Issue：更新契约即可，不必改为 `blocked`；若确需阻断，也必须先记录其当前状态。
5. `in_progress`、`needs-review`、`approved` 或已有 open PR 的 Issue：记录阻断前状态，移除其他活动标签并置 `blocked`。该目标同步实现且补齐当前 artifact 证据，使其本行达到 `synced` 或获批 `not-applicable` 后，立即移除该目标的 `blocked` 并恢复本行记录的阻断前状态；不得等待其他目标行，也不得凭当前处理者自行选择 `ready`、`in_progress` 或 `needs-review`。阻断前状态缺失、非法，或因 PR 关闭、契约变化等原因已经过期时，不猜测恢复目标，转 `needs-lead` 由总负责人裁决。
6. 已 closed / `done` 的 Issue 保留历史记录且不改回活动状态；若已交付产物受影响，为它创建新的回归 Issue，把原 Issue、依赖路径、受影响 artifact 和 inherited gates 写入新 Issue，并将新 Issue 作为传播矩阵目标。

源 shared-invariant PR 与下游同步采用单向门禁：源 PR 在传播矩阵的影响面已可靠枚举、依赖路径和目标已验证、所有目标均有行且不存在 `unknown` 后即可按自身验收批准和合并；下游行此时允许仍为 `pending`，不得用这些下游未同步项反向阻塞源 PR。源 PR 合入后，未同步的下游 Issue / PR 继续保持各自行的阻断。

传播解除按行独立提交：每行一达到 `synced/not-applicable` 就 comment 证据并恢复该目标，不受仍为 `pending/unknown` 的兄弟行影响。这样依赖链 `S -> A -> B` 中，A 同步后可先恢复并产出 B 所需 artifact，B 不会因等待全局汇总而死锁。

整个传播事务只有在所有下游矩阵行均为 `synced` 或由总负责人明确判定 `not-applicable`、所有受影响下游 open PR 已更新到包含修复的 head、当前 artifact 的累计 gate 有新证据、不存在 `unknown`，且受影响的 closed / `done` 交付物均已创建所需回归 Issue后，才在源 Issue 留最终汇总并关闭源传播任务。最终汇总只收尾，不再批量解除下游；各目标此前已经逐行恢复。

## 5. 累计 inherited gates

每个 shared invariant gate 使用稳定 ID，例如 `G-10-token-refresh`。下游 Issue 不只继承直接父 Issue，而是继承所有直接和间接上游中仍适用的 gate。

| Gate ID | 来源 | 适用范围 | 验证命令 / 操作 | 预期信号 | 当前 artifact 证据 | 状态 |
|---------|------|----------|-------------------|----------|---------------------|------|
| `G-10-token-refresh` | #10 / PR #18 | auth client | `npm test -- auth` | all pass | `[URL/path + SHA]` | pass |

- Planner 建 Issue 时建立初始累计表。
- Executor 在当前 PR head 上执行自身验收和全部适用 gate。
- Reviewer 核对 gate 来源、适用性、artifact 身份和证据，不接受历史候选替代。
- Integrator 在合并后 main artifact 上重跑全部适用 gate。
- gate 只有经总负责人 comment 说明不再适用并更新传播矩阵后才能移除，不能由下游 Executor 自行删除。

## 6. 最小 artifact 身份

代码 PR 至少记录：

```text
repo=<owner/name>
base=<branch>@<sha>
head=<branch>@<sha>
build=<package/version/digest 或 not-built>
runtime-config=<不含密钥的版本或 digest>
evidence-generated-at=<timestamp>
```

证据必须能反查到这组身份。无法证明证据属于当前 artifact 时，状态是 `unknown`，不是 `pass`。
