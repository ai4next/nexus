# 00 · 总览

## 1. 问题

Agent 技能生态已经解决了「能力供给」问题：技能可以被发现、被合并去重、按名加载、按调用策略区分模型/用户入口。但它没有解决「能力组合」问题。

当一项任务需要多个技能协作时，今天的做法通常是：

1. 把技能目录塞进上下文，让模型自己决定加载顺序；
2. 或者写一段一次性编排脚本（命令式、不可恢复、不可审计）；
3. 或者由人手工在多个会话之间搬运中间产物。

这三条路分别有各自的硬伤：

| 做法 | 硬伤 |
|---|---|
| 模型自行决定顺序 | 无契约、不可复现、上下文随步数膨胀、失败无法定位到具体环节 |
| 一次性脚本 | 不可恢复、不可审计、无法复用、无法治理（预算/权限/审批全缺） |
| 人工搬运 | 不可规模化，中间产物丢失，人成为瓶颈 |

根因是缺少一个**控制平面**：它知道「哪些技能」之外，还知道「以什么顺序、在什么条件下、携带什么数据、由哪个 Agent、在什么约束下执行」。

## 2. 定位

Nexus 就是这一层控制平面。一句话：

> **Nexus 把技能注册表从「能力目录」升级为「可执行的组合契约」。**

分层关系：

```
┌─────────────────────────────────────────────────────────┐
│  任务 (Task)                                             │
└───────────────────────┬─────────────────────────────────┘
                        │ 路由 / 规划
┌───────────────────────▼─────────────────────────────────┐
│  组合层 (Composition)   Flow DSL · Router · 校验器        │
│  —— 声明式、可版本化、可评审、可静态校验                    │
└───────────────────────┬─────────────────────────────────┘
                        │ 编译
┌───────────────────────▼─────────────────────────────────┐
│  执行层 (Execution)     Scheduler · Executors · Journal   │
│  —— 调度、隔离、重试、检查点、预算、审批、审计              │
└───────────────────────┬─────────────────────────────────┘
                        │ 解析 / 加载
┌───────────────────────▼─────────────────────────────────┐
│  能力层 (Capability)    技能注册表 (已有，复用不重建)       │
│  —— 发现、去重、优先级、按需加载技能正文                    │
└─────────────────────────────────────────────────────────┘
```

**关键设计选择：Nexus 不重建技能注册表。** 它把注册表当作只读的 `Catalog` 依赖：`list()` 拿候选（用于路由），`get(name)` 拿正文（用于执行）。这带来两个好处：技能作者不需要学新格式；Nexus 可以直接继承注册表已有的作用域分层、优先级、调用策略与热更新能力。

## 3. 设计原则

| # | 原则 | 含义 | 反面 |
|---|---|---|---|
| P1 | **图即上下文契约** | 节点只能看到自己声明的输入，看不到编排者的完整对话历史 | 把所有历史透传给每个节点 → 上下文膨胀、行为不可预测 |
| P2 | **声明式优先，命令式为逃生舱** | 能用 DSL 表达的编排必须声明式；确需任意代码时显式降级到脚本节点并标记 | 编排逻辑藏在脚本里 → 无法静态校验、无法恢复、无法审计 |
| P3 | **一切决策可记录、可重放** | 路由、审批、预算、重试决策全部写入日志，附输入 | 只记录结果不记录决策 → 线上问题无法归因 |
| P4 | **默认失败关闭** | 校验失败、权限越界、预算超限、审批超时 → 停止，而不是降级放行 | 静默降级 → 产生看似成功实则错误的产物 |
| P5 | **权限与预算单调不扩张** | 子节点/子流程的权限与预算只能是父级的子集 | 模型规划的子图自我提权 → 越权执行 |
| P6 | **模型输出是数据，不是指令** | 模型产出的计划/路由结果必须经同一套静态校验器验证 | 直接执行模型生成的编排 → 提示注入即任意执行 |
| P7 | **无界循环不存在** | 任何迭代结构必须声明上界 | `while(true)` 类结构 → 成本不可控 |
| P8 | **正文可信，输入不可信** | 技能正文是本地可信内容；技能产出的工件/引文是数据 | 把外部抓取内容当指令执行 |

## 4. 概念模型

