# 落地检查清单与常见坑

## 落地检查清单

- [ ] 确定 GitHub Repository 作为任务总线
- [ ] 建立 Issue 模板（Planner 填写规范）
- [ ] 建立 PR 模板（Executor 填写规范，含 `Closes #<issue>`）
- [ ] 建立 Comment 规范（各节点反馈规范）
- [ ] 配置 Labels 和 Milestones
- [ ] 配置各节点 Session：Prompt + 触发频率/方式
- [ ] 确认 Reviewer 与 Executor 是独立的 **Agent 实例**（不共享实施推理）；GitHub 账号可共用
- [ ] 在 Issue/PR 记录执行主体、审核主体、集成主体
- [ ] 明确「指定执行者」= 编排（`assignee`，保留 `ready`）≠ 认领（claim + `in_progress`）；被指定者仍走 claim 协议
- [ ] `source:human` / `human-only` 指定执行者时，先判被指定者是不是真人：真人 ⇒ 可省告知；纯 Agent 账号 ⇒ 仍须先经人类确认
- [ ] 设定打回升级阈值（同一 PR 打回 N 次 → `needs-lead`；只有真人权限事项才 `needs-human`）
- [ ] 将 `approved` 纳入标签和状态机，规定 Integrator 在 main 验证后才设置 `done`
- [ ] 为合并建立新鲜度闸门：记录 main SHA 与 PR head SHA，检查 GitHub mergeable/CI；main 前进时在更新分支或合并结果上重跑受影响验证
- [ ] 为 shared invariant 建立反向依赖传播矩阵，覆盖 open、closed、`done` 的直接/间接 Issue、受影响组件、已交付 artifact 和 open PR
- [ ] 为每个下游 Issue 累计继承仍适用的上游 gate，并绑定当前 artifact 身份
- [ ] 总负责人每轮扫描 `in_progress` / `ready` claim；pending 在 activation grace 内不回收，过期后双重重读仍无 active 且状态匹配才清理；active 可见后才派 Agent
- [ ] 确定流水线拓扑（线性 / DAG / 环形 / 扇入扇出）
- [ ] 设定初始 backlog（第一批 Issue）
- [ ] 跑一轮验证：观察 Issue 流转是否顺畅
- [ ] 根据实际阻塞点调整 Issue 模板和 Prompt

## 常见坑

