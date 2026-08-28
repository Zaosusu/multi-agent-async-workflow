# 各节点 Comment 规范

## Planner

- 新建 Issue：`📋 新建 Issue #[number]，背景：xxx`
- 更新需求：`📝 更新 Issue #[number]：xxx`
- 分配主体：`📌 任命：执行主体 xxx；审核主体 yyy；集成主体 zzz`
- 认领竞选：`🔒 claim-id=<id>; state=pending; actor=<Agent实例>; lease-until=<UTC>; activation-grace-until=<UTC>`
- 认领激活：`🔒 claim-id=<id>; state=active; actor=<Agent实例>; lease-until=<UTC>`（该 comment、assignee 和 `in_progress` 均可见后才派 Agent）
- 认领续租：`🔒 claim-id=<id>; state=active; actor=<Agent实例>; lease-until=<UTC>; reason=renewal`
- 放弃/回收：`⚠️ claim-id=<id>; state=abandoned; reason=<lost-race/dispatch-failed/lease-expired/voluntary>`
- 激活失败：`⚠️ claim-id=<id>; state=failed; reason=<mutation-step/activation-interrupted>; observed-through-comment=<id>`（scanner 只有 lease/grace 到期且双重重读仍无 active、Issue 仍匹配时才写；写后重读发现快照后 active 则转 needs-lead 且不清状态；loser 不清共享标签）
- 传播开始：`📡 shared invariant 已变化：源 #x / PR #y / artifact sha；已记录各目标阻断前状态，开始同步 #a、#b 与 PR #c；未同步下游 blocked，不反向阻塞已验证矩阵的源 PR`
- 单行同步完成：`✅ 目标 #x 已 synced/not-applicable；证据 [链接/路径]；现立即恢复阻断前状态 [state]，不等待其他行`
- 传播汇总完成：`✅ 全部传播行完成：直接/间接目标及 closed/done 回归 Issue [列表]；当前 artifact [身份]；累计 gates [列表]；unknown=0；关闭源传播任务`
- 无法枚举：`⬆️ 依赖枚举不完整：已知范围 xxx，未知原因 yyy；已知下游 blocked，源 Issue 转 needs-lead`

## Researcher

- 研究完成：`🔍 研究完成：结论是 xxx，建议方案：yyy`
- 需澄清：`❓ 关于 [某点]，需要 Planner 确认：xxx`
- 产出子 Issue：`📋 基于研究，新建子 Issue #[number]`

## Executor

- 开始处理：`👋 开始处理，预计 [时间]`
- 完成提交：`✅ 已提交 PR #[number]`（同时把标签换成 `needs-review`；PR body 必须写 `Closes #[issue]`）
- 打回后重提：`🔄 已按 review 意见修正，PR #[number] 请复审`
- 同步上游基线：`🔄 已同步源 #x / PR #y 到当前 head [sha]；inherited gates [列表] 已在当前 artifact 重跑，证据：[链接/路径]`
- 遇到阻塞：`🚧 阻塞原因：xxx，需要 @xxx 确认`
- 需要澄清：`❓ 关于 [某点]，能否确认 [具体问题]`

## Reviewer

- 通过：`审核主体：xxx；实现主体：yyy；👍 审核通过，可以合并`（Issue 转 `approved`）
- 打回：`审核主体：xxx；实现主体：yyy；🚫 打回（第 N 次）：具体问题 xxx，请修正后重新提交`（问题要具到文件和行，Issue 回 `in_progress`）
- 建议改进：`💡 建议改进：xxx（不影响通过）`
- 超出范围：`↩️ 这一点超出本 Issue 验收标准，已建议 Planner 开新 Issue，本 PR 不因此阻塞`
- 升级：`⬆️ 同一 PR 已打回 2 次，分歧在验收标准本身，打 needs-lead 升级`
- 传播阻断：`🚫 下游依赖传播未完成：目标/PR xxx 仍为 pending/unknown，或证据属于历史 artifact；阻断前状态 [state] 已记录，Issue 转 blocked，禁止批准该下游 PR`

## Integrator

- 合并完成：`集成主体：xxx；🔗 已合并 PR #[a]；合并前 main: [sha]；PR head: [sha]；合并后 main: [sha]；验证：`[command]` 通过`（移除活动标签，添加 `done`）
- 集成失败：`❌ 集成测试失败：xxx，请修复`
- 累计门禁失败：`❌ 当前 main artifact [sha/build] 的 inherited gate [G-id] 失败：xxx；Issue 回 in_progress`

## Human

- 确认：`✅ 已确认：xxx`
- 拒绝：`❌ 不同意，理由：xxx`
- 决策：`📌 决策：xxx`
