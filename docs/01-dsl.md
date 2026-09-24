# 01 · 编排 DSL 规范

> 规范版本：`nexus/v1alpha1` · 状态：草案
>
> 本文定义 Nexus 编排语言（NOL, Nexus Orchestration Language）的语法与语义。DSL 是**语言无关**的：以 YAML 为规范序列化格式，JSON 为等价形式，两者可无损互转。

## 1. 设计目标

| 目标 | 手段 |
|---|---|
| 可读可评审 | 声明式 YAML，结构扁平，不引入图灵完备的语法 |
| 可静态校验 | 类型化端口 + 表达式静态检查 + 引用可解析性 |
| 可版本化 | `metadata.version` + 语义化版本 + 技能引用可钉版本 |
| 可降级为 IR | 控制流糖语法统一降级到 6 种内核节点 |
| 不可越权 | 权限/预算在语法层就是可静态比较的声明 |
| 无隐藏控制流 | 禁止动态 `uses`（除非显式开启且受白名单约束） |

**明确排除**：条件表达式不图灵完备；不允许在 DSL 内定义函数、导入模块、执行任意代码。需要任意逻辑时使用脚本节点（§9.3），并被显式标记为不可静态分析。

## 2. 文档结构

```yaml
apiVersion: nexus/v1alpha1     # 必填，规范版本
kind: Flow                      # 必填，v1 仅 Flow
metadata:                       # 必填
  name: repo-refactor           # 必填，kebab-case，全局唯一标识
  version: 1.0.0                # 必填，semver
  description: "..."            # 可选，路由与目录展示用
  labels: { domain: refactor }  # 可选，路由过滤用
  annotations: { owner: "@team" } # 可选，不参与语义
spec:                           # 必填
  inputs:   <Schema>            # 可选，缺省为空对象
  outputs:  <Schema>            # 可选，与 result 二选一
  result:   <Binding>           # 可选，结果的便捷映射形式
  permissions: <Permissions>    # 可选，缺省为「无额外权限」
  budget:   <Budget>            # 可选，缺省继承父级
  defaults: <Defaults>          # 可选，节点级默认值
  nodes:    [<Node>...]         # 必填，非空
```

`outputs` 与 `result` 同时出现时，`result` 的值必须通过 `outputs` 的 schema 校验。

### 2.1 Defaults

节点级默认值。节点上显式声明的同名字段覆盖此处。

```yaml
defaults:
  mode: subagent              # 缺省执行模式
  concurrency: 4              # forEach / parallel 的缺省并发
  maxItems: 32                # forEach 的缺省硬上界
  inlineThreshold: 32KiB      # 输入/输出内联阈值，超出则溢出为工件引用（02 §5.3）
  agent: { model: inherit }   # 缺省 agent 规格
  policy:                     # 缺省策略；与节点 policy 做字段级合并
    timeout: 15m
    retry: { max: 2, backoff: exponential }
```

**合并语义**：`defaults.policy` 与节点 `policy` 做**字段级浅合并**（节点优先）；`retry` 等嵌套对象整体替换，不做深合并——避免出现「作者只改了 `max`，却继承了意料之外的 `retryOn`」。

## 3. 类型系统

端口类型使用 **JSON Schema 子集**。允许的关键字：

```
type          object | array | string | number | integer | boolean | null
properties    仅 type=object
required      仅 type=object
items         仅 type=array
enum          string/number 枚举
const         常量
additionalProperties  true | false（缺省 false，即封闭对象）
oneOf         仅用于判别联合（顶层）
```

**禁止**：`$ref`（无外部引用）、`patternProperties`、`if/then/else`、`format`（作为校验依据）。理由：保持类型可比对，使静态兼容性检查可判定。

### 3.1 类型兼容性

静态检查在两个方向使用兼容性判定：

- **生产者 → 消费者**：节点 A 的输出 schema 必须可赋给节点 B 的输入 schema（`A ⊆ B`）。
- **封闭性**：`additionalProperties: false` 是缺省，因此未声明的字段在静态期即报错，而不是运行期丢失。