1. **Issue 太粗糙** → 实施侧无法直接动手，反复 comment 确认，变成同步模式。**解法：Planner 必须写清楚验收标准。**
2. **Issue 太大** → 一个 Issue 要做三天，阻塞整个流水线。**解法：拆分到 1-2 小时可完成粒度。**
3. **没有退出条件** → 实施侧过度设计。**解法：Issue 里明确说「只做 xxx，不做 yyy」。**
4. **优先级混乱** → 贪多，什么都接。**解法：Planner 控制 Issue 流出速度，一次最多 N 个 in_progress。**
5. **节点职责不清** → 多个节点同时处理同一个 Issue，互相覆盖或冲突。**解法：用 Labels + Assignee 明确每个 Issue 的当前负责人。**
6. **阻塞无人处理** → 实施侧卡住后没人管，Issue 一直 in_progress。**解法：设置自动提醒或 Human 定期扫 blocked 队列。**
7. **Reviewer 成为瓶颈** → 所有 PR 堆积在 needs-review，Executor 等待。**解法：并行 Review，或按优先级分批处理。**
8. ** Researcher 产出无法落地** → 研究报告写完没人接。**解法：Researcher 产出必须转成具体的可执行 Issue。**
9. **PR 不关联 Issue** → Reviewer 拿到一堆无从判定对错的 diff，只能凭感觉审。**解法：PR body 必须写 `Closes #<issue>`，没写的直接打回，不进入 review。**
10. **只开 PR 不换标签** → Reviewer 的 loop 扫不到，任务静默死在 `in_progress`。**解法：把「换标签」写进 Executor 的收尾动作，和开 PR 同等强制。**
11. **让实施侧自审** → 写代码的人既写又审，等于自己批自己的作业，抓不到自己的盲区。**解法：Reviewer 不能是这个 PR 的代码作者。注意这条只约束代码作者——总负责人（Planner）审别人实现的 PR 完全合法，把它外推成「规格作者不能审」会让「1 个 lead + N 个 Executor」这种最常见配置直接死锁，没人能合并任何 PR。**
12. **Reviewer 在 review 里加需求** → PR 永远合不进去，Executor 陷入无限修改。**解法：判据是「问题在不在这个 diff 里」，不是「验收标准有没有写」——diff 内部的正确性问题/回归/安全缺陷该打回（哪怕标准没提），想让 PR 多做一件事则回 Planner 开新 Issue。可以拒收坏的实现，不可以扩大要求。**
13. **打回死循环** → B 和 C 在「我觉得可以了 / 我觉得还不行」之间无限对打，烧 token 不收敛。**解法：同一 PR 打回 2 次后打 `needs-lead`；只有花钱、对外承诺、法律权限、业务方向或外部信息才转 `needs-human`。**
14. **人去 review 每个 PR** → 人重新变成瓶颈，多智能体的吞吐收益归零。**解法：人 review 这套系统而不是单个产出——看吞吐、打回率、`blocked` 堆积、升级频次；指标异常时改 Issue 模板和节点 prompt。**
15. **用共享上下文的 agent team 替代总线** → 上下文窗口成为规模天花板，且审核独立性丧失。**解法：team 只作为单个节点的内部实现（如 Researcher 内部定位问题），对总线仍只暴露一个结论 comment。团队在节点内，协议在节点间。**
16. **把 GitHub 账号当作 Agent 身份** → 同一用户控制多个 Agent 时会误判自审，或因无法原生 approval 卡死。**解法：以用户任命的 Agent 实例为边界；用 PR 评论、Issue 标签和 main 验证留痕，原生 review 仅作可选增强。**
17. **Reviewer 通过就关 Issue** → PR 还未真正进入 main，或 main 上集成失败却被宣告完成。**解法：`needs-review -> approved -> done`；Integrator 合并前检查 main 新鲜度，合并后在 main 验证并收尾标签。**
18. **仅靠 SSH ref 报告 GitHub 状态** → 看见分支却误报 Issue、PR、CI 已检查。**解法：GitHub CLI/API 不可用时只报告远程分支和提交观察，不推断标签、评论、PR 状态或 CI。**
19. **main 前进后仍按旧 PR 结论合并** → 旧 diff 虽然曾经通过，但与新 main 的组合未经验证。**解法：合并前记录 main SHA 与 PR head SHA，确认 GitHub `mergeable` 为可合并且状态 clean；main 已变化时更新分支或验证合并结果。**
20. **审核只看功能、不逐文件看范围** → 未授权的数据模型、接口、配置或重构被作为“额外工作”带入。**解法：Reviewer 逐文件对照 Issue 的涉及范围和非目标；越界内容删除或另开 Issue，不以范围外产出换取通过。**
21. **共享修复只改源 Issue** → 直接依赖看似更新了，间接依赖、closed / `done` 交付物和已有 open PR 仍按旧基线运行。**解法：把修复当成传播事务，遍历 open/closed 完整依赖图并交叉核对受影响文件；源 PR 在影响面矩阵验证后可先合并，只阻止未同步下游批准和合并，受影响的已交付物另开回归 Issue。**
22. **Phase-N 只测本阶段** → 前序能力在新 artifact 中退化，但旧阶段曾通过被误当成当前结论。**解法：每个下游 Issue 累计继承全部仍适用 gate，并在当前 commit/build/config 上重跑；历史证据明确标为 `historical`。**
23. **聊天派工和 Issue 认领并存** → 用户与总负责人各派一个 Agent，两个实现同时修改同一任务。**解法：GitHub Issue 是唯一认领源；使用 append-only 的 `pending -> active -> failed/abandoned` 两阶段 claim，active 可见后才启动 Agent。直接消息接单前也必须查 Issue。**
24. **崩溃留下假锁** → pending、assignee 或 `in_progress` 写了一半，扫描器可能永久等待，也可能误杀正在激活的 winner。**解法：claim 带 pending lease 与 activation grace；grace 内不回收，过期后双重重读仍无 active 且状态匹配才终结；旧记录不删除。**
25. **依赖关系不可见却假定传播完成** → API 权限不足或 Issue 没写依赖时，“搜不到”被误当成“没有”。**解法：记录已枚举范围，把已知下游置 `blocked`，源 Issue 打 `needs-lead`；补齐依赖图前不得宣称完成。**
26. **解除传播阻断时凭感觉选状态** → 原本待审核的 PR 被退回 `ready`，或已失效的批准被直接恢复。**解法：置 `blocked` 前在矩阵记录唯一的阻断前状态；同步后只恢复该状态，缺失、非法或已过期时转 `needs-lead` 裁决。**
27. **把「指定执行者」当成「已认领」** → 总负责人写了 `指定执行者` 就置 `in_progress`，或反过来以为指定了就不用认领。**解法：指定（编排）与认领（成交）分开落——`ready`/`in_progress`/`assignee` 三者正交；指定后保留 `ready`，被指定者仍走 claim 协议；工件写 Agent 实例名、`assignee` 填 GitHub 账号，两个都记。**
28. **指定了真人却仍走一遍「告知人类」** → 被指定者本身就是真人，还让他等一轮确认。**解法：被指定者是真人（本人账号）⇒ 指派即代表真人已拍板接活，可省告知；被指定者是纯 Agent 账号（背后无真人）⇒ 仍须补足「先经人类确认 + 告知接活/验收」。先判是不是真人，别只看 `--add-assignee` 成功。**