```
Skill（技能）      原子能力单元。由技能注册表提供：name/description/whenToUse/content。
   │
   │ 被引用
   ▼
Node（节点）       一次技能调用。= SkillRef + 输入绑定 + 执行模式 + 策略。
   │
   │ 组成
   ▼
Flow（编排）       节点与依赖构成的有向无环图 + 输入/输出契约 + 权限与预算。
   │
   │ 实例化
   ▼
Run（运行）        一次执行实例。持有节点状态、工件、日志、预算计数。
   │
   ├── Envelope（信封）   节点返回的结构化结果，必须匹配节点声明的输出 schema。
   ├── Artifact（工件）   内容寻址的大载荷，按引用传递。
   └── Journal（日志）    仅追加的事件序列，是恢复与审计的唯一依据。
```

辅助概念：

| 概念 | 说明 |
|---|---|
| **Catalog** | 技能注册表的只读视图，Nexus 的候选来源 |
| **Router** | 依据任务描述从 Catalog 中选出技能的策略组件 |
| **Select 节点** | 路由的运行时载体：纯函数，无副作用，产出候选而非直接执行 |
| **Gate 节点** | 审批/校验检查点，可暂停 Run 等待人 |
| **Executor** | 节点执行策略：`inline` / `subagent` / `delegate` |
| **Plan** | Flow 经校验与降级后得到的内核 IR，Scheduler 只认 Plan |
| **Permission** | 能力声明：工具、文件系统、网络、可路由技能集合 |
| **Budget** | 层级化配额：节点数、深度、时长、token、成本、并发 |

### 4.1 Flow 也是 Skill（递归组合）

一个 Flow 可以被打包成一个技能对外暴露：它的 `description`/`whenToUse` 描述「何时该用这条编排」，正文是流程说明。于是：

- 编排可以嵌套编排（子流程），深度受预算约束；
- 已有的技能发现、路由、调用策略机制**无需改动**即可复用；
- 一条精心设计的编排可以被别的编排当作一个原子步骤使用。

这是整个设计中复用度最高的一点：**组合层与能力层同构**。

## 5. 端到端走读

任务：「调研仓库现状，产出架构改造方案，实现并验证，关键改动需人工确认」。

```yaml
apiVersion: nexus/v1alpha1
kind: Flow
metadata: { name: repo-refactor, version: 1.0.0 }
spec:
  inputs:
    type: object
    required: [repo, goal]
    properties:
      repo: { type: string }
      goal: { type: string }

  permissions:
    skills: { allow: ["repo-*", "arch-design", "code-edit", "test-*", "review-*"] }
    tools:  { allow: ["read", "grep", "glob", "bash", "write", "edit"] }
    filesystem: { write: ["${ inputs.repo }/**"] }

  budget: { maxNodes: 40, maxDepth: 3, maxTokens: 3000000, maxWallClock: 2h, maxConcurrency: 4 }

  defaults:
    mode: subagent
    policy: { timeout: 20m, retry: { max: 2, backoff: exponential } }

  nodes:
    # 1) 调研：上下文密集，隔离到子 Agent
    - id: recon
      uses: repo-explorer
      with: { repo: "${ inputs.repo }", goal: "${ inputs.goal }" }
      outputs:
        type: object
        required: [findings]
        properties: { findings: { type: object } }

    # 2) 方案：需要调研结论作为唯一输入
    - id: design
      uses: arch-design
      needs: [recon]
      with: { requirements: "${ nodes.recon.outputs.findings }" }
      outputs:
        type: object
        required: [summary, proposal]
        properties: { summary: {type: string}, proposal: {type: object} }

    # 3) 对抗评审：扇出 3 个独立评审者，法定人数 2 通过
    - id: critique
      forEach:
        over: [security, performance, maintainability]
        as: lens
        concurrency: 3
        do:
          uses: review-adversarial
          with: { lens: "${ item }", proposal: "${ nodes.design.outputs.proposal }" }
        reduce:
          join: "quorum:2"
          merge: array-collect
          minSuccess: 2

    # 4) 条件门：评审未通过则回到设计（有界循环）
    - id: revise
      needs: [critique]
      when: "${ nodes.critique.outputs.verdict == 'changes_requested' }"
      loop:
        maxIterations: 3
        until: "${ nodes.critique.outputs.verdict == 'approved' }"
        do:
          uses: arch-design
          with: { requirements: "${ nodes.critique.outputs.findings }" }

    # 5) 动态路由：由 Router 从白名单技能中选实现者
    - id: pick
      uses: nexus.select
      needs: [critique, revise]
      join: any
      select:
        task: "implement: ${ nodes.design.outputs.summary }"
        candidates: { tags: [implementation], source: [project-dsh, bundled] }
        strategy: llm-select
        max: 1
        allowDynamic: true
      outputs:
        type: object
        required: [selected]
        properties: { selected: { type: array } }

    # 6) 执行：uses 由上一步结果绑定（受 allowDynamic 与白名单约束）
    - id: implement
      uses: "${ nodes.pick.outputs.selected[0].name }"
      needs: [pick]
      with: { plan: "${ nodes.design.outputs.proposal }" }

    # 7) 验证
    - id: verify
      uses: test-runner
      needs: [implement]
      policy: { onError: fail }

    # 8) 人工审批：暂停点
    - id: approve
      uses: nexus.gate
      needs: [verify]
      gate:
        kind: approval
        prompt: "批准合入？"
        preview: "${ nodes.implement.outputs.diff }"
        timeout: 24h
        onTimeout: reject

  result:
    outcome: "${ nodes.implement.outputs }"
    evidence: "${ nodes.verify.outputs }"
    approved: "${ nodes.approve.outputs.approved }"
```

