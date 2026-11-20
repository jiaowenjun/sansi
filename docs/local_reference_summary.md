# 自适应审慎思考：本地论文与参考 Skill 源码汇总

> **核查日期：** 2026-11-20<br>
> **资料范围：** `papers/` 中 8 篇 PDF 全文，`reference-skills/` 中 5 个 Git 仓库<br>
> **核查方式：** PDF 全文抽取与逐页定位、首页渲染检查、源码与测试阅读、可执行测试复跑<br>
> **文档定位：** 为 [论文综述](research_review.md)、[Skill 调研（一）](skill_review_v1.md) 和 [Skill 调研（二）](skill_review_v2.md) 提供统一的本地证据底稿

## 1. 结论摘要

本地资料支持的结论不是“只要让模型多想几次就会更准确”，而是以下四点：

1. **额外计算的收益高度不均匀。** 简单问题可能很早进入负边际收益，困难问题才可能从更长推理、更多候选或多 Agent 中获益。
2. **论文中的自适应机制通常依赖 Prompt 之外的能力。** 隐藏状态、Token 熵、带标签校准、模型多次采样、全局队列和实际成本计量都需要模型接口或 Runtime 支持。
3. **普通 Skill 最适合控制可观察的任务流程。** 它可以做任务分级、问题分解、证据门槛、复核目标和停止条件，但不能声称直接控制模型内部推理深度。
4. **现有开源 Skill 的工程成熟度和效果证据必须分开看。** 测试通过可以证明解析器、Schema、Hook 或验证器符合软件契约，不能自动证明模型准确率得到提升。

因此，Adaptive Deliberation 最合理的形态是分层系统：

```text
Skill：任务画像、策略路由、复核目标、证据与停止规则
  ↓
Runtime：模型/effort、采样次数、工具与 Agent、Token/延迟预算
  ↓
Verifier：测试、引用、规则、独立评审和真实任务结果
  ↓
Telemetry：质量、成本、回归与停止决策的可追溯记录
```

## 2. 本地资料快照

### 2.1 论文 PDF

| 编号 | 本地文件 | 页数 | 主要对象 |
|---:|---|---:|---|
| 1 | [Plan and Budget](../papers/01_plan_and_budget_iclr2026.pdf) | 25 | 子问题级预算分配 |
| 2 | [When More Thinking Hurts](../papers/02_when_more_thinking_hurts_acl2026_findings.pdf) | 11 | 负边际收益与过度思考 |
| 3 | [Adaptive Thinking / Sonata](../papers/03_adaptive_thinking_iclr2026.pdf) | 17 | Prefill 阶段思考门控 |
| 4 | [EAGer](../papers/04_eager_icml2026.pdf) | 15 | Token 熵驱动的分支采样 |
| 5 | [Think Deep, Not Just Long](../papers/05_think_deep_not_just_long_icml2026.pdf) | 19 | 内部推理深度指标 DTR |
| 6 | [AERA](../papers/06_aera_arxiv2608.27964.pdf) | 9 | 剩余证据价值与在线停止 |
| 7 | [Multi-Agent Reasoning](../papers/07_multi_agent_reasoning_acl2026_srw.pdf) | 14 | 多种测试时扩展策略的成本前沿 |
| 8 | [The Shadow Price of Reasoning](../papers/08_shadow_price_of_reasoning_icml2026.pdf) | 15 | 批量级全局预算配置 |

页数以本地 PDF 的物理页数为准。论文的发表状态、模型和数据集不同，不能把各篇报告的最高提升直接横向相加。

### 2.2 参考源码

| 仓库 | 本地提交 | 提交日期 | 本次重点 |
|---|---|---|---|
| [unlazy](../reference-skills/unlazy/) | `1667149` | 2026-09-03 | Gate、证据绑定、调度与 Stop Hook |
| [cc-thinking-skills](../reference-skills/cc-thinking-skills/) | `7b8fece` | 2026-08-07 | 28 个思维 Skill、路由与证据审计 |
| [self-refine-skill](../reference-skills/self-refine-skill/) | `620cb9b` | 2026-11-15 | Prompt 驱动的生成、批判、修订循环 |
| [Reasoning Workflow](../reference-skills/Agent--deep-research-Workflow-Skills/) | `c4de05d` | 2026-11-19 | 五级深度、路由、状态契约和验证脚本 |
| [moonlight-agent-skills](../reference-skills/moonlight-agent-skills/) | `e943e32` | 2026-11-11 | Deep Research 1.13.0 的证据与报告 Gate |

