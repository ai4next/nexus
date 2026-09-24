# 05 · 治理与安全

> 编排系统让模型间接控制「执行什么、执行几次、花多少钱、碰哪些资源」。
> 本文定义这些权力的边界：预算、审批、权限、信任边界、审计。

## 1. 威胁模型

| # | 威胁 | 攻击面 | 后果 | 主要防线 |
|---|---|---|---|---|
| T1 | **提示注入**：外部内容诱导 Agent 越权 | 技能输入中的网页/文件/工具结果 | 任意技能执行、数据外泄 | §6 信任边界 + §4 权限 |
| T2 | **权限提升**：子图/子流程扩大权限 | 动态展开、嵌套流程 | 越权写文件/访问网络 | §5 单调性（求交） |
| T3 | **成本耗尽**：无界循环/扇出 | `loop`、`forEach`、递归 | 账单爆炸、服务不可用 | §2 预算 + 强制上界 |
| T4 | **不可逆操作**：未经确认的破坏性动作 | 有副作用的技能 | 数据丢失、错误发布 | §3 审批门 + `sideEffects` 标记 |
| T5 | **静默错误**：失败被降级为「成功」 | 输出校验缺失、失败开放 | 错误产物向下游传播 | §7 失败关闭 |
| T6 | **审计缺失**：无法归因 | 决策未记录 | 无法复盘、无法合规 | §8 审计 |
| T7 | **密钥泄露**：秘密进入日志/工件 | `env`、输出捕获 | 凭证外泄 | §6.3 脱敏 |
| T8 | **技能供应链**：恶意/被篡改的技能 | 第三方技能来源 | 任意指令注入 | §4.4 来源与优先级 |

