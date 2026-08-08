# 各节点 Comment 规范

## Planner

- 新建 Issue：`📋 新建 Issue #[number]，背景：xxx`
- 更新需求：`📝 更新 Issue #[number]：xxx`
- 关闭 Issue：`✅ 已关闭，需求已实现`

## Researcher

- 研究完成：`🔍 研究完成：结论是 xxx，建议方案：yyy`
- 需澄清：`❓ 关于 [某点]，需要 Planner 确认：xxx`
- 产出子 Issue：`📋 基于研究，新建子 Issue #[number]`

## Executor

- 开始处理：`👋 开始处理，预计 [时间]`
- 完成提交：`✅ 已提交 PR #[number]`（同时把标签换成 `needs-review`；PR body 必须写 `Closes #[issue]`）
- 打回后重提：`🔄 已按 review 意见修正，PR #[number] 请复审`
- 遇到阻塞：`🚧 阻塞原因：xxx，需要 @xxx 确认`
- 需要澄清：`❓ 关于 [某点]，能否确认 [具体问题]`

## Reviewer

- 通过：`👍 通过，可以合并`
- 打回：`🚫 打回（第 N 次）：具体问题 xxx，请修正后重新提交`（问题要具到文件和行）
- 建议改进：`💡 建议改进：xxx（不影响通过）`
- 超出范围：`↩️ 这一点超出本 Issue 验收标准，已建议 Planner 开新 Issue，本 PR 不因此阻塞`
- 升级：`⬆️ 同一 PR 已打回 2 次，分歧在验收标准本身，打 needs-human 升级`

## Integrator

- 合并完成：`🔗 已合并 PR #[a] 和 #[b]，集成测试通过`
- 集成失败：`❌ 集成测试失败：xxx，请修复`

## Human

- 确认：`✅ 已确认：xxx`
- 拒绝：`❌ 不同意，理由：xxx`
- 决策：`📌 决策：xxx`