这些目录由 `.gitignore` 排除；提交哈希用于固定本次结论所对应的源码版本。

## 3. 八篇论文逐篇核查

### 3.1 Plan and Budget：理论预算模型与工程启发式

**问题。** 固定 Token 上限会同时造成简单子问题浪费和困难子问题预算不足。论文把复杂推理拆成连续子问题，研究如何在总预算下分配计算。

**机制。** BAM（Budget Allocation Model）给出理论形式，描述子问题难度、预算和成功概率的关系。但实际系统无法在推理时完整估计 BAM 所需参数；落地实现采用子问题复杂度权重与位置衰减等启发式规则，再通过 Plan-and-Budget Prompt 约束执行。它是“理论指导的启发式调度”，不是持续测量真实不确定性的在线控制器。

**实验。** 论文在多种推理模型与任务上报告最高 70% 的准确率增益、39% 的 Token 降低和 193.8% 的 E3 指标提升；这些都是跨设置的最佳结果，不代表同一个配置同时取得全部最高值，也不是通用 Agent 任务保证。

**边界。** 子问题划分和复杂度判断仍受模型质量影响；预算分配的有效性依赖任务能否合理分解。论文证据主要来自可自动判分的推理基准。

**可迁移设计。** Skill 可以采用“先分解、再按风险与难度分配任务级预算”，但应把复杂度分数标记为代理判断，并允许执行后根据失败证据重新分配。

### 3.2 When More Thinking Hurts：推理长度存在负边际收益

**问题。** 测试时计算扩展常默认更长推理单调改善准确率，这篇论文直接测量每增加 500 Token 的边际收益，以及答案从正确翻转为错误的事件。

**机制与设置。** 研究使用 budget forcing，在 500 到 16,000 Token 之间以 500 为步长控制推理，分析 AIME 2024、MATH-500 和 GPQA Diamond；主要比较 R1-32B 与 s1-32B。指标包括边际效用、正/负答案翻转、答案振荡和反思性语言信号。

**关键结果。** AIME 上 R1-32B 的准确率在约 12K 达峰，7K 后负翻转开始超过正翻转；GPQA Diamond 约在 10K 达峰。MATH-500 的最优预算随难度从约 1K 到 7.5K，相差 7.5 倍。组合早停信号取得约 97% 的峰值准确率，同时显著减少计算；论文还检查了 312 个自然产生超过 8K Token 的样本，以降低 budget forcing 人工效应的疑虑。

**边界。** 结果来自少数推理模型与可判分基准；“过度思考”在编程、研究或工具使用任务中可能呈现不同信号。早停信号具有误判风险，不能替代任务验收。

**可迁移设计。** 每次追加复核都应有具体目标；若新一轮只重复旧观点、引入答案振荡或连续没有新证据，应停止或切换为外部验证，而不是继续同一路径自我反思。

### 3.3 Adaptive Thinking / Sonata：Prefill 阶段的模型专属门控

**问题。** 能否在正式生成长推理前，预测一个问题是否值得使用思考模式？

**机制。** Sonata 从基础模型最后一层隐藏表示取特征，训练两层 MLP Adapter。训练标签由多次采样的一致性和 ground truth 构造。主算法主要做二元“思考/不思考”决策；论文另有多级预算扩展实验，但不能把扩展写成主方法的默认能力。

**实验。** 在 Qwen3、GPT-OSS 以及 AIME25、GSM8K、MATH500、GPQA、LiveCodeBench 等设置中，论文报告在保持相近准确率时减少约 20%–60% 的思考 Token，或在相同成本下最高提升约 2 个百分点；Adapter 推理开销低于 0.1%。

**边界。** Adapter 可跨任务迁移，但与底层模型绑定，不能直接跨模型复用。标签构建依赖可判分答案，且方法要求访问隐藏状态。它不等价于一个通用 Prompt Router。

**可迁移设计。** Skill 可借鉴“先门控再投入”的思想，用风险、失败历史、证据缺口和可逆性作为外部代理；若要复现论文机制，则必须由模型侧或 Runtime 暴露隐藏表示并训练独立门控器。

### 3.4 EAGer：熵驱动分支，但三个版本的监督条件不同

**问题。** Best-of-N 对所有问题平均采样，不能把候选预算集中到模型真正不确定的位置。