### 3.2 工件类型

大载荷不内联，使用工件引用：

```yaml
type: object
properties:
  report: { type: string, x-nexus-artifact: true, x-nexus-mediaType: text/markdown }
```

`x-nexus-*` 为保留扩展关键字，语义见 [02 执行语义 §5](02-execution.md#5-数据传递与工件)。

## 4. 表达式语言（NEL）

### 4.1 语法

表达式包裹在 `${ }` 中，出现在任何 `<Binding>` 位置。NEL 是 **CEL 的子集**，额外限制如下：

| 能力 | 是否允许 | 说明 |
|---|---|---|
| 算术 / 比较 / 逻辑 / 三元 | ✅ | `+ - * / % == != < <= > >= && \|\| ! ?:` |
| 成员访问 / 下标 | ✅ | `nodes.recon.outputs.findings.items[0]` |
| 字符串插值 | ✅ | `"hello ${ inputs.name }"` |
| 列表/对象字面量 | ✅ | `[1,2,3]`, `{a: 1}` |
| 函数调用 | ⚠️ 仅白名单 | 见 §4.3 |
| 宏 / 推导式 | ❌ | `map()`, `filter()` 不可用；用 `forEach` 节点替代 |
| 赋值 / 循环 / 语句 | ❌ | 表达式即值，无副作用 |
| 任意函数 / 反射 | ❌ | 无宿主语言逃逸 |

### 4.2 命名空间（根变量）

| 根 | 类型 | 可用范围 | 说明 |
|---|---|---|---|
| `inputs` | Flow 输入 | 全局 | Flow 的入参 |
| `nodes.<id>.outputs` | 节点输出 | **仅上游**（支配关系） | 见 §8 规则 6 |
| `nodes.<id>.status` | `string` | 仅上游 | `succeeded`/`failed`/`skipped`/`partial` |
| `nodes.<id>.error` | `object?` | 仅上游 | 失败时的错误对象 |
| `nodes.<id>.cost` | `object` | 仅上游 | `{tokensIn, tokensOut, wallClockMs}` |
| `run.id` / `run.startedAt` | `string` | 全局 | Run 元数据（**确定性上下文**） |
| `env.<KEY>` | `string?` | 全局 | 仅 `permissions.env` 白名单内的键 |
| `item` / `index` / `items` | 任意 | 仅 `forEach` 内 | 当前元素 / 序号 / 全部结果 |
| `carry` | 任意 | 仅 `loop` 内 | 循环携带状态 |
| `attempt` | `integer` | 仅 `retry` 内 | 第几次尝试（从 1 起） |

### 4.3 函数白名单

```
纯函数（可用于任何位置，含 when/until）：
  len, size, default, coalesce, concat, join, split, contains, matches,
  startsWith, endsWith, lower, upper, trim, keys, values, merge, pick, omit,
  flatten, unique, sort, sum, min, max, abs, floor, ceil, round, toJson, fromJson

确定性受限（仅 run 作用域，禁止出现在 when/until/join）：
  now(), uuid(), random()

类型断言：
  isString, isNumber, isObject, isArray, isNull
```

**确定性规则**：`when`、`until`、`join` 的判定必须**纯且确定**。编译期拒绝在这些位置出现 `now/uuid/random`。理由：否则同一 Run 重放会产生不同的图结构，破坏可重放性（[02 §8](02-execution.md#8-确定性与录制回放)）。

### 4.4 绑定（Binding）

一个绑定有三种形态：

```yaml
with:
  # 形态 1：整体表达式 —— 保留原始类型（对象就是对象，不是字符串）
  findings: "${ nodes.recon.outputs.findings }"

  # 形态 2：字符串插值 —— 结果必为 string
  title: "Report for ${ inputs.repo }"

  # 形态 3：结构化字面量 —— 表达式可出现在任意叶子
  options:
    lens: [security, performance]
    depth: "${ inputs.depth }"
    label: "run-${ run.id }"
```

> **整体表达式规则（关键）**：当且仅当字符串**完全等于**一个 `${...}` 且无前后缀时，绑定取表达式的**原始类型值**；否则按字符串插值处理（非字符串值序列化为 JSON 文本）。
>
> 这条规则消除了「对象被字符串化」这一最常见的编排 bug，且规则本身可静态判定。

## 5. 节点（Node）

```yaml
- id: design                      # 必填，kebab-case，Flow 内唯一
  uses: arch-design                   # 必填，SkillRef（见 §5.1）
  mode: subagent                  # 可选，inline | subagent | delegate（缺省取 defaults）
  needs: [recon]                  # 可选，依赖的上游节点 id
  join: all                       # 可选，all | any | quorum:<n>（缺省 all）
  when: "${ inputs.depth > 0 }"   # 可选，bool 表达式；false 则节点跳过
  with: { ... }                   # 可选，输入绑定
  outputs: <Schema>               # 可选，但强烈建议；缺省为任意对象
  agent:                          # 可选，仅 subagent/delegate
    model: inherit                # inherit | <model-id>
    tools: { allow: [...], deny: [...] }
    instructions: "..."           # 追加到子 Agent 的系统提示
    maxTurns: 12
  policy:
    timeout: 20m                  # 可选，时长字面量
    retry:                        # 可选
      max: 2
      backoff: exponential        # fixed | exponential
      base: 2s
      max: 60s
      jitter: true
      retryOn: [Timeout, ExecutorError, InputValidationFailed]
    onError: fail                 # fail | continue | fallback:<nodeId>
    optional: false               # true 时，本节点失败不阻塞下游（下游需处理缺失）
    idempotencyKey: "${ run.id }:design"   # 有副作用的节点必填
    replay: if-missing            # if-missing | always（恢复时的行为）
```

### 5.1 SkillRef

| 形式 | 示例 | 解析时机 | 说明 |
|---|---|---|---|
| 名字 | `arch-design` | 静态 | 由注册表解析为当前作用域下的胜出技能 |
| 钉版本 | `arch-design@2.1.0` | 静态 | 技能元数据含版本时必须精确匹配 |
| 限定来源 | `skill://bundled/arch-design` | 静态 | 限定 provider，避免歧义 |
| 内联 | `inline:` + `content` | 静态 | 内联指令正文，不经过注册表；审计标记为 `inline` |
| 子流程 | `flow://review-cycle@1.2.0` | 静态 | 嵌套编排（[§9](01-dsl.md)） |
| 内建 | `nexus.gate` / `nexus.select` / `nexus.reduce` / `nexus.noop` | 静态 | 保留前缀 `nexus.` |
| 动态 | `"${ nodes.pick.outputs.selected[0].name }"` | **运行期** | 需 `permissions.skills.allowDynamic: true`，且结果必须命中白名单 |

内联形式：

```yaml
- id: normalize
  uses:
    inline:
      description: "把用户输入规范化为小写 kebab-case"
      content: |
        将给定字符串转换为小写 kebab-case，仅输出转换结果。
```

### 5.2 执行模式（mode）

| 模式 | 上下文 | 成本 | 适用 |
|---|---|---|---|
| `inline` | 注入编排者当前上下文 | 最低（无交接损失） | 短、紧耦合、低 token 的步骤 |
| `subagent` | 全新子 Agent，仅以声明输入为种子 | 中（有交接损失） | 上下文密集、可并行、需隔离的步骤 |
| `delegate` | 子 Agent + 独立模型/工具策略 | 高 | 需专门模型或受限权限的步骤 |

详见 [04 多智能体 §2](04-multi-agent.md#2-执行模式)。

### 5.3 内建节点

**`nexus.select`** —— 路由选择，**纯函数、无副作用**：

```yaml
- id: pick
  uses: nexus.select
  select:
    task: "implement: ${ nodes.design.outputs.summary }"   # 路由依据文本
    candidates:                       # 候选过滤（先过滤后排序）
      tags: [implementation]
      source: [project-dsh, bundled]
      namePattern: "impl-*"
      exclude: [impl-legacy]
    strategy: llm-select              # rule | embedding | llm-select
    max: 1                            # 返回候选数
    minConfidence: 0.6                # 低于此值弃权
    onAbstain: fail                   # fail | escalate | fallback:<nodeId>
    allowDynamic: true                # 是否允许结果被用作 uses
  outputs:
    type: object
    required: [selected]
    properties:
      selected:
        type: array
        items:
          type: object
          required: [name, confidence]
          properties:
            name: { type: string }
            confidence: { type: number }
            rationale: { type: string }
```

**`nexus.reduce`** —— 聚合。其参数构成 **Reduce Spec**，用于两个位置：`forEach.reduce`（糖语法，隐式降级为一个 `nexus.reduce` 节点）与独立的 `uses: nexus.reduce` 节点（spec 置于节点的 `reduce` 字段）。

```yaml
# Reduce Spec 的全部字段
join: "quorum:2"          # all | any | quorum:<n>；缺省 all
merge: object-merge       # object-merge | array-collect | last-write-wins | by-rank；缺省 array-collect
onConflict: error         # error | last-write-wins | by-rank；缺省 error
onPartial: fail           # continue | fail；缺省 fail
minSuccess: 1             # 成功实例数下限；缺省取 join 的要求
```

独立节点形态：

```yaml
- id: merge
  uses: nexus.reduce
  reduce:
    join: "quorum:2"
    merge: object-merge
    onConflict: error
    minSuccess: 1
  with: { verdicts: "${ items }" }
```

> 需要**自定义归约技能**（而非内建合并）时，不要用 `nexus.reduce`：直接写一个普通节点 `uses: <skill>`，把各上游输出用 `with` 绑定进去即可。

**`nexus.gate`** —— 检查点/审批，见 [05 §3](05-governance.md#3-审批门)。

**`nexus.noop`** —— 汇合点/占位，用于把多条分支收敛为一个依赖。

## 6. 控制流糖语法

控制流以**语法糖**形式提供，编译期统一降级为内核 IR（§10）。糖语法不引入新的运行时语义，因此 Scheduler 只需理解 IR。

### 6.1 `parallel` —— 显式并行

```yaml
- id: fanout
  parallel:
    concurrency: 4
    join: all
    branches:
      - id: a
        uses: skill-a
        with: { x: "${ inputs.x }" }
      - id: b
        uses: skill-b
        with: { x: "${ inputs.x }" }
```

降级：`branches` 提升为普通节点，自动 `needs: [<本节点上游>]`；产生一个隐式 `nexus.noop` 汇合节点作为 `id`。

### 6.2 `forEach` —— 扇出/映射

```yaml
- id: critique
  forEach:
    over: "${ nodes.design.outputs.lenses }"   # 必须是 array
    as: lens                                    # 元素名，缺省 item
    concurrency: 4                              # 缺省取 defaults.concurrency
    maxItems: 16                                # 硬上界，必填或继承 defaults.maxItems（缺省 32）
    do:
      uses: review-adversarial
      with: { lens: "${ item }", proposal: "${ nodes.design.outputs.proposal }" }
    reduce:                                     # 可选，Reduce Spec（隐式绑定 items）
      join: "quorum:2"
      merge: array-collect
      minSuccess: 2
```

语义要点：

- 每个元素一个独立节点实例，实例 id 为 `<id>#<index>`；
- `item`/`index` 仅在 `do` 内可见；`items` 仅在 `reduce` 内可见，是**成功实例输出的有序数组**；
- `maxItems` 是硬上界：`over` 的实际长度超过 `maxItems` 时**编译期无法判定则运行期失败**（不静默截断）；
- 子实例失败的处理由 `reduce.join` 决定（`all` 则任一失败即整体失败）。

### 6.3 `loop` —— 有界迭代

```yaml
- id: refine
  loop:
    maxIterations: 3                    # 必填，硬上界
    until: "${ carry.score >= 0.9 }"    # 可选，收敛条件
    carry:                              # 可选，迭代间传递的状态
      draft: "${ nodes.design.outputs.proposal }"
      score: 0
    do:
      uses: refine-draft
      with: { draft: "${ carry.draft }" }
      outputs:
        type: object
        required: [draft, score]
        properties: { draft: {type: object}, score: {type: number} }
```

语义要点：

- **`maxIterations` 必填**，违反即为编译错误（原则 P7）；
- 每次迭代结束后，`carry` 按声明重新绑定（`carry.*` 的表达式在迭代末尾求值）；
- `until` 为真则提前结束；达到 `maxIterations` 仍未满足 → 节点状态 `succeeded` 但 `outputs.converged: false`，并发出 `LoopNotConverged` 警告事件；
- 迭代是**串行**的（携带状态），并发请用 `forEach`。

### 6.4 `switch` —— 条件分支

```yaml
- id: route
  switch:
    on: "${ nodes.recon.outputs.findings.kind }"
    cases:
      - when: "'legacy'"
        do: { uses: migrate-legacy }
      - when: "'greenfield'"
        do: { uses: scaffold-new }
    default:
      do: { uses: nexus.noop }
```

降级：每个 case 成为一个带 `when` 的节点；分支互斥由编译期注入的守卫保证（后续 case 的 `when` 自动与前面所有 case 的否定合取）。

### 6.5 `try` —— 局部容错

```yaml
- id: guarded
  try:
    do:
      uses: risky-skill
      with: { input: "${ inputs.x }" }
    catch:
      - on: [Timeout, ExecutorError]
        do: { uses: fallback-skill, with: { input: "${ inputs.x }" } }
    finally:
      do: { uses: cleanup-skill }
```

降级为 `policy.onError: fallback:<catchNode>` + 一条 `finally` 的**保证执行边**（无论成功失败均执行；`finally` 节点失败不覆盖主结果，仅记录事件）。

## 7. 子流程与递归组合

```yaml
- id: review
  uses: flow://review-cycle@1.2.0
  with: { proposal: "${ nodes.design.outputs.proposal }" }
  outputs:
    type: object
    required: [verdict]
    properties: { verdict: { type: string } }
```

约束：

1. **深度**：嵌套深度受 `budget.maxDepth` 约束，超限即 `BudgetExceeded`；
2. **权限单调**：子流程的 `permissions` 必须 ⊆ 调用方 `permissions`（[§8 规则 8](01-dsl.md)）；
3. **预算单调**：子流程预算 ≤ 调用方剩余预算；
4. **循环引用**：编译期检测流程引用环并拒绝（`flow://` 图必须无环）；
5. **版本钉定**：`flow://` 必须钉精确版本，禁止浮动（保证可重放）。

### 7.1 Flow 作为 Skill

一个 Flow 可同时以技能形式发布（`kind: Flow` + 技能元数据），此时它的 `description`/`whenToUse` 参与路由。这使编排可被当作原子技能复用，且**无需改动技能注册表**。

## 8. 静态校验规则

编译期（`parse → validate → lower → plan`）必须全部通过，否则拒绝运行（失败关闭，原则 P4）。

| # | 规则 | 违反时 |
|---|---|---|
| 1 | `apiVersion`/`kind` 受支持 | 编译错误 |
| 2 | `metadata.name` kebab-case、`version` 为 semver | 编译错误 |
| 3 | 节点 `id` 唯一、kebab-case、不以 `nexus.` 开头 | 编译错误 |
| 4 | 所有 `needs` 指向存在的 id | 编译错误 |
| 5 | 依赖图无环（`loop` 以显式上界降级，不算环） | 编译错误 |
| 6 | `when`/`until`/`with` 中的 `nodes.*` 引用必须**被上游支配**（dominance） | 编译错误 |
| 7 | `with` 的键 ⊆ 目标声明的输入属性（封闭对象） | 编译错误 |
| 8 | 目标的必填输入被绑定，除非目标 `optional` 或节点 `optional` | 编译错误 |
| 9 | `when`/`until` 静态类型为 `boolean` | 编译错误 |
| 10 | 生产者输出 schema 与消费者输入 schema 兼容（§3.1） | 编译错误 |
| 11 | 权限单调：节点/子流程权限 ⊆ Flow 权限 | 编译错误 |
| 12 | 预算单调：子预算 ≤ 剩余父预算 | 编译错误 |
| 13 | 静态 `uses` 可解析（`onMissing` 未开启时） | 编译错误 |
| 14 | 有副作用的节点（`permissions.sideEffects: true`）必须提供 `idempotencyKey` | 编译错误 |
| 15 | `loop` 必有 `maxIterations`；`forEach` 必有可判定的 `maxItems` | 编译错误 |
| 16 | 动态 `uses` 仅在 `permissions.skills.allowDynamic: true` 时允许 | 编译错误 |
| 17 | `env.*` 引用必须在 `permissions.env` 白名单内 | 编译错误 |
| 18 | `when`/`until`/`join` 不含 `now/uuid/random` | 编译错误 |

### 8.1 静态引用缺失的降级策略

```yaml
policy:
  onMissing: fail        # fail（缺省）| skip | route
```

- `fail`：编译错误（推荐，显式依赖应当显式失败）；
- `skip`：节点标记 `skipped`，下游若强依赖则失败；
- `route`：退化为一次 `nexus.select`（需要白名单），用于「技能可能被重命名」的场景。

## 9. 逃生舱

### 9.1 内联指令节点

`uses.inline` 允许直接写指令正文，跳过注册表。用于一次性、流程私有的步骤。审计标记来源为 `inline`，且**不参与路由**（无 description）。

### 9.2 动态引用

`uses` 为表达式时（§5.1 动态行）必须同时满足：`allowDynamic: true`、结果命中 `permissions.skills.allow`、结果为合法技能名。任一不满足 → `PolicyDenied`，失败关闭。

### 9.3 脚本节点（v1 保留，未启用）

规范为任意代码预留 `uses: nexus.script`，但 **v1alpha1 不实现**。原因：脚本破坏可静态校验与可恢复性，需要独立的沙箱与权限模型。若未来启用，必须：显式标注 `analyzer: opaque`、禁止继承 Flow 的写权限、强制 `idempotencyKey`、并在审计中标红。

## 10. 内核 IR（Plan）

Scheduler 只消费 Plan。糖语法在此前已完全消失。

```
Plan {
  flow:       { name, version }
  nodes:      [KernelNode]
  edges:      [Edge]
  inputs:     Schema
  outputs:    Schema
  result:     [Binding]
  permissions: Permissions        # 已与父级求交
  budget:     Budget              # 已按剩余额度收敛
  sourceMap:  { nodeId -> 原始 DSL 位置 }   # 报错与审计可回溯
}

KernelNode {
  id:        string               # forEach 实例为 "<id>#<index>"
  kind:      skill | flow | gate | select | reduce | noop
  ref:       SkillRef | FlowRef | null
  mode:      inline | subagent | delegate
  inputs:    [Binding]
  outputs:   Schema
  policy:    Policy               # 已合并 defaults
  agent:     AgentSpec | null
  origin:    DSLPath              # 糖语法来源，如 "nodes[2].forEach.do"
}

Edge {
  from:      nodeId
  to:        nodeId
  kind:      data | control | finally
  condition: NEL | null           # 仅 control 边
}
```

### 10.1 降级规则汇总

| 糖语法 | 降级结果 |
|---|---|
| `parallel` | N 个 `skill` 节点 + 1 个 `noop` 汇合节点；边为 `control` |
| `forEach` | N 个实例化 `skill` 节点（`<id>#<i>`）+ 可选 `reduce` 节点；`over` 长度运行期确定时按实际长度实例化 |
| `loop` | 一个 `skill` 节点 + 一个受 `maxIterations` 约束的**迭代控制器**；`carry` 成为控制器状态 |
| `switch` | 每个 case 一个节点，`condition` 为守卫表达式（含前置 case 的否定） |
| `try/catch` | `policy.onError: fallback:<id>` + `control` 边 |
| `try/finally` | `finally` 类型的边（保证执行） |
| `nexus.select` | `select` 节点（纯函数） |
| `flow://` | `flow` 节点（嵌套 Run） |

> **`forEach` 与动态长度**：当 `over` 在编译期不可静态求值（引用上游输出），实例在**运行期**按实际长度展开。展开前必须校验 `len(over) <= maxItems`，否则 `BudgetExceeded`。

## 11. 完整示例

见 [00 总览 §5](00-overview.md#5-端到端走读) 的 `repo-refactor` 流程，它覆盖了本规范的全部主要构造。

补充一个更小的、聚焦路由与多智能体的例子：

```yaml
apiVersion: nexus/v1alpha1
kind: Flow
metadata:
  name: research-and-verify
  version: 1.0.0
  description: "多源调研 + 对抗验证 + 汇总"
  labels: { domain: research }

spec:
  inputs:
    type: object
    required: [question]
    properties:
      question: { type: string }
      depth: { type: integer, enum: [1, 2, 3], default: 2 }

  permissions:
    skills: { allow: ["research-*", "verify-*", "summarize"] }
    tools: { allow: ["read", "grep", "glob", "web_search", "web_fetch"] }
    network: { hosts: ["*"] }
    sideEffects: false

  budget: { maxNodes: 24, maxDepth: 2, maxTokens: 1500000, maxWallClock: 45m, maxConcurrency: 3 }

  defaults:
    mode: subagent
    policy: { timeout: 10m, retry: { max: 1, backoff: fixed, base: 3s } }

  nodes:
    # 1) 由 Router 选出调研技能
    - id: pick
      uses: nexus.select
      select:
        task: "research: ${ inputs.question }"
        candidates: { namePattern: "research-*" }
        strategy: embedding
        max: 1
        minConfidence: 0.55
        onAbstain: escalate
        allowDynamic: true
      outputs:
        type: object
        required: [selected]
        properties: { selected: { type: array } }

    # 2) 并行调研 3 个角度
    - id: gather
      needs: [pick]
      forEach:
        over: [facts, counterpoints, prior-art]
        as: angle
        concurrency: 3
        maxItems: 8
        do:
          uses: "${ nodes.pick.outputs.selected[0].name }"
          with: { question: "${ inputs.question }", angle: "${ item }" }
          outputs:
            type: object
            required: [claims]
            properties:
              claims: { type: array }
        reduce:
          join: all
          merge: array-collect
          minSuccess: 2
          with: { bundles: "${ items }" }

    # 3) 对抗验证：独立 Agent 尝试证伪
    - id: verify
      needs: [gather]
      uses: verify-claims
      with: { claims: "${ nodes.gather.outputs.merged }" }
      outputs:
        type: object
        required: [survived, refuted]
        properties: { survived: {type: array}, refuted: {type: array} }

    # 4) 有界修订：被证伪则补充调研，最多 2 轮
    - id: patch
      needs: [verify]
      when: "${ len(nodes.verify.outputs.refuted) > 0 }"
      loop:
        maxIterations: 2
        until: "${ len(nodes.verify.outputs.refuted) == 0 }"
        do:
          uses: "${ nodes.pick.outputs.selected[0].name }"
          with:
            question: "${ inputs.question }"
            angle: "refuted: ${ toJson(nodes.verify.outputs.refuted) }"

    # 5) 汇总（inline：短步骤，无需隔离）
    - id: summarize
      uses: summarize
      needs: [patch]
      join: any
      mode: inline
      with:
        question: "${ inputs.question }"
        survived: "${ nodes.verify.outputs.survived }"
        refuted: "${ nodes.verify.outputs.refuted }"

  result:
    answer:   "${ nodes.summarize.outputs.text }"
    evidence: "${ nodes.verify.outputs.survived }"
    refuted:  "${ nodes.verify.outputs.refuted }"
```

## 12. 规范演进

| 版本 | 变更 |
|---|---|
| `v1alpha1` | 初稿：Flow/Node/Edge、NEL、控制流糖、6 种内核节点、校验规则 18 条 |

兼容性承诺：`apiVersion` 变更即可能不兼容；同版本内新增可选字段不视为破坏性变更。未知字段**报错**而非忽略（失败关闭）。
