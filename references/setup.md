# 从零搭一条流水线

## 1. 选定总线 repo

可以是代码仓库本身，也可以是独立的任务仓库（跨多个代码库时用后者）。

## 2. 建标签

```bash
gh label create backlog            --color ededed --description "待规划"
gh label create ready              --color 0e8a16 --description "待认领"
gh label create in_progress        --color fbca04 --description "实施中"
gh label create needs-research     --color 1d76db --description "需调研"
gh label create needs-review       --color d93f0b --description "待审核"
gh label create needs-clarification --color d4c5f9 --description "验收标准不清，退回 Planner"
gh label create blocked            --color b60205 --description "阻塞"
gh label create needs-lead         --color 5319e7 --description "需总负责人裁决（规格/优先级/方案分歧）"
gh label create needs-human        --color e11d21 --description "需真人决定（花钱/对外承诺/法律权限/业务方向）"
gh label create approved           --color 0e8a16 --description "已批准，待合并"
```

`needs-lead` 和 `needs-human` **必须分开**：前者总负责人自己就能拍板，后者必须到真人。合成一个的后果是两头都错——本该一句话解决的分歧堆着等真人，或者 AI 替你决定了它不该决定的事。

## 3. 放模板到 GitHub 认的位置

`assets/` 下的模板是给人看的副本，**GitHub 只认这两个路径**：

```bash
mkdir -p .github/ISSUE_TEMPLATE
cp assets/issue-template.md .github/ISSUE_TEMPLATE/task.md
cp assets/pr-template.md .github/pull_request_template.md
```

`.github/ISSUE_TEMPLATE/task.md` 需要在开头加 front matter：

```markdown
---
name: 流水线任务
about: Planner 创建的可执行任务
labels: backlog
---
```

## 4. 建立审核独立性与任命记录（关键）

「实施者不能批自己的 PR」如果只写在 prompt 里，就只是自觉。让它变成硬约束：

1. **按 Agent 实例任命角色**：用户可以让多个 Agent 共用一个 GitHub 账号。每个可执行 Issue 记录 `指定执行者`、`审核负责人`、`集成负责人`；这些字段写 Agent 实例名，不写 GitHub 用户名。
2. **禁止同一 Agent 实例审核或合并自己的实现**：GitHub 账号相同不代表自审，代码作者才是边界。
3. **同账号模式用工件留痕**：GitHub 可能不允许同账号发原生 approval/request-changes；Reviewer 用 PR 评论写明审核主体和结论，Issue 标签按 `needs-review -> approved -> done` 流转。
4. **branch protection 是可选增强**：有独立 GitHub 身份时，可要求 approval、CI 和禁止 force push；共用账号时不能把原生 approval 作为协议的必经条件，仍必须要求 CI 和禁止直接推送 main。

工件中的任命、评论、标签和 main 验证共同形成可追溯闭环；GitHub 身份只是可选的额外门禁。

## 5. 编排节点

最小可用配置是三个节点：

| 节点 | 数量 | 建议 |
|------|------|------|
| 总负责人（Planner + Reviewer + Integrator） | 1 | 用推理强的模型。它定规格、验收、持合并权，决定整条流水线的质量上限 |
| Executor | 1-N | 用代码能力强 / 长上下文的模型；N 决定并发度 |
| （可选）review 负责人 | 0-1 | 由总负责人任命。**换一家的模型**——同模型容易犯同样的错，交叉验证才有价值 |

最小可用配置就是 **1 个总负责人 + 1 个 Executor**。Researcher / 独立 Reviewer 在需要时再加。

真人是必需的，但只在两处投入：**① 底层框架期逐项验收骨架**（框架的错会被每个后续任务继承，且抽查结果发现不了）；**② 框架通过后只看交付结果**，不看 PR。另外要有人扫 `needs-human` 队列。

## 6. 驱动方式

逻辑模型是「看队列 → 处理 → 流转 → 再看」的循环，但实现上有三种：

| 方式 | 做法 | 取舍 |
|------|------|------|
| **定时单次 pass**（推荐起步） | cron 每 N 分钟唤醒一次，跑一个 pass 后退出 | 空队列不烧 token，进程崩了下一轮自动复活。延迟 = 轮询间隔 |
| 常驻循环 | 进程内无限循环，处理完继续扫 | 响应快；但空转有成本，且崩了不会自愈 |
| Webhook 触发 | Issue labeled / PR opened 时唤醒对应节点 | 实时；需要部署 receiver |

起步用第一种。它把「崩溃恢复」从需要设计的问题变成了免费属性。

## 7. 补存活性（真跑起来才会遇到）

- **回收死任务**：Executor 崩在 `in_progress`，任务会永远躺着。用一个 GitHub Action 定时扫「`in_progress` 且 N 小时无新 comment」→ 退回 `ready`。**用 Action 而不是 agent**，因为 agent 也会死，而没人回收回收者。
- **熔断**：CI 全局坏掉时，Executor 会持续生产注定过不了的 PR。给节点加一条前置检查：main 的 CI 是红的就停下报告。

## 8. 跑一轮验证

放 5 个 Issue，看它在哪断。重点不是产出，是观察：

- 有没有任务卡在某个标签上没人接（→ 节点扫描条件写错了）
- 打回率是多少（太高 = Issue 写得糙；太低 = Reviewer 在放水）
- 有没有出现双份 PR（→ 认领协议没生效）
- 平均几轮能合一个 PR

这些数字才是调 prompt 的依据。