**机制。** `EAGER-init` 在初始生成的高熵 Token 处分支；`EAGER-adapt` 在无标签测试时根据任务批次动态重分配预算；完整 `EAGER` 使用目标标签识别失败问题，再把资源集中到这些问题上。完整版本只适合数学、代码、RLVR 等存在可验证答案的场景。

**实验。** 论文在有目标标签的设置中报告 Pass@k 最高提升约 37%、Token 减少 59%；测试时无标签设置最高提升约 12%、Token 减少 64%。两组数字来自不同条件，不能合并成同一方法的统一收益。

**边界。** 方法需要 Token 概率/熵和分支生成接口，普通 Skill 无法读取。高熵也不总等于任务级高风险；自然语言开放任务缺少目标标签时，完整版本无法直接使用。

**可迁移设计。** 外部 Skill 可把方案分歧、测试失败、引用冲突和未解决问题作为熵的代理，只在这些节点生成替代路径。

### 3.5 Think Deep, Not Just Long：内部深度比长度更有解释力

**问题。** 推理 Token 很长可能只是重复；论文试图区分真正触发模型内部预测修正的 Token。

**机制。** Deep-Thinking Token 依赖各网络层的 hidden state、unembedding 后的预测分布及层间 JSD，Deep-Thinking Ratio（DTR）统计这类 Token 的比例。Think@n 先生成约 50-Token 前缀，估计候选的 DTR，再筛选值得继续生成的路径。

**实验。** 在 AIME 24/25、HMMT 25、GPQA-Diamond 及 GPT-OSS、DeepSeek-R1、Qwen3 等设置中，DTR 对正确率的预测优于单纯长度和常用置信度指标。Think@n 大约筛掉一半候选，把成本降到标准 self-consistency 的约一半，同时保持或改善表现。

**边界。** 该指标依赖白盒内部状态和模型输出头，普通 API 与 Prompt Skill 通常无法计算。50-Token 前缀筛选的效果也依赖候选早期表示能否预测后续质量。

**可迁移设计。** 工程上应记录“实质性假设修正、有效新证据、独立验证通过”而非只记轮数或输出长度；这是理念迁移，不是 DTR 的等价实现。

### 3.6 AERA：窄范围在线证据，不是通用停止器

**问题。** 当前多数票或置信度不能直接回答“再采样是否仍有价值”。AERA 学习预测继续生成后剩余的纠错空间。

**机制。** 控制器综合答案分布、时间序列、重新求解、语义和已用计算等特征，决定停止或继续。训练和校准按 benchmark 分开进行，不存在已验证的跨域通用 Gate。

**关键结果。** 最严格的 frozen-threshold 在线验证使用 50 道 GSM8K 校准题和 300 道未触碰测试题。AERA 平均使用 4.44 次响应，准确率 92.61%，Fixed-128 为 93.01%；配对差异的 95% CI 为 `[-1.79, +0.80]`。Completion Token 节省 95.99%，283/300 问题在 4 次响应时停止。它与 Fixed-128 的准确率差异不显著，但对所有固定预算也没有显著优势。

**边界。** GPQA 在线压力测试未满足预设校准约束；论文明确指出响应次数只是成本代理，未测量端到端吞吐、能耗和生产开销。现有证据只支持“GSM8K 上经校准的控制器可接近满预算准确率并大幅减少采样”，不支持“通用 Agent 停止器已被验证”。

**可迁移设计。** 停止规则应经过目标任务校准，并同时报告质量差异的置信区间与真实成本，不能只展示平均 Token 节省。

### 3.7 Multi-Agent Reasoning：高预算下的多样性收益

**问题。** 在给定计算预算下，self-consistency、self-refinement、debate 和 mixture-of-agents（MoA）哪一种更接近 Pareto 最优？

**设置。** 论文在 MMLU-Pro 和 BBH 上每个评估点使用 `n=1000`，模型为非 reasoning 的 4-bit Llama 3.1 8B/70B。成本由 FLOPs 和模型权重 memory transfer 的理论公式换算，不是线上实测延迟。

**结果。** 在约 20× CoT 的最高预算区域，MoA 在 MMLU-Pro 上达到 71.4%，相较 self-consistency 的 68.7% 高 2.7 个百分点；debate 为 70.0%，高 1.3 个百分点。MoA 在高预算下更稳健，但预算不超过约 3× CoT 时，self-consistency 更实际。

**边界。** 论文是 ACL Student Research Workshop 工作；模型、量化方式、理论成本模型和两个基准限制了外推。多 Agent 也可能共享训练偏见，增加视角不等于独立真值验证。

