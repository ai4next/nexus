# 03 · 技能路由

> 路由回答一个问题：**给定任务描述，应该用哪个（些）技能？**
> 本文定义路由的时机、契约、策略谱系、动态子图展开的封装规则，以及如何评估路由质量。

## 1. 为什么路由是独立问题

技能目录一旦超过约 20 个，「把目录塞进上下文让模型自己挑」就开始失效：目录本身吃掉上下文，选择质量随候选数下降，且选择过程不可审计、不可缓存、不可回归测试。

路由把「选择」从「执行」中拆出来，获得四个好处：

| 好处 | 说明 |
|---|---|
| **可测试** | 选择是纯函数，可用标注数据集做回归 |
| **可缓存** | 同任务文本 + 同 catalog 版本 → 同决策 |
| **可审计** | 每个决策附候选快照、理由、置信度 |
| **可约束** | 白名单、权限、预算在**选择之前**过滤，而非执行时才拒绝 |

## 2. 路由的三个时机

| 时机 | 触发 | 载体 | 说明 |
|---|---|---|---|
| **静态（编译期）** | `uses: <literal>` | 无运行时开销 | 作者已知用哪个技能，直接写死。**应优先使用** |
| **运行期（单节点）** | `uses: nexus.select` | `select` 节点 | 需要按任务文本动态选一个/多个技能 |
| **运行期（子图）** | `uses: nexus.select` + `strategy: model-planned` | `select` 节点 + 展开器 | 模型规划出**多个**步骤，展开为子图 |

> **设计立场**：静态优先。路由是**成本与不确定性**的来源；能用静态引用时不应引入路由。`nexus.select` 的存在理由是「技能集合在作者写流程时不确定」（如插件生态、多租户目录）。

## 3. 两阶段契约

路由严格分为两个阶段，**选择阶段绝无副作用**：

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│ 1. select   │ →  │ 2. bind      │ →  │ 3. invoke    │
│ 纯函数       │    │ 解析 + 校验   │    │ 执行          │
├─────────────┤    ├──────────────┤    ├──────────────┤
│ 候选过滤     │    │ SkillRef 解析 │    │ Executor     │
│ 策略排序     │    │ 输入 schema   │    │ 派发          │
│ 产出候选+置信 │    │ 权限求交      │    │ 产出信封      │
│ 写入决策事件  │    │ 失败→PolicyDenied│  │              │
└─────────────┘    └──────────────┘    └──────────────┘
```

**为什么必须分离**：

1. `select` 可被缓存、被回放、被离线评估——它没有副作用；
2. `bind` 是**唯一的越权检查点**：动态 `uses` 的结果必须在此处命中白名单，否则 `PolicyDenied`；
3. 分离让「选了但没执行」（审批拒绝、预算耗尽）成为一个可表达的中间状态。

## 4. 候选过滤管线

**先过滤，再排序**——这是路由质量与成本的第一决定因素。对 200 个技能做语义排序既慢又不准；先按硬约束收窄到 10 个再排序则又快又准。

```
Catalog (全部技能)
   │
   ├─ ① 调用策略过滤      invocation.modelInvocable = true
   │                      （用户触发的路由用 userInvocable）
   │
   ├─ ② 权限白名单        permissions.skills.allow 的模式匹配
   │                      ★ 安全边界：白名单外的技能永不可被选中
   │
   ├─ ③ 来源过滤          source ∈ candidates.source（project-dsh / bundled / ...）
   │
   ├─ ④ 标签过滤          labels ⊇ candidates.tags
   │
   ├─ ⑤ 名称模式          namePattern / exclude
   │
   ├─ ⑥ 显式排除          candidates.exclude
   │
   └─ ⑦ 去重与稳定排序    按 name 字典序（确定性）
          │
          ▼
      Candidate Set  → 策略排序
