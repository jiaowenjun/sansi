# AI Agent「三思而后行」与按需思考 Skill 调研报告

> **调研日期：** 2026-11-20  
> **筛选原则：** 仅按与「三思而后行」和「按需思考」的理念相关性排序，不考虑 Stars、下载量和社区热度。  
> **核心目标：** 根据任务复杂度、风险和不确定性，决定初始推理投入；在执行中动态调整；达到足够质量后及时停止。<br>
> **源码复核版：** `Reasoning Workflow 3.2.0`、`Deep Research 1.13.0` 与 Self-Refine 的完整证据见[本地资料汇总](local_reference_summary.md)

## 一、调研结论

本次选取三个 Skill：

| 排名 | Skill | 最相关的设计理念 | 最值得借鉴的部分 |
|---|---|---|---|
| 1 | Reasoning Workflow | 通用任务的自适应思考深度 | Light → Standard → Deep → Max → Ultra，动态升降深度 |
| 2 | Deep Research | 按复杂度设置初始预算，并按信息缺口迭代 | 复杂度分级、轮数上限、搜索预算、停止判断 |
| 3 | Self-Refine | 按任务难度分配自我审查次数 | Generate → Critique → Refine → Check，分级复核与收敛 |

**我的总体建议：不要简单地把三个 Skill 叠加使用，而是将它们的核心机制整合成一个独立的 Adaptive Deliberation Skill（自适应审慎思考）。**

三个项目分别解决不同问题：

- Reasoning Workflow 解决「这个任务值得投入多少思考」。
- Deep Research 解决「复杂任务如何分配和调整探索预算」。
- Self-Refine 解决「初步答案产生后，应该复核几次，以及何时停止」。

最终应形成一个闭环：

`任务评估 → 初始预算分配 → 执行与观察 → 必要时增加预算 → 独立复核 → 收敛停止`

---

# 二、第一名：Reasoning Workflow

