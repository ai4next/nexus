# 04 · 多智能体协作

> 本文定义 Nexus 如何把技能派发到独立子 Agent，以及由此产生的契约、拓扑、聚合与失败模式。
> 核心问题不是「能不能多 Agent」，而是**什么时候值得多 Agent**。

## 1. 多智能体解决什么

单 Agent 顺序执行技能有三个天花板：

| 天花板 | 表现 | 多智能体的解法 |
|---|---|---|
| **上下文膨胀** | 第 10 步时历史里塞满了前 9 步的原始材料 | 子 Agent 消化原始材料，只回传结构化信封 |
| **串行延迟** | 三个独立调研必须排队 | 并行扇出 |
| **无独立验证** | 同一个 Agent 既生产又检查，错误相关性高 | 独立子 Agent 作为对抗评审者 |
| **权限一刀切** | 整个会话拥有全部工具权限 | 每个子 Agent 独立工具/文件系统权限 |

但它同时引入新的代价（§7）。**Nexus 的立场是：多智能体是手段，不是目标。**

## 2. 执行模式

| 维度 | `inline` | `subagent` | `delegate` |
|---|---|---|---|
| 执行位置 | 编排者当前步骤 | 全新子 Agent | 全新子 Agent |
| 上下文 | 共享编排者历史 | 隔离，仅声明输入 | 隔离，仅声明输入 |
| 技能正文 | 注入当前步骤 | 注入子 Agent 初始上下文 | 同左 |
| 模型 | 编排者模型 | `agent.model`（可 `inherit`） | 独立模型 |
| 工具权限 | 编排者权限 | `agent.tools` 求交 | 独立工具策略 |
| 可并行 | ❌ | ✅ | ✅ |
| 交接损失 | 无 | 有 | 有 |
| 上下文成本 | 高（污染编排者） | 低（隔离） | 低 |
| 启动成本 | 最低 | 中 | 中高 |

### 2.1 选择准则

```
该步骤是否会产生大量原始材料（读文件、抓网页、跑测试）？
  ├─ 是 → subagent（隔离消化，回传摘要）
  └─ 否 → 继续
       该步骤是否与其它步骤无数据依赖？
         ├─ 是 → subagent（并行）
         └─ 否 → 继续
              该步骤是否需要独立验证（对抗性）？
                ├─ 是 → subagent（相关性去耦）
                └─ 否 → 继续
                     该步骤是否需要不同权限或不同模型？
                       ├─ 是 → delegate
                       └─ 否 → inline
```

**缺省倾向**：`subagent`。`inline` 应用于明确短小（< 2k token 往返）、且与后续步骤强耦合的收尾步骤（如最终汇总）。

### 2.2 `inline` 的真实成本

`inline` 不是「免费」的。技能正文与中间产物**永久留在编排者历史中**，后续每一步都要为它付 token。一个 3k token 的技能正文，在 20 步流程中可能被重复计费 20 次（受前缀缓存缓解，但不为零）。

> **经验规则**：若一个技能正文 > 1.5k token 或产出 > 2k token，且后续还有 > 3 步，用 `subagent`。

## 3. 结果信封

子 Agent 的返回值不是自由文本，而是**结构化信封**。这是多智能体可靠性的基石。

```json
{
  "status": "succeeded",
  "outputs": { "verdict": "changes_requested", "findings": [ "..." ] },
  "artifacts": [
    { "name": "full-review", "uri": "artifact://sha256:ab12…", "mediaType": "text/markdown", "bytes": 18244 }
  ],
  "notes": "总体结构合理，但错误处理缺少超时分支。",
  "citations": [
    { "claim": "缺少超时", "source": "src/client.ts:88", "kind": "file" }
  ],
  "confidence": 0.78,
  "openQuestions": [ "重试上限应该是 3 还是 5？未在需求中明确" ],
  "cost": { "tokensIn": 12400, "tokensOut": 1800, "wallClockMs": 41200, "toolCalls": 9 }
}
```

| 字段 | 必填 | 作用 |
|---|---|---|
| `status` | ✅ | `succeeded` / `failed` / `partial` |
| `outputs` | ✅ | **必须匹配节点声明的 `outputs` schema** |
| `artifacts` | | 大载荷按引用回传，避免信封膨胀 |
| `notes` | | 低成本表达「发现了但没地方放」的信息 |
| `citations` | | 让声明可追溯，是对抗幻觉的主要手段 |
| `confidence` | | 供下游聚合与裁决使用（不用于自动放行） |
| `openQuestions` | | 显式暴露不确定性，触发升级或澄清 |
| `cost` | ✅ | 预算记账；由执行器填充 |

