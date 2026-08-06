# 落地检查清单与常见坑

## 落地检查清单

- [ ] 确定 GitHub Repository 作为任务总线
- [ ] 建立 Issue 模板（Planner 填写规范）
- [ ] 建立 Comment 规范（各节点反馈规范）
- [ ] 配置 Labels 和 Milestones
- [ ] 配置各节点 Session：Prompt + 触发频率/方式
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