**项目：** [Agent--deep-research-Workflow-Skills](https://github.com/zy839971925-zyy/Agent--deep-research-Workflow-Skills)

**核心 Skill：** [reasoning-workflow/SKILL.md](https://github.com/zy839971925-zyy/Agent--deep-research-Workflow-Skills/blob/main/skills/reasoning-workflow/SKILL.md)

本地核查固定到提交 `c4de05d`。仓库不只有 Prompt：还包含 task profile、routing、runtime、host adapter、事件/状态 Schema 和交付验证脚本。本地 `python3 tests/run_all.py` 通过 82 项测试；但 `RELEASE.json` 同时明确记录 `model_compliance_evaluated=false`、`remote_published=false`。因此这些测试证明软件契约可执行，不证明真实模型会稳定遵守五级深度，也不证明任务质量提升。

这是三个项目中最接近「让 Agent 根据任务难度决定思考深度」这一目标的方案。

## 2.1 核心设计理念

传统 Skill 通常采用固定流程：

1. 接收任务。
2. 按照预先定义的步骤执行。
3. 完成任务。
4. 输出结果。

这种方式存在明显问题：

- 简单任务不需要复杂的分析流程，却可能被迫执行多轮思考。
- 复杂任务需要更深入的分析，却可能因为流程过于简单而遗漏关键问题。
- 任务执行过程中可能出现新的不确定性，但初始流程无法动态调整。

Reasoning Workflow 采用不同的思路：

**先判断任务需要多深的工作，再决定加载什么能力、执行什么流程。**

它不是一个要求所有任务都执行完整流程的 Skill，而是一个负责管理其他 Skill 和工作流程的上层入口。

## 2.2 实现逻辑：五级思考深度

它采用五级深度：

| 深度 | 适用场景 | 思考策略 |
|---|---|---|
| Light | 简单、明确、低风险的问题 | 直接回答，进行必要检查 |
| Standard | 常规分析、一般检索任务 | 处理重要信息缺口，检查结果 |
| Deep | 多因素分析、复杂技术问题 | 分析竞争性解释、证据质量和边界条件 |
| Max | 高不确定性、关键决策 | 针对主要瓶颈深入调查，进行独立挑战 |
| Ultra | 特别复杂的系统性问题 | 重新审视问题定义、证据体系、遗漏和共同失效路径 |

这里有两个非常值得借鉴的设计。

### 设计一：采用最小充分深度

它要求选择能够充分完成任务的最低深度，而不是默认使用最高深度。

例如：

- 「Rust 中 `Option` 和 `Result` 有什么区别？」→ Light。
- 「比较三个 Rust 异步运行时的性能与生态」→ Deep。
- 「设计一个长期运行的量化交易系统，并分析其风险控制架构」→ Max 或 Ultra。

这避免了将「认真思考」错误地理解成「每次都进行最大规模的思考」。

### 设计二：思考深度可以动态变化

它不是在任务开始时确定一次等级就不再改变。

例如，初始任务可能被判断为 Standard，但在执行中发现：

- 关键资料之间存在矛盾；
- 重要假设无法验证；
- 出现新的因果关系；
- 多个方案的结论相互冲突；
- 结果的不确定性显著高于预期。

这时应当提升到 Deep，必要时提升到 Max。

反过来，如果发现问题实际上比预期简单，也可以降低深度。

因此，深度是一个持续更新的运行时判断，而不是一次性的分类结果。

## 2.3 实现逻辑：任务画像与能力路由

Reasoning Workflow 不建议仅根据用户输入的字数或关键词判断复杂度，而是综合考虑：

- 任务复杂度；
- 不确定性；
- 信息时效性；
- 因果关系和系统耦合程度；
- 错误后果；
- 新证据改变最终答案的可能性。

它还会根据任务类型决定加载哪些能力：

- Core Reasoning：基础问题理解与推理。
- Deep Research：需要证据调查和多来源验证时。
- Decision Analysis：需要比较、取舍和决策时。
- Execution Control：需要执行持续性任务或管理外部操作时。
- Audit / Verification：需要深入核验结果时。

值得注意的是，它强调渐进式加载，而不是一开始就把所有参考文件和所有 Skill 全部加载进上下文。

这是一种很好的 Token 管理思想：**先用少量上下文判断任务，再按实际需要加载更多推理资源。**

## 2.4 如何落地

可以直接借鉴其五级深度体系，但建议把「任务复杂度」和「模型推理预算」分开。

例如定义：

```yaml
depth_profiles:
  light:
    description: "简单明确的问题"
    reasoning_effort: low
    review_rounds: 0
    tool_budget: minimal

  standard:
    description: "一般分析任务"
    reasoning_effort: medium
    review_rounds: 1
    tool_budget: moderate

  deep:
    description: "复杂、多因素任务"
    reasoning_effort: high
    review_rounds: 1
    tool_budget: expanded

  max:
    description: "高风险或高不确定性任务"
    reasoning_effort: xhigh
    review_rounds: 2
    tool_budget: high

  ultra:
    description: "系统性、关键性复杂任务"
    reasoning_effort: max
    review_rounds: 2
    tool_budget: very_high
```

以上是一个供 Runtime 实现的示例配置，不是该项目原有的配置，也不代表所有模型都支持这些 effort 值。

实际使用时，应当根据模型能力映射到可用参数。

## 2.5 优点与不足

**优点**

1. 与按需思考的目标高度一致。
2. 不强制所有任务走完整流程。
3. 支持动态提高或降低思考深度。
4. 将任务路由、推理深度和能力加载联系起来。
5. 强调最小充分投入，避免无意义的多 Agent 和多轮分析。

**不足**

1. 五级深度主要是工作流层面的指导，不等于模型内部实际推理 Token 数量。
2. 深度等级的选择仍然依赖模型自身判断，缺乏独立的复杂度评估器时可能出现误判。
3. 如果没有 Runtime 支持，`reasoning_effort` 等配置只是描述性约定，不能保证真正生效。
4. 五级体系需要结合具体任务校准，不能认为 Ultra 一定比 Max 更准确。

**最值得借鉴：** 五级深度、最小充分投入、动态重新评估、渐进式加载。

---

# 三、第二名：Deep Research Skill

**项目：** [moonlight-lupin/agent-skills](https://github.com/moonlight-lupin/agent-skills)

**核心 Skill：** [research/deep-research/SKILL.md](https://github.com/moonlight-lupin/agent-skills/blob/main/research/deep-research/SKILL.md)

本地核查固定到 `moonlight-agent-skills@e943e32` 的 Deep Research 1.13.0。当前源码已经包含 evidence store、quote anchoring、claim/citation/report gates、反证检查和 source-bias 检查；5 个以上来源的报告要求通过三个 deterministic gate，不应再概括为只有搜索轮数和停止 Prompt。测试入口本次因环境缺少 `pytest` 而未运行，错误为 `ModuleNotFoundError: No module named 'pytest'`。

这个 Skill 专门面向深度调研任务，但其预算分配和停止机制具有很强的通用参考价值。

## 3.1 核心设计理念

Deep Research 将研究过程设计为一个循环：

`Think → Search → Extract → Synthesize → Stop`

而不是简单地执行固定数量的搜索。

它首先判断任务复杂度，设置一个最大工作预算，然后在每一轮结束时重新评估：

- 已经解决了多少问题？
- 还有哪些重要信息缺口？
- 是否存在相互矛盾的证据？
- 是否需要进一步搜索？
- 当前投入是否已经足够？

这解决了一个重要问题：

**初始预算应当根据任务复杂度设定，但实际投入不应当机械地消耗完预算。**

## 3.2 实现逻辑：复杂度分级

该 Skill 将研究任务分成三个级别：

| 复杂度 | 最大研究轮数 | 每轮搜索广度 | 适用场景 |
|---|---:|---:|---|
| Simple | 2 | 2 | 单一事实、范围明确的问题 |
| Moderate | 3 | 3 | 多方面问题、对比分析、操作指南 |
| Complex | 5 | 4 | 多领域综合分析、复杂系统问题 |

例如：

- 查询 Python 当前版本：Simple。
- 比较 Qdrant 与 pgvector：Moderate。
- 全面分析自托管 AI 基础设施与托管 API 的成本和技术权衡：Complex。

这里的轮数和搜索广度是该项目针对深度研究的具体参数，不应直接照搬到所有 Agent 任务。

更重要的是它明确规定：

**最大轮数是上限，不是目标。**

Simple 任务如果第一轮就得到充分答案，应当立即停止，不需要为了达到两轮而继续搜索。

## 3.3 实现逻辑：初始预算与动态预算

它通过复杂度确定初始搜索预算：

```text
Search Budget = Per-round Breadth × Max Rounds
```

例如：

```text
Simple:
  breadth = 2
  max_rounds = 2
  budget = 4

Moderate:
  breadth = 3
  max_rounds = 3
  budget = 9

Complex:
  breadth = 4
  max_rounds = 5
  budget = 20
```

这是一种简单但有效的资源分配机制。

在执行过程中：

1. 第一轮根据研究计划搜索。
2. 提取信息并形成当前结论。
3. 识别尚未解决的问题。
4. 第二轮针对缺口生成新查询。
5. 更新综合结论。
6. 判断是否还需要继续。

它不是每一轮重复相同的工作，而是让后续投入由前一轮发现的信息缺口驱动。

## 3.4 实现逻辑：停止条件

停止判断是这个 Skill 很值得借鉴的部分。

它在每一轮综合信息后，重新判断答案是否已经足够完整。

停止条件主要包括：

- 研究计划中的核心问题已经得到回答；
- 关键事实有足够证据支撑；
- 重要矛盾已经解决或明确记录；
- 继续搜索的预期收益已经较低；
- 剩余预算不足以支持更多有效工作。

特别值得注意的是，它要求后续轮次针对信息缺口，而不是简单地增加搜索数量。

例如，假设研究一个开源编辑器：

第一轮发现：
- 编辑器 A 支持桌面端和移动端；
- 编辑器 B 支持丰富的扩展机制；
- 编辑器 C 在移动端输入体验方面有优势。

第二轮不应该再次搜索三个编辑器的介绍，而应该针对关键缺口调查：
- A 的移动端键盘适配是否存在已知问题？
- B 的扩展机制是否影响包体积和维护成本？
- C 的表格和数学公式能力是否满足需求？

这才是真正有信息增益的追加思考。

## 3.5 实现逻辑：Token Budget Awareness

该 Skill 还会在综合研究状态时关注 Token 预算。

当上下文逐渐变长时，需要：
- 压缩已经得到的结论；
- 保留关键证据和引用；
- 避免重复读取已经掌握的信息；
- 不要因为上下文压缩而丢失重要结论；
- 在预算不足时优先完成核心问题。

这可以迁移到通用 Agent：

**思考预算不只是决定思考多少，还需要决定哪些信息值得保留在后续思考中。**

## 3.6 如何落地

可以将其复杂度分级机制扩展到一般任务：

```yaml
task_budget:
  simple:
    max_reasoning_rounds: 0
    max_review_rounds: 0

  moderate:
    max_reasoning_rounds: 1
    max_review_rounds: 1

  complex:
    max_reasoning_rounds: 3
    max_review_rounds: 2

  critical:
    max_reasoning_rounds: 5
    max_review_rounds: 3
```

这里的 reasoning rounds 指可观察的任务级推理/执行循环，而不是模型内部不可见的思维步骤。

每一轮结束后生成一个简短的状态：

```yaml
progress:
  resolved_questions:
    - "问题 A"
    - "问题 B"

  unresolved_questions:
    - "问题 C"

  uncertainty:
    - "某项结论缺少独立证据"

  next_best_action:
    - "针对问题 C 搜索一手资料"

  should_continue: true
  reason: "核心结论仍缺少关键证据"
```

这样 Agent 就可以根据实际进展决定是否增加预算。

## 3.7 优点与不足

**优点**

1. 复杂度与初始预算直接关联。
2. 预算具有明确上限。
3. 任务执行过程中能够动态调整。
4. 每一轮都围绕信息缺口展开。
5. 具有明确的停止判断。
6. 考虑上下文长度和 Token 预算。

**不足**

1. 主要针对研究任务，不能直接覆盖所有代码、写作和决策任务。
2. 搜索轮数不等于模型内部推理深度。
3. 预算分级主要是规则配置，不是经过普遍验证的最优参数。
4. 如果任务复杂度判断错误，初始预算仍然可能不足或过量。

**最值得借鉴：** 初始预算、预算上限、缺口驱动的追加投入、每轮停止判断。

---

# 四、第三名：Self-Refine Skill

**项目：** [Thomaszhou22/self-refine-skill](https://github.com/Thomaszhou22/self-refine-skill)

**核心 Skill：** [SKILL.md](https://github.com/Thomaszhou22/self-refine-skill/blob/main/SKILL.md)

它最直接地对应「三思而后行」中的反思、检查和修改机制。

## 4.1 核心设计理念

Self-Refine 的核心思想是：

不要直接把第一次生成的答案交付给用户，而是先检查答案是否存在问题，再决定是否需要修改。

其基本循环：

```text
GENERATE
   ↓
CRITIQUE
   ↓
REFINE
   ↓
CHECK
   ↓
DELIVER
```

其中：

- Generate：生成初始答案。
- Critique：检查初始答案的缺陷。
- Refine：针对缺陷进行修改。
- Check：确认修改是否真正改善结果。
- Deliver：达到质量要求后交付。

它不要求每个任务都执行完整的反思循环，而是根据任务复杂度选择不同深度。

## 4.2 实现逻辑：四级反思深度

该 Skill 设计了四个等级：

| 等级 | 适用任务 | 最大复核轮数 | 项目给出的额外 Token 成本估计 |
|---|---|---:|---:|
| L0 | 简单问答 | 0 | 0% |
| L1 | 一般任务 | 1 | 约 15% |
| L2 | 复杂技术任务 | 2 | 约 30% |
| L3 | 高风险、对抗性任务 | 3 | 约 50% |

这些成本数字是项目自身的估计，不是所有模型、任务和运行环境下的固定成本。

这套设计的关键是：

**任务越复杂，允许的复核次数越多；但复核次数存在上限。**

例如：

- 解释一个简单概念：L0。
- 编写普通邮件：L1。
- 设计复杂的软件架构：L2。
- 审查安全敏感代码或高风险决策：L3。

## 4.3 实现逻辑：六个检查维度

Self-Refine 将 Critique 过程分成六个维度：

| 检查维度 | 具体检查内容 |
|---|---|
| Logical Completeness | 推理是否完整，有无逻辑断层 |
| Factual Accuracy | 事实是否准确，有无未经验证的断言 |
| Response Completeness | 是否回答了用户所有关键问题 |
| Conciseness | 是否存在冗余内容 |
| Actionability | 建议是否具体、可执行 |
| Internal Consistency | 不同部分之间是否存在矛盾 |

这比简单地要求模型「再检查一遍」更有操作性。

因为模型需要针对明确的质量维度进行审查，而不是泛泛地重新生成一遍答案。

## 4.4 实现逻辑：收敛与停止

Self-Refine 最重要的部分不是反复修改，而是知道什么时候停止。

它采用的停止逻辑包括：

1. 没有发现重要缺陷；
2. 剩余问题只是风格偏好；
3. 继续修改已经无法带来明显收益；
4. 已达到最大复核轮数。

这能够避免一个常见问题：

第一次答案可能已经足够好，但模型不断修改，最后反而引入新的错误。

因此，复核不应该等同于强制改写。

如果没有发现值得修改的问题，应该保留原答案。

## 4.5 实现逻辑：Reflexion Memory

项目还借鉴了 Reflexion 的思路。

当模型在复核中发现重复出现的错误模式时，可以把经验记录下来，以便之后避免同类问题。

例如：

```text
历史错误：
- 技术方案比较时忽略了移动端 Safari。
- 推荐开源库时没有检查最近维护状态。
- 代码修改后只检查编译，没有检查实际运行效果。
```

这些经验可以转化为未来任务的检查规则。

但需要注意：

不要将每次偶然错误都永久保存为规则。只有重复出现、能够推广且有证据支持的错误模式，才值得沉淀。

## 4.6 如何落地

建议将 Self-Refine 的六个检查维度作为通用复核器：

```yaml
review:
  dimensions:
    - logical_completeness
    - factual_accuracy
    - response_completeness
    - conciseness
    - actionability
    - internal_consistency

  stopping:
    max_rounds: 3
    stop_if_no_critical_issue: true
    stop_if_only_style_changes: true
    stop_if_diminishing_returns: true
```

复核时不应该直接要求模型重写，而是先输出缺陷列表：

```yaml
critique:
  critical_issues:
    - "结论缺少关键证据"

  minor_issues:
    - "第二部分存在重复解释"

  style_issues:
    - "标题可以更简洁"

  decision: refine
```

如果没有重要缺陷：

```yaml
critique:
  critical_issues: []
  minor_issues: []
  decision: accept
```

如果发现缺陷，则只修改对应部分，并在修改后重新检查。

这比每次都从头生成答案更稳定，也更容易控制 Token 成本。

## 4.7 项目自测结果的可信度

Self-Refine 项目提供了一份 20 道合成题、五类任务、单次运行的自测报告，报告声称整体评分提升约 21.2%。生成与评分由同一模型完成。

但这不能直接当作普遍有效的实验结论：

- 测试规模较小；
- 主要由同类模型生成和评价；
- 评价标准具有一定主观性；
- 缺乏充分的消融实验；
- 没有多随机种子重复、置信区间或延迟测量；
- 没有证明所有任务类型都能获得相同收益。

因此，我认为它的**流程设计有参考价值，但其量化提升幅度不应直接用于预估实际收益**。

**最值得借鉴：** 分级复核、六维 Critique、只修改真实缺陷、收敛停止、错误经验沉淀。

---

# 五、三个 Skill 的综合比较

| 维度 | Reasoning Workflow | Deep Research | Self-Refine |
|---|---|---|---|
| 任务开始时评估复杂度 | 强 | 强 | 强 |
| 根据复杂度决定初始投入 | 五级深度 | 复杂度与预算映射 | 四级复核深度 |
| 执行中动态调整 | 强 | 强 | 主要体现在复核循环 |
| 控制额外思考轮数 | 间接 | 明确 | 明确 |
| 明确停止条件 | 有 | 有 | 有 |
| 适用任务范围 | 通用 | 研究为主 | 通用输出复核 |
| 多能力路由 | 强 | 研究流程内部 | 弱 |
| 自我纠错 | 有 | 证据缺口驱动 | 核心机制 |
| 实际模型 Token 控制 | 需要 Runtime | 需要 Runtime | 需要 Runtime |

三个项目并不是互相替代的关系。

更合适的组合方式：

```text
              User Task
                  │
                  ▼
       Reasoning Workflow
       任务分类与初始深度
                  │
                  ▼
          Deep Research
       复杂任务预算与动态调整
                  │
                  ▼
           Self-Refine
       结果复核与迭代收敛
                  │
                  ▼
             Deliver
```

但这不是要求每个任务都依次执行三个 Skill。

实际应该是按需激活：

- 简单问答：只需要 Reasoning Workflow 的 Light 判断。
- 普通任务：Reasoning Workflow + 一轮必要复核。
- 深度调研：Reasoning Workflow + Deep Research。
- 复杂技术方案：Reasoning Workflow + 有针对性的 Self-Refine。
- 高风险任务：根据实际瓶颈增加独立验证，而不是机械地增加自我反思次数。

---

# 六、如何将理念落地为一个通用 Adaptive Deliberation Skill

我更建议自己实现一个轻量的上层 Skill，而不是直接安装三个庞大的工作流。

建议命名：

`adaptive-deliberation`

目标：

> 根据任务复杂度、不确定性和错误后果，动态分配初始推理预算；执行过程中根据新信息调整投入；通过独立复核和收敛判断，确保投入与任务价值相匹配。

## 6.1 总体架构

```text
┌──────────────────────────────┐
│          User Task           │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│       Task Assessment        │
│                              │
│  Complexity                  │
│  Uncertainty                 │
│  Consequence                 │
│  Scope                       │
│  Verifiability               │
│  Reversibility               │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│       Budget Allocation      │
│                              │
│  Initial Effort              │
│  Reasoning Rounds             │
│  Tool Budget                 │
│  Review Budget               │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│       Task Execution         │
│                              │
│  Understand                  │
│  Reason                      │
│  Research / Act              │
│  Observe                     │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│      Re-evaluation Gate      │
│                              │
│  New uncertainty?            │
│  Contradiction?              │
│  Missing evidence?           │
│  Unexpected failure?         │
└──────────────┬───────────────┘
               │
         Need more work?
           /       \
         Yes        No
          │          │
          ▼          ▼
    Increase      Self-Review
    Budget        & Verify
          │          │
          └────┐     ▼
               │  Deliver
               │
               └── Back to Execution
```

## 6.2 第一层：Task Assessment

建议不要只用一个简单的复杂度分数。

至少评估以下六个维度：

| 维度 | 评估问题 |
|---|---|
| Complexity | 是否涉及多步骤、多领域、多变量？ |
| Uncertainty | 关键事实、前提和因果关系是否明确？ |
| Consequence | 判断错误会造成多大影响？ |
| Scope | 任务范围是否广泛，是否需要多个独立子问题？ |
| Verifiability | 是否存在可靠方法验证结论？ |
| Reversibility | 错误是否容易修正，是否涉及不可逆操作？ |

这几个维度不能简单地等权相加。

例如：

- 一个简单的资金转账操作，复杂度可能很低，但错误后果很高。
- 一个复杂的概念解释，复杂度可能很高，但错误后果相对较低。
- 一个技术调研任务，复杂度中等，但由于信息时效性较高，可能需要额外验证。

因此，建议采用规则路由，而不是仅凭一个 0–100 分数决定所有事情。

## 6.3 第二层：初始预算分配

建议设置五级预算：

| 级别 | 推理投入 | 适用场景 |
|---|---|---|
| L0 | 极低 | 简单事实、直接转换、明确指令 |
| L1 | 低 | 一般解释、常规写作、简单分析 |
| L2 | 中 | 多步骤问题、普通技术比较 |
| L3 | 高 | 复杂技术设计、深度调研、重要决策 |
| L4 | 极高 | 高风险、系统性问题、关键结论验证 |

建议初始映射：

```yaml
budget_profiles:
  L0:
    effort: low
    reasoning_rounds: 0
    review_rounds: 0

  L1:
    effort: low
    reasoning_rounds: 1
    review_rounds: 0

  L2:
    effort: medium
    reasoning_rounds: 2
    review_rounds: 1

  L3:
    effort: high
    reasoning_rounds: 3
    review_rounds: 2

  L4:
    effort: xhigh
    reasoning_rounds: 5
    review_rounds: 3
```

这是一份初始工程配置，应该通过实际任务测试进行校准。

特别要注意：

- `effort` 是模型级推理强度参数。
- `reasoning_rounds` 是任务级工作循环次数。
- `review_rounds` 是输出复核次数。

三者不是同一个概念，不能混为一谈。

## 6.4 第三层：动态预算调整

任务开始时的复杂度判断不可能完全准确。

因此，每次完成一个重要步骤后，都应该检查是否需要重新分配预算。

建议设置以下升级条件：

| 触发条件 | 调整方式 |
|---|---|
| 发现重要前提不成立 | 重新分析问题 |
| 出现关键证据冲突 | 增加调查与验证 |
| 多次尝试仍然失败 | 更换策略或提高深度 |
| 发现新的关键变量 | 扩展任务模型 |
| 结论依赖未经验证的假设 | 增加独立复核 |
| 出现高影响、难逆转的决策 | 提高验证强度 |

也要允许降低预算：

- 关键问题已经解决；
- 新证据没有改变原有判断；
- 后续步骤只是机械执行；
- 继续分析无法带来新的有效信息。

这可以避免任务进入无休止的思考循环。

## 6.5 第四层：Self-Review

在交付前，根据任务风险决定是否复核。

建议把复核分成两种：

**普通自我复核**

由同一个模型检查：
- 是否回答完整；
- 是否有逻辑漏洞；
- 是否遗漏关键条件；
- 是否存在前后矛盾。

**独立验证**

当任务重要性较高时，使用不同的验证路径：
- 运行测试；
- 检查原始资料；
- 使用确定性工具计算；
- 让独立 Agent 审查；
- 对关键结论寻找反证。

需要特别强调：同一个模型重复阅读自己的答案，不等于独立验证。真正有价值的独立验证应当尽量采用不同的信息来源或不同的失败路径。

## 6.6 第五层：Stopping Criteria

建议明确规定以下停止条件：

1. 用户要求的目标已经实现。
2. 关键问题已经得到充分回答。
3. 没有尚未解决的关键缺陷。
4. 重要结论已经完成必要验证。
5. 继续投入的预期收益低于成本。
6. 已达到预算上限。

达到预算上限但仍存在关键不确定性时，不应伪装成已经得到确定结论，而应明确说明限制。

---

# 七、Skill 与 Runtime 必须分层

这是整个落地方案中最重要的工程问题。

`SKILL.md` 本质上是给 Agent 的工作流程指令。它可以告诉模型：

- 先判断任务复杂度；
- 简单问题直接回答；
- 复杂问题进行多轮分析；
- 发现不确定性时增加调查；
- 输出前检查答案；
- 达到停止条件后结束。

但它不能保证模型内部真的分配了更多推理 Token。

如果希望真正控制模型的计算预算，需要 Runtime 支持。

建议分成两层：

## 7.1 Skill Layer

负责决策：

```text
任务是什么？
需要多深的思考？
为什么需要更多思考？
应该采用什么验证方法？
是否需要追加预算？
何时停止？
```

## 7.2 Runtime Layer

负责执行：

```text
选择模型支持的 reasoning effort
设置 thinking budget（如果 API 支持）
设置 max output tokens
控制工具调用次数
控制 Agent / Subagent 数量
管理多轮调用
统计 Token 和实际成本
```

例如，假设 Runtime 支持动态设置 effort：

```typescript
type DeliberationLevel =
  | "light"
  | "standard"
  | "deep"
  | "max";

type DeliberationBudget = {
  level: DeliberationLevel;
  reasoningEffort: string;
  maxToolCalls: number;
  maxReviewRounds: number;
};
```

Skill 负责决定 `level`，Runtime 负责把它转换为模型真正支持的参数。

如果模型只支持固定的推理配置，则 Skill 仍然可以通过多轮调用、工具预算和复核流程控制任务级投入，但不能声称控制了模型内部推理预算。

## 7.3 关于模型原生 Adaptive Thinking

Anthropic 的官方文档已经提供了一个很好的工程参照：

[Steering thinking](https://platform.claude.com/docs/en/build-with-claude/thinking-steering-and-cost)

其设计允许模型根据每次请求的复杂度，决定是否需要思考以及思考多少；开发者通过 effort 参数和 Prompt 提供软性引导。

官方文档明确指出：
- 简单事实问题可能直接回答，不产生额外思考块；
- 多步骤数学问题或复杂调试任务更可能触发深入思考；
- effort 是软性指导，不保证固定 Token 数量；
- 模型可以在工具调用之间继续思考。

这说明「按需思考」不只是 Skill 层面的理念，也可以由模型和 Runtime 原生支持。

如果底层模型本身具备 Adaptive Thinking，建议优先使用模型原生机制，再用 Skill 补充任务级预算、验证和停止策略，而不是用大量 Prompt 强制模型重复思考。

---

# 八、建议的最终实现：Adaptive Deliberation Skill

我建议最终只实现一个主 Skill，避免多个思考类 Skill 互相叠加、重复检查。

推荐目录：

```text
adaptive-deliberation/
├── SKILL.md
├── references/
│   ├── task-assessment.md
│   ├── budget-profiles.md
│   ├── dynamic-reassessment.md
│   ├── review-protocol.md
│   └── stopping-criteria.md
└── scripts/
    ├── estimate-budget.py
    └── evaluate-result.py
```

### SKILL.md 的核心逻辑

```markdown
# Adaptive Deliberation

## Objective

Allocate reasoning effort according to task complexity,
uncertainty, consequence, and expected value of further analysis.

The goal is not to maximize thinking.
The goal is to use the minimum sufficient reasoning
to produce a reliable result.

## 1. Assess

Before solving a substantive task, assess:

- Complexity
- Uncertainty
- Consequence
- Scope
- Verifiability
- Reversibility

Do not use task length as a proxy for complexity.

## 2. Allocate

Choose the lowest sufficient deliberation level:

- L0: Direct response
- L1: Lightweight reasoning
- L2: Structured reasoning
- L3: Deep reasoning and verification
- L4: Extensive investigation and independent review

Respect explicit user instructions and runtime limits.

## 3. Execute

Perform the task using the selected budget.

Do not add reasoning rounds without a concrete purpose.

## 4. Reassess

Increase or decrease the budget when new evidence,
contradictions, uncertainty, failures, or scope changes
materially affect the task.

Every additional round must have a specific objective.

## 5. Review

Review according to task risk.

Check:
- Logical completeness
- Factual accuracy
- Response completeness
- Internal consistency
- Verifiability

Use independent verification for consequential claims
when supported by available tools.

## 6. Stop

Stop when:
- The requested outcome is achieved.
- Material issues are resolved.
- Required verification is complete.
- Further work has low expected value.
- The budget limit is reached.

Never claim certainty when important uncertainty remains.
```

---

# 九、如何验证这个 Skill 是否真的有效

不能仅凭「模型看起来思考得更认真」就认定 Skill 有效。

建议设计一套对照实验。

## 9.1 实验组

| 实验组 | 策略 |
|---|---|
| A | 原始模型，单次回答 |
| B | 固定 Medium effort |
| C | 固定 High effort |
| D | 固定三轮 Self-Refine |
| E | Adaptive Deliberation |
| F | Adaptive Deliberation + 独立验证 |

这样可以区分：
- 单纯提高模型推理强度的收益；
- 增加复核次数的收益；
- 根据任务复杂度动态分配预算的收益；
- 独立验证带来的额外收益。

## 9.2 测试任务

建议覆盖：

1. 简单事实问答；
2. 普通知识解释；
3. 多步骤数学问题；
4. 复杂技术方案比较；
5. 代码调试；
6. 长文档分析；
7. 多来源研究；
8. 高风险决策分析。

需要确保测试集既有简单任务，也有真正复杂的任务。

否则无法验证「简单任务少思考、复杂任务多思考」是否真的有效。

## 9.3 评价指标

| 指标 | 评价目标 |
|---|---|
| Accuracy | 答案正确率 |
| Task Success | 任务完成率 |
| Completeness | 关键问题覆盖率 |
| Hallucination Rate | 幻觉率 |
| Regression Rate | 复核是否引入新错误 |
| Token Cost | 实际 Token 消耗 |
| Latency | 完成时间 |
| Cost per Success | 每个成功任务的平均成本 |

最重要的不是单纯提高正确率，而是：

**在相同成本下提高正确率，或在维持正确率的同时降低成本。**

## 9.4 实验注意事项

- 固定模型版本和工具权限。
- 使用相同的测试任务进行配对比较。
- 尽可能使用独立评价者或人工盲评。
- 记录实际 Token，而不是只记录配置的 effort。
- 区分模型内部推理 Token、可见输出 Token 和工具调用成本。
- 分别分析简单任务与复杂任务，不能只看总体平均分。
- 至少重复多次，以减小模型输出随机性带来的影响。

---

# 十、最终建议

如果目标是打造一个真正实用的「三思而后行、按需思考」Skill，我建议：

**第一阶段：实现 Reasoning Workflow 的核心理念。**

先实现任务画像、五级深度、最小充分投入和动态重新评估。这是整个系统的决策核心。

**第二阶段：借鉴 Deep Research 的预算机制。**

引入初始预算、最大预算、信息缺口驱动的追加投入和停止条件。让每一次额外思考都有明确目的。

**第三阶段：引入 Self-Refine 的复核机制。**

根据任务风险分配复核次数，优先检查真实缺陷，不强制反复改写。

**第四阶段：接入 Runtime。**

如果使用的模型支持原生 Adaptive Thinking，就利用模型原生机制；如果支持 effort 或 thinking budget，就由 Runtime 动态设置；如果不支持，则通过任务级多轮调用、工具预算和独立验证实现有限的外部控制。

**最重要的设计原则：**

1. 简单任务不要过度思考。
2. 复杂任务不要过早结束。
3. 高风险任务不能只依赖模型自我判断。
4. 每次增加思考投入，都应该有明确的信息增益目标。
5. 复核不是为了修改而修改。
6. 达到足够质量后及时停止。
7. 预算不足时诚实说明不确定性，而不是输出虚假的确定性。

最终目标不是让 Agent 永远思考三次，而是让它知道：

**什么时候应该直接回答，什么时候应该认真推理，什么时候应该停下来重新审视，什么时候必须进一步验证。**

## 参考资料

### Skill 项目

1. [Reasoning Workflow — SKILL.md](https://github.com/zy839971925-zyy/Agent--deep-research-Workflow-Skills/blob/main/skills/reasoning-workflow/SKILL.md)
2. [Deep Research — SKILL.md](https://github.com/moonlight-lupin/agent-skills/blob/main/research/deep-research/SKILL.md)
3. [Self-Refine — SKILL.md](https://github.com/Thomaszhou22/self-refine-skill/blob/main/SKILL.md)
4. [Self-Refine — Benchmark](https://github.com/Thomaszhou22/self-refine-skill/blob/main/BENCHMARK.md)

### 官方技术文档

5. [Anthropic — Steering thinking](https://platform.claude.com/docs/en/build-with-claude/thinking-steering-and-cost)
6. [Anthropic — Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)
7. [Anthropic — Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)

### 相关研究

8. [Self-Refine: Iterative Refinement with Self-Feedback (Madaan et al., 2023)](https://arxiv.org/abs/2303.17651)
9. [Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al., 2023)](https://arxiv.org/abs/2303.11366)

---

*源码说明：本报告的设计评价与效果证据分开处理。结构化 Gate、Schema 和测试可以约束工作流，但不能单独证明来源真实、模型遵循或准确率提升；完整版本快照和复跑记录见[本地资料汇总](local_reference_summary.md)。*
