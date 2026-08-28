Closes #[issue-number]

## 主体
- 实现主体：[Agent 实例]
- 审核主体：[预期审核 Agent 实例]
- 集成主体：[预期集成 Agent 实例]

## 变更文件范围
- `[path/to/file]`：[变更原因]

## 改了什么
[一两句说清这个 PR 做的事，不要复述 Issue]

## 当前 artifact 身份
- repo：`[owner/name]`
- base：`[branch]@[sha]`
- head：`[branch]@[sha]`
- build：`[package/version/digest / not-built]`
- runtime config：`[不含密钥的版本或 digest]`
- 证据生成时间：`[timestamp]`

## 验收标准自检
> 逐条对应 Issue 里的验收标准，不要新增也不要漏

- [x] 测试用例 A 通过（`npm test -- xxx`）
- [x] 不影响现有功能 B（手动验证：xxx）

## 累计 inherited gates
| Gate ID | 来源 | 当前 head 验证方式 | 结果 / 证据 |
|---------|------|-------------------|-------------|
| `[G-id]` | `[#issue / PR]` | `[command/operation]` | `pass: [URL/path + head SHA]` |

## 依赖传播状态
- 本 PR 是否修改 shared invariant：`[是 / 否]`
- 传播源 Issue / PR：`[#... / 无]`
- 本 PR 角色：`[源 shared-invariant PR / 下游 PR / 不适用]`

| 传播源 | 目标 | 依赖路径 | 受影响组件 | 阻断前状态 | 当前 artifact | 关联 open PR | inherited gate | 证据 | 同步状态 |
|--------|------|----------|------------|------------|-----------------|--------------|----------------|------|----------|
| `[#source / PR]` | `[#target / PR / regression Issue]` | `[#source -> #target]` | `[component]` | `[ready/in_progress/needs-review/approved/未阻断]` | `[branch@sha / build digest]` | `[PR #... / 无]` | `[G-id / not-applicable]` | `[URL/path + SHA / pending]` | `pending/synced/unknown/not-applicable` |

- 影响面枚举与矩阵覆盖：`[validated / unknown；unknown 时源 PR 不得批准]`
- 未同步下游：`[无 / 列表；仅阻断对应下游，不反向阻塞已验证矩阵的源 PR]`

## 明确没做
- [不在本 Issue 范围内的事项，避免 Reviewer 当成遗漏]
- [如发现新问题，写「已开 Issue #xxx 跟进」]

## 验证方式
[Reviewer 怎么自己验一遍：命令、路径、预期输出]

## 合并新鲜度记录（Integrator 填写）
- 合并前 main SHA：`[sha]`
- PR head SHA：`[sha]`
- GitHub 合并状态：`mergeable=[MERGEABLE]`，`mergeStateStatus=[CLEAN]`
- main 前进后的补充验证：[无 / 更新分支或合并结果上的命令与结果]

## 风险与影响面
[有无破坏性变更、需不需要迁移、影响哪些调用方；没有就写「无」]