### 3.1 校验与修复

```
信封返回
   │
   ├─ 结构校验：是合法 JSON 且含必填字段？
   │     └─ 否 → 修复重试（1 次）
   │
   ├─ outputs 校验：匹配节点 outputs schema？
   │     └─ 否 → 修复重试（1 次，把校验错误回喂）
   │
   ├─ 成本校验：cost 存在且合理？
   │     └─ 否 → 由执行器实测值覆盖（不信任自报）
   │
   └─ 通过 → 写入工件 + 记录 NodeSucceeded
```

**修复重试**（schema-repair）：把「你的输出不满足以下 schema，错误是 …，请只输出修正后的 JSON」回喂一次。这是**允许的唯一一次**对同一节点的自动重试之外的额外尝试，且**计入预算**。

修复仍失败 → `OutputSchemaViolation` → 按 `policy.onError` 处理。**绝不放行不合规输出**（原则 P4）。

### 3.2 为什么信封比自由文本可靠

| 自由文本 | 结构化信封 |
|---|---|
| 下游需解析自然语言 → 脆弱 | 下游按 schema 读取 → 确定 |
| 「我完成了」无法验证 | `status` + `outputs` 可校验 |
| 无来源 → 幻觉不可查 | `citations` 可抽查 |
| 不确定性被隐藏 | `confidence`/`openQuestions` 显式 |
| 大载荷只能内联 | `artifacts` 按引用 |

> **信封纪律是多智能体与「多个 Agent 各说各话」的分界线。**

## 4. 协作拓扑模式库

以下模式均由 DSL 构造表达，无需新的运行时原语。

### 4.1 链式流水线

```yaml
- id: a
  uses: research
- id: b
  uses: draft
  needs: [a]
  with: { material: "${ nodes.a.outputs }" }
- id: c
  uses: polish
  needs: [b]
  with: { draft: "${ nodes.b.outputs }" }
```

**适用**：步骤严格依赖。**风险**：交接损失沿链累积——每步只回传声明字段，第 3 步可能已丢失第 1 步的关键细节。**缓解**：把关键原始材料作为工件沿链传递，而非只传摘要。

### 4.2 扇出-归约（Map-Reduce）

```yaml
- id: shards
  forEach:
    over: "${ nodes.plan.outputs.files }"
    as: file
    concurrency: 6
    maxItems: 64
    do:
      uses: analyze-file
      with: { path: "${ item }" }
    reduce:
      merge: array-collect
      minSuccess: 8
      with: { findings: "${ items }" }
```

**适用**：可独立分片的大工作量。**关键**：`minSuccess` 决定部分失败是否可接受。

### 4.3 主管-工人（Supervisor-Worker）

```yaml
- id: supervise
  loop:
    maxIterations: 4
    until: "${ carry.done }"
    carry: { done: false, tasks: "${ nodes.plan.outputs.tasks }" }
    do:
      uses: supervisor
      with: { tasks: "${ carry.tasks }", results: "${ carry.results }" }
      outputs:
        type: object
        required: [done, tasks]
        properties: { done: {type: boolean}, tasks: {type: array} }
```

主管动态分配任务并复核结果。**必须**有 `maxIterations`（原则 P7）。**风险**：主管成为单点，其判断错误会放大。

### 4.4 生产者-批评者-裁决（对抗验证）

```yaml
- id: produce
  uses: draft-proposal

- id: critics
  needs: [produce]
  forEach:
    over: [security, performance, maintainability]
    as: lens
    concurrency: 3
    maxItems: 4
    do:
      uses: critique
      with: { lens: "${ item }", artifact: "${ nodes.produce.outputs }" }

- id: judge
  needs: [critics]
  uses: judge-verdict
  with: { critiques: "${ nodes.critics.outputs }", artifact: "${ nodes.produce.outputs }" }
```

**适用**：高错误代价的产出（架构、安全、发布）。**收益**：错误去相关——不同视角的独立 Agent 同时犯同一错误的概率低于单个 Agent 自查。**成本**：N+2 倍。