**可迁移设计。** 低预算优先少量独立候选和可执行验证；只有任务价值与预算足够时才启用多 Agent 聚合，并单独计量并发、上下文和聚合成本。

### 3.8 The Shadow Price of Reasoning / CLEAR：批量级全局预算分配

**问题。** 当系统同时服务一批查询且总预算有限时，如何把计算分给最可能从额外推理中获益的查询？

**机制。** CLEAR 以影子价格表示计算资源的边际价值，使用 DeBERTa-v3-base 预测每个问题达到不同质量阈值所需的预算，再在批量层面做全局约束优化。系统允许对部分问题“理性放弃”，把预算转给更有希望的问题。

**实验。** 论文在极低总预算下报告相较平均分配最高约 3 倍的整体准确率；主要收益来自资源非常紧张以及允许放弃部分查询的场景。它不是单查询内部逐步决定思考多久的方法。

**边界。** 阈值预测器依赖域内训练和质量标签；如果每个用户请求都必须交付，理性放弃策略并不适用。随着预算提高，收益会收窄。

**可迁移设计。** 对后台批处理、评测队列或批量数据处理，可以学习任务级收益曲线做全局调度；对交互式单请求，则只能借鉴边际价值概念，不能直接套用 CLEAR。

## 4. 论文横向矩阵

| 方法 | 决策粒度 | 关键观测 | 额外训练 | 普通 Prompt 可复现 | 主要适用边界 |
|---|---|---|---|---|---|
| Plan-and-Budget | 单查询内的子问题 | 启发式复杂度、位置 | 否 | 部分 | 可合理分解的推理任务 |
| Overthinking 分析 | 推理长度/停止点 | 边际效用、答案翻转 | 否 | 只能用代理 | 少数 reasoning model 基准 |
| Sonata | 请求级门控 | 最后一层隐藏状态 | 是，且模型专属 | 否 | 有校准标签、可访问模型内部 |
| EAGer | Token 分支与批次 | Token 熵、失败标签 | 完整版需标签 | 否 | 可分支、可判分任务 |
| DTR / Think@n | 候选路径 | 各层分布与 JSD | 否 | 否 | 白盒模型 |
| AERA | 多次响应后的停止 | 轨迹、语义、采样历史 | 是，按域校准 | 否 | 当前仅严格验证 GSM8K 在线 |
| Multi-Agent | 候选/Agent 配置 | 输出多样性、理论成本 | 否 | 需 Runtime | 高预算、可并行场景 |
| CLEAR | 查询批次 | 收益阈值、全局预算 | 是，域内训练 | 否 | 允许跨请求调度和放弃 |

## 5. 五个参考 Skill 源码核查

### 5.1 unlazy：完成纪律与可执行证据

**Prompt 层。** [SKILL.md](../reference-skills/unlazy/SKILL.md) 规定 Depth Tree、Leaf 所有权、四轮工作流程和根任务回查。重点不是“想得更聪明”，而是把未完成项和验收条件外部化。

**Runtime 层。** `gate-check.mjs` 解析并运行经授权的检查；`gate-lint.mjs` 检查 Gate 质量；lease/dispatch 机制管理并行 Leaf；Stop Hook 在结构条件未满足时阻止提前结束。命令执行与 approval 绑定，证据带身份和新鲜度约束。

**验证层。** Gate 包含 `CHECK`、`EXPECT` 和证据，形成 Leaf、Parent、Branch、Root 多层验证。它适合代码、文件和可观察状态，不适合用模型自评替代事实验证。

**本地复跑。** `npm test` 全部通过，主要套件分别为 34/34、27/27、51/51、24/24、29/29、8/8，自检 15/15。

**证据边界。** 测试证明 Gate 解析、授权、证据绑定、租约和 Hook 的软件行为；没有证明使用该 Skill 会普遍提高模型推理正确率。

### 5.2 cc-thinking-skills：方法目录和显式路由

**Prompt 层。** 仓库提供 28 个 active Skill，包括具体思维模型、router 和 combination；每个 Skill 说明触发条件、反例、步骤与输出结构。Router 依据 Problem Fit、输入可得性、执行时间、使用者可用性和应用能力评分。

**Runtime 层。** 主要依赖宿主加载对应 Skill，没有统一的模型预算控制器。仓库的脚本集中在结构校验、评测和证据登记，而非多轮模型编排。