这条流程一次性覆盖了本设计要解决的全部问题：

| 走读点 | 对应能力 | 文档 |
|---|---|---|
| `recon` 隔离执行 | 多智能体：子 Agent 上下文隔离 | [04](04-multi-agent.md) |
| `critique` 扇出 + 法定人数 | 并行、join 语义、部分失败 | [02](02-execution.md) |
| `revise` 有界循环 | 无界循环禁止、收敛条件 | [01](01-dsl.md) · [02](02-execution.md) |
| `pick` 动态选技能 | 技能自动路由 | [03](03-routing.md) |
| `implement` 的 `uses` 是表达式 | 动态引用 + 白名单 + 权限单调 | [03](03-routing.md) · [05](05-governance.md) |
| `approve` 暂停 | 审批门、检查点、恢复 | [05](05-governance.md) |
| `budget` / `permissions` | 治理 | [05](05-governance.md) |
| 全流程可重放 | 日志、录制回放 | [02](02-execution.md) |

## 6. 术语表

| 术语 | 英文 | 定义 |
|---|---|---|
| 编排 | Flow | 一份可版本化的 DAG 规范，`kind: Flow` |
| 节点 | Node | Flow 中的一次技能调用或内建步骤 |
| 端口 | Port | 节点声明的具名类型化输出 |
| 绑定 | Binding | 把表达式结果接到节点输入的声明 |
| 表达式 | NEL | Nexus Expression Language，CEL 子集 |
| 内核 IR | Plan | 降级糖语法后的规范化执行图 |
| 运行 | Run | 一次 Flow 实例 |
| 信封 | Envelope | 节点的结构化返回，必须匹配输出 schema |
| 工件 | Artifact | 内容寻址的载荷，按引用传递 |
| 日志 | Journal | 仅追加事件序列，恢复与审计的依据 |
| 就绪集 | Ready Set | 当前可执行的节点集合 |
| 法定人数 | Quorum | join 策略：至少 n 个上游成功 |
| 弃权 | Abstain | 路由主动放弃选择并要求升级处理 |
| 展开 | Expansion | 模型规划产出的子图，需经校验后并入 |

## 7. 非目标

明确不做，避免范围蔓延：

1. **不是通用工作流引擎**——不执行任意确定性业务代码，不替代 Temporal/Airflow 的场景。
2. **不是技能注册表的替代品**——发现、去重、优先级、热更新仍归能力层。
3. **不是可视化编排器**——v1 不做拖拽画布；DSL 可被工具生成，但规范不依赖 UI。
4. **v1 不做分布式调度**——单进程/单会话内调度，持久化用于恢复而非水平扩展。
5. **v1 不做补偿事务（saga）**——副作用回滚由技能自身负责，编排层只提供幂等键与审计。
6. **不保证 LLM 的确定性**——只保证「调度确定性」与「决策可重放」，模型随机性通过录制回放处理。

## 8. 阅读顺序

- 想理解**为什么这样设计** → 本文档 §2–§3
- 想**写一条编排** → [01 编排 DSL](01-dsl.md)
- 想理解**运行时行为** → [02 执行语义](02-execution.md)
- 想做**自动选技能** → [03 技能路由](03-routing.md)
- 想做**多 Agent 协作** → [04 多智能体](04-multi-agent.md)
- 关心**安全与成本** → [05 治理与安全](05-governance.md)
- 想**实现它** → [06 参考实现](06-reference-impl.md)
- 想知道**先做什么** → [07 路线图与权衡](07-roadmap.md)
