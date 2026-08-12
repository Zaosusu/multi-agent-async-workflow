# 设计理由

这一章不是操作指南，是回答「为什么这么设计」——用于评估这套架构是否适合你的场景。

## 核心模式

**Issue 在这里不是 bug tracker，是跨 session 的持久化消息队列。**

每个节点只关心自己负责的状态转换。节点之间不共享上下文、不互相调用、不需要中心调度器，它们只做一件事：看队列 → 认领 → 处理 → 流转状态 → 再看。

```
backlog → ready → in_progress → needs-review → approved → done
            ↖        ↓              ↓             ↓
             ← 打回 ─┴─ blocked / needs-lead / needs-human
                                      集成失败 → in_progress
```

## 上下文靠工件传递，不靠 session

这是整套架构的核心机制。多 Agent 协作的常见直觉是「怎么把上下文传给下一个 Agent」，这里的答案是**不传，外化成工件**：

```
Issue  ──承载──→  why / what / 验收标准    = 契约
  ↕ 双向关联（Closes #N）
 PR    ──承载──→  how / diff / CI / 自检    = 交付
```

Reviewer 的输入 = PR diff + 反查到的 Issue，足够独立判定「是否满足验收标准」。契约已经外化，所以不需要共享上下文。

审核独立性以用户任命的 Agent 实例为边界，不以 GitHub 登录名为边界。因此多个 Agent 可以共用一个 GitHub 账号；同一实例仍不能审核或合并自己的实现。

推论：**你没写在 Issue / PR 上的东西等于不存在。** 下游节点是全新的上下文，看不见你的思考过程。

## 为什么不用共享上下文的 agent team

**因为审核的价值来自独立性，而共享上下文恰好摧毁独立性。**

一个继承了实施者全部上下文的 reviewer，已经读过对方的推理链并被它说服了。它会把对方的假设当前提、把对方的取舍当既定事实——**它抓不到对方的错，因为错就在它自己的前提里。** 空上下文不是缺陷，是审核者唯一的资产：它只能看见 PR 里真实存在的东西，而不是实施者声称自己做了什么。

| 维度 | 共享上下文的 agent team | Issue 总线上的独立节点 |
|------|------------------------|----------------------|
| 审核独立性 | 无——reviewer 继承实施者的盲区 | 强——只看工件，不看过程 |
| 规模上限 | 一个上下文窗口就是天花板 | 无界，Issue 数量不受窗口限制 |
| 崩溃恢复 | session 挂了上下文全丢 | Issue/PR 都在盘上，换个节点接着做 |
| 并发 | 内部本质是串行对话 | N 个 Executor 真并行 |
| 模型异构 | 通常同一个模型 | 每节点可换模型；换厂商 review 能交叉出同模型的共同错法 |
| 可审计 | 埋在会话记录里 | 每步都是 comment / review，可回溯可统计 |
| 人的位置 | 人 review 每一份产出 | 人 review 协议和队列健康度 |

最后一行是真正的收益：**人不该 review 每个 PR，人该 review 这套系统。** 看吞吐、打回率（太高说明 Issue 写得糙，太低说明 Reviewer 在放水）、`blocked` 堆积、升级频次。指标异常时改的是 Issue 模板和节点 prompt，不是某个 PR。

### agent team 什么时候反而合适

强耦合、一次性、需要高频来回的探索——比如「这个偶发 bug 到底在哪」，拆成 Issue 的成本高于收益。

这时它的正确位置是**某个节点的内部实现**（Researcher 内部开 team 去定位），对总线仍只暴露一个结论 comment。**团队在节点内，协议在节点间。**

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

**Issue 是代码和需求之间的「标准化接口」。**

## 为什么 Executor 的交付物必须是 PR

| 属性 | 说明 |
|------|------|
| 可审核 | diff 有明确边界，Reviewer 知道审什么 |
| 可自动化 | CI 天然挂在 PR 上，跑绿了才轮到审查 |
| 可回滚 | 一个 PR 一个 squash commit，出事直接 revert |
| 可关联 | 原生关联 Issue、Commit、CI、Review 记录——这些就是上下文本身 |
| 可并发 | N 个 Executor 各开各的分支，互不覆盖 |

## 流水线拓扑

### 线性
```
Planner → Researcher → Executor → Reviewer → Integrator → Done
```
需求明确、流程标准、需要多道质量关卡。

### DAG
```
Planner ├── Issue-A → Executor-A → Reviewer-A
        └── Issue-B → Researcher → Executor-B → Reviewer-B
```
并行任务多，部分需研究、部分可直接实施。

### 环形反馈
```
Executor → Reviewer → [打回] → Executor
                    → [通过] → Done
```
质量要求高，允许迭代修正。**必须配升级阈值**，否则不收敛。

### 扇入/扇出
```
Planner ├── Issue-1 → Executor-A ┐
        ├── Issue-2 → Executor-B ├→ Integrator
        └── Issue-3 → Executor-C ┘
```
一个 Epic 拆成多个子任务，最后需要集成。

### 质量门控
```
Executor → [CI] → [绿] → Reviewer → [通过] → Integrator
              ↓                  ↓
           [红] → Executor    [打回] → Executor
```
CI 作为第一道关卡，减少审核负担。

### 按领域拆分
```
Planner（技术债）  → Executor（后端） → Reviewer（后端）
Planner（业务需求）→ Executor（前端） → Reviewer（前端）
```
不同领域用不同 milestone 或 repo 隔离，避免交叉干扰。

## 这套架构不适合什么

- **需要实时来回的任务**：轮询延迟 + 工件化开销，比直接对话慢得多
- **一次性小改动**：写 Issue 的成本高于改代码本身
- **无法写出可判定验收标准的需求**：整套系统的正确性都压在验收标准上，标准含糊则全线失效