**验证层。** `validate-skills.js` 和 structural eval 检查目录与元数据；证据框架设置自动保留门槛，并允许把证据不足的 Skill 隔离。

**本地复跑。** 28/28 个 Skill 结构校验通过，结构评测通过。

**证据边界。** 当前证据审计将 28 个 active Skill 全部标记为 `manual-only quarantine`，自动确认提升为 0。`scientific-method` 的初步 `+4pp` 低于项目 `+5pp` 门槛。这里的“28”包含 router 和 combination，不能写成 28 个都经过效果验证的独立思维模型。

### 5.3 self-refine-skill：轻量 Prompt 循环

**Prompt 层。** [SKILL.md](../reference-skills/self-refine-skill/SKILL.md) 采用 Generate → Critique → Refine → Check，按任务难度给出不同复核深度，并用逻辑、事实、完整性、简洁性、可执行性和一致性等维度检查输出。

**Runtime 层。** 仓库没有执行脚本或独立控制器；循环由同一个宿主模型按提示完成。“Zero API Cost”只能理解为无额外外部服务，不能理解为没有额外 Token、延迟或平台成本。

**验证层。** 主要是模型自我批判与可选 Reflexion Memory，没有独立测试、引用 Gate 或事实验证器。

**项目自报证据。** `+21.2%` 来自 20 道合成题、同模型生成与评分、单次运行；没有独立评审、多随机种子、置信区间或延迟测量。因此只能视为示范性自测，不能用于预测通用收益。

### 5.4 Reasoning Workflow 3.2.0：工作流契约比模型效果证据更成熟

**Prompt 层。** [核心 Skill](../reference-skills/Agent--deep-research-Workflow-Skills/skills/reasoning-workflow/SKILL.md) 定义 Light、Standard、Deep、Max、Ultra 五级深度，并根据复杂度、不确定性、时效性、系统耦合、错误后果和信息价值进行 profile/routing。

**Runtime 层。** 仓库包含 `prepare_task.py`、`route_task.py`、`route_resources.py`、`runtime_core.py`、host adapter、事件/状态 Schema、恢复与学习记录，已经超过纯提示词范畴。

**验证层。** Delivery、dual verification、event、raw evidence、state、workflow 等验证脚本检查跨文件契约和可追溯性；打包和版本文件固定发布内容。

**本地复跑。** `python3 tests/run_all.py` 通过 82 项测试。

**证据边界。** [RELEASE.json](../reference-skills/Agent--deep-research-Workflow-Skills/skills/reasoning-workflow/RELEASE.json) 明确记录 `model_compliance_evaluated=false`、`remote_published=false`。82 项测试验证代码与数据契约，不验证真实模型是否遵循五级策略，也不证明任务质量提升。

### 5.5 Deep Research 1.13.0：证据卫生与确定性 Gate

**Prompt 层。** [核心 Skill](../reference-skills/moonlight-agent-skills/research/deep-research/SKILL.md) 使用 Think → Search → Extract → Synthesize → Stop 循环，以信息缺口驱动后续轮次，而不是机械耗尽最大轮数。

**Runtime 层。** `source_db.py` 维护来源与证据记录，`research_validation.py` 执行报告验证，`url_health.py` 检查链接健康。搜索、来源判断和综合仍由模型及宿主工具完成。

**验证层。** 当前版本不仅有简单轮数限制，还包含 evidence store、quote anchoring、claim/citation/report gates、反证检查和 source-bias 检查。5 个以上来源的报告要求通过三个 deterministic gate。

**本地复跑。** 测试未执行成功，因为当前环境缺少 `pytest`，入口报 `ModuleNotFoundError: No module named 'pytest'`；本次未为文档核查额外改动环境。

**证据边界。** 确定性 Gate 可以证明引用格式、覆盖关系和记录完整性符合规则，不能证明来源本身真实可靠，也不能证明模型综合出的结论为真。

## 6. 源码能力横向比较

| 项目 | Prompt/流程 | 可执行 Runtime | 客观/结构化验证 | 当前效果证据 |
|---|---|---|---|---|
| unlazy | 强：分解、四轮流程 | 强：Gate、lease、Hook | 强：命令与证据绑定 | 软件测试充分；无质量提升实验 |
| cc-thinking-skills | 强：方法路由 | 弱：以校验脚本为主 | 中：结构与证据审计 | 28 个均 manual-only quarantine |
| self-refine-skill | 中：反思循环 | 无 | 弱：主要自评 | 20 道合成题自报 +21.2% |
| Reasoning Workflow | 强：五级深度与路由 | 强：状态、Schema、adapter | 强：契约与交付验证 | 82 测试；未评估模型遵循性 |
| Deep Research | 强：缺口驱动研究 | 中：证据库与验证器 | 强：引用和报告 Gate | 测试本次未复跑；无事实正确性证明 |