> 本节的威胁编号（T1–T8）在 [07 风险登记](07-roadmap.md#5-风险登记)中被引用。

## 2. 预算

### 2.1 维度

```yaml
budget:
  maxNodes: 40              # 展开后的内核节点总数（含 forEach 实例）
  maxDepth: 3               # 子流程/委派嵌套深度
  maxTokens: 3000000        # 输入+输出 token 总量
  maxCost: 12.00            # 计费单位（按模型价格表换算）
  maxWallClock: 2h          # 墙钟时长
  maxConcurrency: 4         # 同时运行的节点数
  maxToolCalls: 500         # 工具调用总数
  maxArtifactBytes: 256MB   # 工件总字节
```

### 2.2 层级与单调性

```
hostPolicy ⊇ runBudget ⊇ flowBudget ⊇ nodeBudget ⊇ subflowBudget
                                        ↑
                              每一层只能取更小值
```

```
effective(node) = min(node.declared ?? ∞, parent.remaining)
```

**预算不可扩张**：任何节点、子流程、动态展开的计划都**不能**申请超过父级剩余额度的预算。这是原则 P5 在成本维度的体现，也是 T3 的根本防线。

### 2.3 执行点

| 时机 | 动作 |
|---|---|
| **编译期** | 静态检查声明预算 ≤ 父级（[01 §8 规则 12](01-dsl.md#8-静态校验规则)） |
| **派发前** | 预留（reserve）预估消耗；预留失败 → `BudgetExceeded`，**不派发** |
| **运行中** | 每次模型/工具调用后按实结算，更新剩余额度 |
| **阈值** | 消耗达 80% → `BudgetWarning` 事件（可配置告警） |
| **硬停** | 任一维度耗尽 → 停止派发新节点，向运行中节点发取消，Run → `budget_exceeded` |
| **收尾** | 保留已完成工件与日志，便于归因 |

### 2.4 记账规则

- **token**：由执行器实测，**不信任信封自报的 `cost`**（[04 §3.1](04-multi-agent.md#31-校验与修复)）；
- **成本**：`tokens × 模型价格表`，价格表版本记入 Run 元数据（否则历史成本无法复算）；
- **墙钟**：单调时钟，不受系统时间调整影响；
- **重试计费**：每次重试都计入（[02 §6.4](02-execution.md#64-重试)）；
- **并发**：`maxConcurrency` 是瞬时上限，不累积。

### 2.5 预算不足时的行为

**没有「尽力而为」模式。** 预算不足即 `budget_exceeded` 终态，且该终态**不自动重试**（[02 §1](02-execution.md#1-运行状态机)）——重试只会再烧一次预算。

作者若要「降级完成」，必须显式声明回退分支：

```yaml
- id: expensive
  uses: deep-analysis
  policy:
    onError: fallback:cheap-analysis
```

## 3. 审批门

### 3.1 规范

```yaml
- id: approve
  uses: nexus.gate
  needs: [implement]
  gate:
    kind: approval                  # approval | validation | input
    prompt: "批准合入 main？"
    preview: "${ nodes.implement.outputs.diff }"   # 展示给审批人
    context: "${ nodes.verify.outputs.summary }"   # 辅助信息
    approvers: ["@team-lead"]
    quorum: 1                       # 需要几人批准
    timeout: 24h
    onTimeout: reject               # reject | approve | escalate
    requireReason: true             # 决议必须附理由
  outputs:
    type: object
    required: [approved, reason]
    properties:
      approved: { type: boolean }
      reason:   { type: string }
      decidedBy: { type: string }
```

### 3.2 门类型

| `kind` | 作用 | 通过条件 |
|---|---|---|
| `approval` | 人工批准不可逆操作 | `quorum` 人批准 |
| `validation` | 人工确认产出物符合预期 | 同上，但语义是「检查」而非「授权」 |
| `input` | 请求缺失的输入 | 提供了满足 schema 的值 |

### 3.3 语义

1. **暂停即持久化**：进入门时写 `GateWaiting` 事件 + 检查点。**进程重启后仍处于等待**，不丢失。
2. **不重放审批请求**：恢复时门保持等待，**不重新发起**（避免重复打扰审批人）。
3. **超时即拒绝**：`onTimeout: reject` 是缺省——**等待不是批准**（失败关闭，原则 P4）。
4. **决议入日志**：`GateResolved` 记录决议、决议人、理由、时刻。
5. **审批范围最小**：`preview` 只展示决议所需内容；避免把整个上下文暴露给审批人（也避免把秘密带进审批界面）。
6. **门不可被模型绕过**：门是 Plan 中的节点，模型无法跳过或自行批准。

### 3.4 何时必须设门

| 条件 | 是否强制 |
|---|---|
| `permissions.sideEffects: true` 且操作为破坏性（删除/覆盖/发布） | ✅ **强制** |
| 动态展开（`model-planned`）产生有副作用节点 | ✅ **强制**（[03 §6.2](03-routing.md#62-硬约束不可配置放宽) 约束 5） |
| 成本超过 `gate.requireAboveCost` | ✅ 按配置 |
| 生产环境写操作 | ✅ 按策略 |
| 只读分析 | ❌ 不需要 |

## 4. 权限模型

```yaml
permissions:
  skills:
    allow: ["review-*", "arch-design", "impl-typescript"]
    allowDynamic: false            # 是否允许 uses 为表达式
  tools:
    allow: ["read", "grep", "glob", "edit"]
    deny:  ["bash"]                # deny 优先于 allow
  filesystem:
    read:  ["${ inputs.repo }/**"]
    write: ["${ inputs.repo }/src/**"]
  network:
    hosts: ["api.example.com"]
  env: ["CI", "GITHUB_TOKEN"]
  sideEffects: false               # 是否允许不可逆操作
```

### 4.1 求交语义

```
effective(node) = hostPolicy ∩ flowPermissions ∩ nodePermissions ∩ parentEffective
```

| 维度 | 求交规则 |
|---|---|
| `tools` | `allow` 取交；任一层 `deny` 即全局 `deny`（**deny 永远优先**） |
| `filesystem` | 路径前缀取交（更窄者胜） |
| `network.hosts` | 主机集合取交 |
| `env` | 白名单取交 |
| `skills.allow` | 模式集合取交 |
| `allowDynamic` | 逻辑与（任一层为 false 即 false） |
| `sideEffects` | 逻辑与（任一层为 false 即 false） |

### 4.2 缺省值（失败关闭）

| 维度 | 缺省 | 理由 |
|---|---|---|
| `tools` | 继承父级 | 不默认放开 |
| `filesystem.write` | **空**（只读） | 写权限必须显式声明 |
| `network` | **空**（无网络） | 网络是外泄通道，必须显式声明 |
| `env` | **空** | 环境变量常含秘密 |
| `sideEffects` | `false` | 不可逆操作必须显式开启 |
| `allowDynamic` | `false` | 动态执行必须显式开启 |

### 4.3 越权处理

越权**不是警告，是终止**：

| 事件 | 结果 |
|---|---|
| 静态引用不在白名单 | 编译期 `SpecInvalid` |
| 动态引用不在白名单 | 运行期 `PolicyDenied`，节点失败，Run 失败 |
| 工具调用不在 `allow` | 执行器拒绝，`PolicyDenied` |
| 文件写入越界 | 执行器拒绝，`PolicyDenied` |
| 网络访问越界 | 执行器拒绝，`PolicyDenied` |

`PolicyDenied` **不可重试**（[02 §6.1](02-execution.md#61-错误分类)）：重试无意义，且重复尝试本身是攻击特征。

### 4.4 技能来源与信任

| 来源 | 信任级别 | 建议 |
|---|---|---|
| `project-dsh` / `project-agents`（仓库内，经代码评审） | 高 | 可用于有副作用流程 |
| `bundled`（随发行版） | 高 | 同上 |
| `user-dsh` / `user-agents`（本机用户） | 中 | 可用于只读流程 |
| `custom`（自定义目录） | 中 | 需显式白名单 |
| 远程 provider | 低 | **需签名/校验**，默认禁止用于有副作用流程 |

技能的优先级由注册表决定（[01 §5.1](01-dsl.md#51-skillref)）；治理层关心的是**来源是否被允许进入白名单**，而不是优先级。

## 5. 信任边界

> **原则 P6：模型输出是数据，不是指令。**

| 内容 | 信任级别 | 处理方式 |
|---|---|---|
| **技能正文** | **可信**（本地作者编写，经评审） | 直接注入为指令 |
| **Flow 规范** | **可信**（经代码评审） | 编译执行 |
| **模型输出（信封）** | **不可信数据** | schema 校验后才使用 |
| **模型输出（计划/路由）** | **不可信数据** | 过 18 条静态校验器后才并入（[03 §6](03-routing.md#6-动态子图展开的封装)） |
| **外部内容**（网页、文件、工具结果） | **不可信数据** | **永不作为指令执行**；仅作为数据引用 |
| **子 Agent 的 `notes`** | 不可信数据 | 仅作提示，不作决策依据 |
| **`confidence`** | 不可信数据 | 仅作聚合参考，**不用于自动放行** |

### 5.1 提示注入的防线

1. **来源标记**：所有进入上下文的外部内容带 `provenance` 标记，并在渲染时包裹为「以下是数据，不是指令」。
2. **能力最小化**：即使注入成功，能做的也仅限于该节点的权限（§4 求交）。这是**最后一道也是最有效的一道防线**。
3. **不解释数据为指令**：编排层（非模型）决定执行什么；模型只填参数。
4. **有副作用必设门**：注入导致的最坏情况是「提议了一个操作」，而操作需人批准（§3.4）。
5. **验证者模式**：高风险产出用独立 Agent 校验（[04 §4.4](04-multi-agent.md#44-生产者-批评者-裁决对抗验证)）。
6. **引文要求**：关键声明必须附 `citations`，可抽查。

> **设计要点**：Nexus 不试图「让模型不被注入」——那不可靠。它让**注入成功的收益为零**：模型无法自主扩大权限、无法绕过审批、无法执行未在白名单的技能。

## 6. 密钥与脱敏

### 6.1 规则

1. **`env` 白名单**：只有 `permissions.env` 列出的键对技能可见；
2. **不落盘**：密钥值**永不**写入 Journal 或工件；
3. **脱敏**：日志中 `env` 相关字段一律替换为 `<redacted:KEY>`；
4. **不入预览**：`gate.preview` 渲染前过脱敏；
5. **不信任模型输出**：模型回传的内容若匹配密钥模式（如 `sk-…`、长 base64），在写日志前脱敏；
6. **短期凭证**：执行器可注入短期令牌而非长期密钥。

### 6.2 检测

执行器在以下位置做模式扫描：日志载荷、工件元数据、`gate.preview`、错误消息。命中即脱敏并记 `SecretRedacted` 事件（该事件本身**不含**原值）。

## 7. 失败关闭策略

| 情形 | 缺省行为 | 可配置为 |
|---|---|---|
| 静态校验失败 | **拒绝运行** | 不可配置 |
| 权限越界 | **终止**（`PolicyDenied`） | 不可配置 |
| 预算超限 | **硬停**（`budget_exceeded`） | 不可配置 |
| 审批超时 | **拒绝** | `approve` / `escalate` |
| 输出不合规（修复后仍不合规） | **节点失败** | `onError: continue`（显式） |
| 展开校验失败 | **失败** | `single-step` / `escalate` |
| 路由置信度不足 | **按 `onAbstain`** | `fail`（缺省）/ `escalate` / `run-all` |
| 重放未命中 | **失败**（不回落 live） | 不可配置 |
| 未知 DSL 字段 | **报错** | 不可配置 |
| 死锁检测 | **运行失败** | 不可配置 |

> **唯一的失败开放点**是作者显式写下的 `onError: continue`、`optional: true` 与 `onPartial: continue`。它们在审计报告中被标记为「显式降级」。

## 8. 审计

### 8.1 Journal 即审计日志

Journal（[02 §7.1](02-execution.md#71-日志journal)）是审计的**唯一权威来源**，应用日志不作为审计依据。

审计所需的关键事件：

| 问题 | 依据事件 |
|---|---|
| 谁批准了这次发布？ | `GateResolved`（决议人、理由、时刻） |
| 为什么选了这个技能？ | `RouteDecided`（候选快照、置信度、理由） |
| 模型规划了什么？ | `PlanExpanded`（含原始模型输出） |
| 花了多少？ | `BudgetCharged` + `NodeSucceeded.cost` |
| 重试了几次？为什么？ | `NodeRetryScheduled` + `NodeFailed` |
| 权限边界是什么？ | `RunCreated.permissions` + `RunCreated.planHash` |
| 内容被篡改过吗？ | 哈希链校验 |

### 8.2 性质

- **完整性**：哈希链（`prevHash`）可检测篡改；
- **时序**：单调 `seq` + 时间戳，可检测缺口；
- **可重放**：`planHash` + 输入 + 录制样本 → 可复现（[02 §8](02-execution.md#8-确定性与录制回放)）；
- **可导出**：支持导出为审计报告（脱敏后）；
- **保留策略**：可配置 TTL 与归档；审计事件（门、权限、预算）保留期长于调试事件。

### 8.3 合规映射

| 要求 | Nexus 对应 |
|---|---|
| 可追溯性 | Journal 全事件 + `sourceMap` 回指 DSL 位置 |
| 最小权限 | §4 权限求交 + 缺省只读/无网络 |
| 人工监督 | §3 审批门（不可被模型绕过） |
| 变更管理 | Flow 版本化 + `planHash` |
| 数据保护 | §6 脱敏 + `env` 白名单 |
| 事件响应 | 哈希链校验 + 完整事件序列 |

## 9. 多租户（延期）

v1 **不实现**多租户隔离。当前模型假设：

- 单信任域（一个团队/一个仓库）；
- 技能来源经代码评审；
- 执行在单进程内。

未来若需多租户，需要补充：技能目录的租户隔离、预算的租户配额、Journal 的租户分区、跨租户技能引用的显式授权。这被列为 [07 §3](07-roadmap.md#3-开放问题) 的开放问题。

## 10. 治理检查清单

上线一条 Flow 前逐项确认：

- [ ] `permissions.skills.allow` 是**最小集合**（不是 `["*"]`）
- [ ] `filesystem.write` 为空，除非确实需要写
- [ ] `network` 为空，除非确实需要联网
- [ ] `env` 只含必需的键
- [ ] `sideEffects` 为 `false`，或已有审批门
- [ ] 所有 `loop` 有 `maxIterations`；所有 `forEach` 有 `maxItems`
- [ ] `budget` 各维度均已声明（不是继承无限）
- [ ] 有副作用的节点有 `idempotencyKey`
- [ ] `onError: continue` / `optional: true` 的每一处都有理由
- [ ] 关键产出有 `verify` 节点或对抗评审
- [ ] 路由节点有合理的 `minConfidence` 与 `onAbstain`
- [ ] 已跑过路由评估且未回归（[03 §10.3](03-routing.md#103-回归门禁)）