**关键纪律**：批评者**必须**独立于生产者（不同子 Agent，不共享生产者的推理过程），否则只是自我确认。`critique` 的输入只有产出物，没有生产者的思路——这是刻意的。

### 4.5 专家路由（Router-of-Experts）

```yaml
- id: pick
  uses: nexus.select
  select: { task: "${ inputs.question }", candidates: { tags: [domain] }, strategy: embedding }
- id: answer
  uses: "${ nodes.pick.outputs.selected[0].name }"
  needs: [pick]
  with: { question: "${ inputs.question }" }
```

见 [03 技能路由](03-routing.md)。

### 4.6 反思循环（Reflexion）

```yaml
- id: refine
  loop:
    maxIterations: 3
    until: "${ carry.score >= 0.85 }"
    carry: { draft: "${ nodes.draft.outputs.text }", score: 0 }
    do:
      uses: self-critique-and-revise
      with: { draft: "${ carry.draft }" }
```

同一技能在循环中依据自评改进。**必须**有上界与收敛条件；未收敛时 `converged: false` 显式暴露。

### 4.7 人在环（Human-in-the-Loop）

```yaml
- id: approve
  uses: nexus.gate
  gate:
    kind: approval
    prompt: "批准合入 main？"
    preview: "${ nodes.implement.outputs.diff }"
    timeout: 24h
    onTimeout: reject
```