## 7. 论文机制到 Skill 设计的映射

| 论文启示 | Skill 可实现 | 必须由 Runtime/模型实现 | 验证要求 |
|---|---|---|---|
| 子问题差异化预算 | 分解、风险分级、轮数上限 | 每阶段真实 Token/effort | 比较各阶段成本与成功率 |
| 负边际收益与答案翻转 | 无新信息则停、保留已验证答案 | 多检查点生成与回滚 | 记录每轮净修正和回归 |
| Sonata 先门控 | 外部任务画像 | 隐藏状态 Adapter | 目标域校准、误门控率 |
| EAGer 不确定节点分支 | 对冲突/失败点开替代路径 | Token 熵与生成分支 | Pass@k、分支成本 |
| DTR 重质量不重长度 | 记录信息增量和假设修正 | 白盒 DTR/JSD | 独立质量指标，不看字数 |
| AERA 预测剩余价值 | 明确停止问题 | 学习型在线控制器 | 冻结阈值、CI、分布漂移 |
| 多 Agent 高预算扩展 | 指定独立角色和聚合 Gate | 并行调度与成本计量 | 与简单 self-consistency 对照 |
| CLEAR 全局调度 | 标记任务价值和可放弃性 | 队列级优化器 | 公平性、SLA、被放弃请求 |

## 8. 建议的实现边界

### 8.1 第一阶段：可验证的纯 Skill

首版只承诺可观察行为：

- 对任务按复杂度、风险、不确定性、可验证性和可逆性分级；
- 选择最低充分深度，不用输入长度代替复杂度；
- 为每次追加分析指定明确问题和停止条件；
- 对代码、研究、决策分别选择测试、引用或独立反方验证；
- 保留已验证结论，防止复核引入回归；
- 达到上限仍不确定时显式报告，不制造虚假确定性。

### 8.2 第二阶段：Runtime 控制

Runtime 才负责：

- 设置模型和 `reasoning_effort` / thinking budget；
- 控制候选数、上下文、工具调用、Agent 数和并发；
- 在失败、冲突或新证据出现时追加预算；
- 记录每轮 Token、延迟、费用和验证结果；
- 支持检查点、回滚和批量级资源调度。

### 8.3 第三阶段：目标域校准

在目标任务集上比较基线、固定高预算和自适应策略。至少报告：

- 准确率或任务成功率及置信区间；
- Token、延迟、工具调用、Agent 调用和货币成本；
- 复核发现缺陷率与复核引入回归率；
- 简单/困难任务的门控误差；
- 成本/成功任务，以及达到预算上限仍失败的比例。

没有这一步，不应声称 Adaptive Deliberation 已经提高准确率。

## 9. 本次可复现验证记录

| 对象 | 命令/检查 | 结果 |
|---|---|---|
| 8 篇 PDF | `pdfinfo`、`pdftotext -layout`、首页渲染 | 全部可读取，首页渲染正常 |
| unlazy | `npm test` | 全部套件通过 |
| cc-thinking-skills | `node scripts/validate-skills.js` 与 structural eval | 28/28，结构评测通过 |
| Reasoning Workflow | `python3 tests/run_all.py` | 82 tests passed；`model_evaluated=false` |
| Deep Research | Python 测试入口 | 未运行：缺少 `pytest` |

## 10. 最终建议

可以直接复用的不是某个“万能深思 Prompt”，而是以下组合：

1. 用 Reasoning Workflow 的任务画像和最低充分深度做入口。
2. 用 unlazy 的验收 Gate 和证据绑定约束“完成”。
3. 用 Deep Research 的证据存储、引用锚定和缺口驱动循环约束研究任务。
4. 把 cc-thinking-skills 当作候选分析方法库，而不是已验证的自动路由器。
5. 把 self-refine 作为低成本复核模板，只修真实缺陷，且始终防范同模型自评偏差。
6. 将模型 effort、采样、多 Agent 和成本控制留给 Runtime；将效果主张留给目标域 Benchmark。

这一路线与八篇论文的共同证据一致：**自适应的关键不是思考次数本身，而是把额外计算投到可识别的瓶颈上，并用独立证据判断何时足够。**