```

管线性质：

- **② 是硬边界**：任何策略（含 `model-planned`）都不能越过。模型幻觉出的技能名在此被拒绝。
- **每一步都是确定的**：同一 catalog 版本 + 同一过滤条件 → 同一候选集。
- **候选集大小必须可观测**：若过滤后 > 50，路由策略应有明确的上限处理（截断需记录 `truncated: true`）。

## 5. 策略谱系

| 策略 | 输入 | 成本 | 确定性 | 适用 | 主要失败模式 |
|---|---|---|---|---|---|
| `exact` | 名字 | 0 | 完全 | 已知目标 | 无 |
| `rule` | 元数据谓词 | 0 | 完全 | 标签/来源清晰 | 规则维护成本 |
| `embedding` | 任务文本 vs 描述 | 低 | 高（可钉模型版本） | 候选多、语义相近 | 描述写得差则失效 |
| `llm-select` | 任务 + 候选清单 | 中 | 中（受模型影响） | 需理解意图 | 幻觉名字、过度自信 |
| `model-planned` | 任务 + 候选 + 工具能力 | 高 | 低 | 多步未知任务 | **越权、成本爆炸、计划不合理** |

### 5.1 `exact`

```yaml
select: { strategy: exact, name: arch-design }
```

退化为静态引用，仅保留 `nexus.select` 的审计外形。不推荐——直接用 `uses: arch-design`。

### 5.2 `rule`

```yaml
select:
  strategy: rule
  rules:
    - when: "${ contains(inputs.goal, 'diagram') }"
      pick: arch-design
    - when: "${ inputs.lang == 'zh' }"
      pick: summarize-zh
  fallback: embedding          # 规则全不匹配时
```

规则按顺序求值，首个命中者胜出。`when` 必须是纯确定表达式（[01 §4.3](01-dsl.md#43-函数白名单)）。

### 5.3 `embedding`

```yaml
select:
  strategy: embedding
  embedModel: "pinned-embed-v3"     # 必须钉版本，否则决策不可复现
  topK: 5
  minScore: 0.35
```

- 索引对象：`name + "\n" + description + "\n" + whenToUse`（三者拼接，因为 `whenToUse` 承载「何时选我」的判别信息）。
- 索引随 catalog 版本失效重建（catalog 变更 → 决策缓存失效）。
- **成本**：每次路由一次嵌入调用（任务文本），候选侧可预计算缓存。

### 5.4 `llm-select`

```yaml
select:
  strategy: llm-select
  model: inherit
  max: 1
  minConfidence: 0.6
  requireRationale: true
  candidatesInPrompt: 20        # 提示中最多列出的候选数
```

**结构化输出契约**（模型必须返回，且经运行时校验）：

```json
{
  "selected": [
    { "name": "impl-typescript", "confidence": 0.82, "rationale": "任务要求 TS 实现，该技能描述匹配" }
  ],
  "considered": ["impl-python", "impl-rust"],
  "abstain": false,
  "abstainReason": null
}
```

**校验规则（任一不满足即拒绝该输出）**：

1. `selected[*].name` 必须**精确命中候选集**（不接受幻觉名字，也不接受模糊匹配）；
2. `confidence ∈ [0,1]`；
3. `requireRationale: true` 时 `rationale` 非空；
4. `len(selected) ≤ max`；
5. 返回的技能必须 `modelInvocable`。

校验失败的处理：**一次修复重试**（把校验错误回喂），仍失败则按 `onAbstain` 处理。**绝不接受未校验的名字**。

### 5.5 `model-planned`（动态子图展开）

模型不只是选一个技能，而是规划出**一串带依赖的技能调用**。这是能力最强也最危险的路由形态，因此被单独封装（§6）。

## 6. 动态子图展开的封装

> **原则 P6：模型输出是数据，不是指令。**

模型产出的「计划」是一份**待校验的 Flow 片段**，它必须通过与人工编写流程**完全相同**的静态校验器，才能并入执行图。

### 6.1 展开流程

```
任务描述
   │
   ├─ ① 生成        模型产出 Flow 片段（JSON）
   │                 schema 校验：结构合法？
   │
   ├─ ② 静态校验    复用 01 §8 全部 18 条规则
   │                 ★ 与人工流程同一套校验器，无特权路径
   │
   ├─ ③ 权限求交    node.permissions ← parent.permissions ∩ plan.permissions
   │                 ★ 单调不扩张（P5）
   │
   ├─ ④ 预算收敛    node.budget ← min(plan.budget, parent.remaining)
   │                 ★ 不得超支
   │
   ├─ ⑤ 白名单核验  每个 uses ∈ permissions.skills.allow
   │
   ├─ ⑥ 审批（条件）  有副作用的节点 → 强制 gate
   │
   └─ ⑦ 并入 Plan   展开为子图，记录 RouteDecided + PlanExpanded 事件