见 [05 §3](05-governance.md#3-审批门)。

### 4.8 分层委派

```yaml
- id: sub
  uses: flow://deep-research@2.0.0
  with: { question: "${ inputs.question }" }
```

一个 Flow 作为节点，其内部又可能是多 Agent 拓扑。深度受 `budget.maxDepth` 约束。**风险**：预算与延迟随深度叠加；**建议**：深度 ≤ 3。

### 4.9 拓扑选择速查

| 需求 | 拓扑 |
|---|---|
| 步骤严格依赖 | 链式 |
| 大量同质独立工作 | 扇出-归约 |
| 步骤未知、需动态分配 | 主管-工人 |
| 高错误代价 | 生产者-批评者-裁决 |
| 多领域、选一个 | 专家路由 |
| 产出可迭代改进 | 反思循环 |
| 不可逆操作 | 人在环 |
| 任务本身是一条流程 | 分层委派 |

## 5. 聚合与 join

### 5.1 join 策略

| 策略 | 语义 | 适用 | 风险 |
|---|---|---|---|
| `all` | 全部上游成功 | 缺一不可的依赖 | 一个失败即整体失败 |
| `any` | 首个成功即放行 | 冗余执行、竞速 | 其余分支的失败被忽略 |
| `quorum:n` | ≥ n 个成功 | 投票、评审 | 需 n 合理，否则形同 `all` 或 `any` |

### 5.2 合并策略

| 策略 | 语义 |
|---|---|
| `object-merge` | 对象浅合并 |
| `array-collect` | 收集为有序数组（保持输入顺序，保证确定性） |
| `last-write-wins` | 按拓扑序最后写入者胜 |
| `by-rank` | 按上游节点的显式 rank 决定 |

### 5.3 冲突处理

```yaml
# Reduce Spec（见 01 §5.3）
merge: object-merge
onConflict: error          # error | last-write-wins | by-rank
```

**缺省 `error`**：多 Agent 产出同一字段的不同值时，静默覆盖会丢失信号。冲突本身是有价值的信息（说明两个 Agent 理解不一致），应当失败或升级，而非悄悄取一个。

### 5.4 部分失败

```yaml
# Reduce Spec（见 01 §5.3）
join: "quorum:2"
minSuccess: 2
onPartial: continue        # continue | fail
```

- `items` 只含**成功实例**的输出；
- `nodes.<id>.failures` 含失败实例的错误清单；
- `onPartial: continue` 时信封 `status: partial`，下游可据此降级处理。

## 6. 隔离与最小权限

每个子 Agent 是一个**权限封闭单元**：

| 维度 | 约束 |
|---|---|
| 输入 | 仅 `with` 声明的绑定，渲染后注入 |
| 技能正文 | 仅本节点的技能正文 |
| 工具 | `agent.tools.allow ∩ Flow.permissions.tools` |
| 文件系统 | `permissions.filesystem` 的读/写路径（**默认只读**） |
| 网络 | `permissions.network.hosts`（默认无网络） |
| 环境变量 | `permissions.env` 白名单 |
| 预算 | ≤ 父级剩余 |
| 时限 | ≤ 父级剩余 |
| 嵌套深度 | ≤ 剩余 `maxDepth` |

**共享可变状态：无。** 子 Agent 之间不共享内存、不共享文件句柄、不共享会话。通信**只能**通过声明的端口（数据边）。

这条约束是可靠性的来源：没有隐式耦合，就没有「A 改了 B 依赖的东西」这类问题。

### 6.1 权限只能求交

```
effective(node) = hostPolicy ∩ flowPermissions ∩ nodePermissions ∩ parentEffective
```

**任何一层都不能扩张。** 这是原则 P5 的运行时实现，也是动态展开（[03 §6](03-routing.md#6-动态子图展开的封装)）安全的根本原因。

## 7. 失败模式

| 失败模式 | 表现 | 根因 | 缓解 |
|---|---|---|---|
| **交接损失** | 下游产出质量突然下降 | 关键信息未写入 `outputs` | 显式 schema + `notes`/`openQuestions` + 工件沿链传递 |
| **错误放大** | 早期小错沿链变成大错 | 每步都信任上一步 | 中途校验节点（`verify`）；关键处对抗评审 |
| **成本爆炸** | 扇出 × 循环 × 嵌套 = N^k | 无界组合 | 全部上界（`maxItems`/`maxIterations`/`maxDepth`/`maxNodes`） |
| **错误相关性** | 多个 Agent 同时错同一处 | 同模型、同提示、同材料 | 视角多样化（不同 lens）、独立材料、必要时不同模型 |
| **协调开销** | 多 Agent 比单 Agent 还慢 | 交接与同步成本超过并行收益 | 只在真有独立工作或需隔离时用多 Agent |
| **指标博弈** | Agent 优化表面指标 | 评审标准被当作目标 | 评审用判别式标准而非评分；保留人工抽查 |
| **上下文漂移** | 循环中越改越偏 | 每轮只看到上一轮 | `carry` 保留原始目标；每轮把原始需求重新注入 |
| **共识幻觉** | 所有批评者都同意 | 批评者看到了彼此的结论 | 批评者并行、互不可见（扇出而非链式） |

> 最后一条最隐蔽：如果批评者能看到前一个批评者的意见，它们会趋同。**对抗验证必须并行扇出**，不能串成链。

## 8. 收益/代价模型

多 Agent 的净收益近似：

```
净收益 ≈ (上下文节省 + 并行加速 + 验证增益) − (交接损失 + 协调开销 + 成本倍数)
```

| 场景 | 建议 |
|---|---|
| 单步、短、强耦合 | **`inline`** —— 多 Agent 纯亏 |
| 大量独立同质工作 | **扇出-归约** —— 并行收益显著 |
| 上下文密集的调研/分析 | **`subagent`** —— 隔离收益显著 |
| 高错误代价产出 | **对抗验证** —— 值得 N 倍成本 |
| 步骤未知 | **主管-工人 / 模型规划** —— 需严格上界 |
| 权限差异大 | **`delegate`** —— 隔离权限 |

**反面清单**：不要为了「看起来更 Agentic」而多 Agent；不要用多 Agent 掩盖单步提示词质量问题；不要在无并行收益时引入子 Agent。

## 9. 反模式

| 反模式 | 问题 | 正解 |
|---|---|---|
| 子 Agent 返回自由文本 | 下游解析脆弱，无法校验 | 强制信封 + schema |
| 把所有上游输出透传给每个子 Agent | 上下文膨胀，回到单 Agent 问题 | 只传声明输入 |
| 批评者串行且可见彼此 | 趋同，验证失效 | 并行扇出，互不可见 |
| 扇出无 `maxItems` | 成本不可控 | 必填上界 |
| 子 Agent 拥有比父级更多权限 | 越权 | 求交，只减不增 |
| 用 `confidence` 自动放行 | 模型自评不可靠 | `confidence` 只作聚合参考，放行靠 schema + gate |
| 深度嵌套（> 3 层） | 调试困难，成本叠加 | 扁平化或改用链式 |
| 全部步骤都用 `subagent` | 交接损失累积，延迟增加 | 按 §2.1 准则选择模式 |