```

### 6.2 硬约束（不可配置放宽）

| # | 约束 | 值来源 |
|---|---|---|
| 1 | 展开深度 ≤ `budget.maxDepth` | 父级预算 |
| 2 | 展开新增节点数 ≤ `budget.maxNodes − 已用` | 父级预算 |
| 3 | 每个 `uses` 必须命中白名单 | `permissions.skills.allow` |
| 4 | 权限只能求交，不能新增 | 原则 P5 |
| 5 | 无副作用节点才可自动执行；有副作用 → 强制审批门 | `permissions.sideEffects` |
| 6 | 不得引入新的 `inputs` 之外的输入源 | 上下文契约 |
| 7 | 不得声明 `env` 新键 | `permissions.env` |
| 8 | 展开图必须无环、无未绑定必填输入 | 静态校验 |
| 9 | 展开的节点不得再触发 `model-planned`（防递归爆炸） | 硬规则 |
| 10 | 展开产物全部记入 Journal（含原始模型输出） | 审计 |

### 6.3 展开的降级

展开失败（校验不通过、预算不足、模型输出不可解析）时的策略：

```yaml
select:
  strategy: model-planned
  onExpandFailure: fail        # fail | single-step | escalate
```

- `fail`：失败关闭（缺省）；
- `single-step`：退化为 `llm-select` 选一个技能，单步执行；
- `escalate`：转人工（`nexus.gate`）。

### 6.4 为什么值得做，以及为什么必须封装

**值得做**：真实任务常常无法在编写流程时穷举步骤（「修好这个 bug」的步骤取决于 bug 是什么）。模型规划能覆盖长尾。

**必须封装**：模型规划 = 让模型间接控制执行图 = 让模型间接控制系统行为。若不封装，提示注入即可导致任意技能执行、越权文件写入、预算耗尽。§6.2 的 10 条约束是**安全边界**，不是建议。

## 7. 歧义与弃权

路由应当能够说「我不确定」，而不是硬猜。**弃权（abstain）是特性，不是失败。**

```yaml
select:
  minConfidence: 0.6
  ambiguityMargin: 0.1          # top1 与 top2 的置信差小于此值 → 视为歧义
  onAbstain: escalate           # fail | escalate | run-all | fallback:<nodeId>
```

| 情形 | 判定 | 处理 |
|---|---|---|
| 最高分 < `minConfidence` | 低置信 | 按 `onAbstain` |
| `top1 − top2 < ambiguityMargin` | 歧义 | 按 `onAbstain` |
| 候选集为空 | 无候选 | 按 `onAbstain`（通常 `fail`） |
| 模型显式 `abstain: true` | 主动弃权 | 按 `onAbstain` |

`onAbstain` 取值：

- `fail`：失败关闭；
- `escalate`：转人工（生成 `nexus.gate` 请求，附候选与分数）；
- `run-all`：并行执行 top-N 候选，由 `nexus.reduce` 或评审节点裁决（成本 N 倍，但质量高）；
- `fallback:<nodeId>`：走预置回退节点。

> **反模式**：把 `minConfidence` 设为 0 以求「总能选出来」。这会把路由错误变成静默的下游失败，且失去弃权信号——而弃权率是路由质量最重要的观测指标之一。

## 8. 决策缓存与可复现

```
decisionKey = hash(
  taskText,
  strategy,
  strategyParams,
  catalogRevision,      # 注册表的 revision / 候选快照 hash
  modelId + modelVersion,
  seed
)
```

- 命中则复用决策，但**仍记 `RouteDecided` 事件**（标注 `cached: true`）——审计视图不应因缓存而缺事件；
- catalog 变更 → `catalogRevision` 变 → 缓存自然失效；
- 重放模式下优先使用日志中的决策，而非重新路由（[02 §8.3](02-execution.md#83-路由决策的可复现)）；
- **`embedModel` 与路由模型必须钉版本**，否则「同一流程两次运行选出不同技能」将无法归因。

## 9. 描述即接口

路由质量的上限由**技能元数据质量**决定。这不是文档问题，是接口设计问题。

| 字段 | 职责 | 写法 | 反例 |
|---|---|---|---|
| `name` | 稳定标识 | kebab-case、领域前缀（`review-security`） | `helper2` |
| `description` | **做什么** | 具体产出物 + 技术栈 + 边界 | 「帮助处理代码」 |
| `whenToUse` | **何时选我**（判别性） | 触发条件、与近邻技能的区分 | 与 `description` 重复 |
| `labels` | 机器过滤 | 领域/阶段/产出类型 | 自由文本 |
| 负例 | 排除场景 | 「不要用于 X，用 Y」 | 省略 |

**撰写规范（供技能作者）**：

1. **判别优先于完整**：写清「我和最像的那个技能差在哪」，比写全功能更有用；
2. **用任务语言，不用实现语言**：用户说「画个架构图」，不是「调用 SVG 渲染器」；
3. **显式负例**：`whenToUse` 中写「不适用于运行时性能分析（用 profile-*）」；
4. **可被检索**：包含同义表述（中英文、常见别称）；
5. **长度**：`description` ≤ 2 句，`whenToUse` ≤ 3 条要点。路由提示的候选数是有限的。

## 10. 评估

路由必须像代码一样被测试。

### 10.1 数据集

标注集：`(taskText, expectedSkill | abstain, context)`。

- **规模**：每个技能 ≥ 5 条正例；每对易混技能 ≥ 3 条判别用例；
- **来源**：真实会话日志（脱敏）+ 人工构造的边界用例；
- **分层**：按领域、按难度、按是否歧义分层报告，不只看总体准确率。

### 10.2 指标

| 指标 | 定义 | 目标 |
|---|---|---|
| **Top-1 准确率** | 首选命中期望技能 | 主指标 |
| **MRR** | 期望技能的排名倒数均值 | 衡量排序质量 |
| **误路由率** | 选了一个**不该用**的技能（比弃权更糟） | 应尽可能低 |
| **弃权率** | 主动弃权占比 | 与准确率权衡；异常升高说明描述退化 |
| **展开成功率** | `model-planned` 通过校验的比例 | 衡量封装规则的松紧 |
| **越权拦截数** | 被白名单拒绝的动态引用次数 | 应 > 0 说明边界在工作；突增说明注入尝试 |

### 10.3 回归门禁

- catalog 变更（新增/修改技能）→ 自动跑路由评估；
- Top-1 下降 > 3% 或误路由率上升 → **阻止合并**；
- 每次评估结果存档，与 `catalogRevision` 绑定，可追溯「哪次技能改动破坏了路由」。

## 11. 反模式

| 反模式 | 问题 | 正解 |
|---|---|---|
| 用路由替代静态引用 | 引入不必要的不确定性与成本 | 已知目标就写 `uses: <name>` |
| `minConfidence: 0` | 掩盖歧义，失去弃权信号 | 设合理阈值 + `onAbstain: escalate` |
| 候选不做过滤直接语义排序 | 慢且不准；越权候选进入排序 | 先跑 §4 过滤管线 |
| 路由模型/嵌入模型不钉版本 | 决策不可复现，无法归因 | 钉版本 + 决策缓存键含版本 |
| 让模型自由输出技能名 | 幻觉名字 → 执行不存在的能力 | 精确匹配候选集，拒绝幻觉 |
| 展开的模型计划跳过校验 | 提示注入 = 任意执行 | 复用同一套 18 条校验器（P6） |
| 只看总体准确率 | 掩盖某领域系统性失败 | 分层报告 + 误路由率 |
| `description` 写成营销文案 | 语义检索失效 | 按 §9 撰写规范，写判别信息 |
