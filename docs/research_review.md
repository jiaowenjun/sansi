# Adaptive Deliberation：基于自适应推理预算的 AI Agent 准确率优化研究

> **调研主题**：通过增加思考次数、推理深度和计算成本，提高 AI Agent 的任务准确率  
> **核心方向**：Adaptive Test-Time Compute Allocation（自适应测试时计算分配）  
> **调研时间**：2025-11-20 至 2026-11-20  
> **论文范围**：仅纳入上述时间范围内发表或公开的研究  
> **文档版本**：1.0

---

## 1. 研究背景

### 1.1 研究问题

传统 LLM 通常采用一次性生成（One-pass Generation）的方式完成任务：

1. 接收用户指令。
2. 生成推理过程。
3. 输出最终答案。

这种方式存在一个问题：无论任务简单还是复杂，模型可使用的推理资源通常受到相同机制的限制。

对于简单任务，模型可能投入过多计算，产生冗余推理；对于复杂任务，模型可能没有投入足够的计算资源，导致分析不完整、逻辑错误或遗漏关键信息。

因此，一个值得研究的问题是：

**能否根据任务复杂度、错误风险和推理过程中的不确定性，动态增加思考次数、推理深度和验证次数，以额外计算成本换取更高的任务准确率？**

### 1.2 核心研究方向

这一问题在当前学术研究中主要涉及以下方向：

| 研究方向 | 核心思想 |
|---|---|
| Test-Time Compute Scaling | 增加模型推理阶段的计算资源 |
| Inference-Time Scaling | 在推理阶段通过更多计算改善输出 |
| Adaptive Compute Allocation | 根据任务难度动态分配计算预算 |
| Iterative Refinement | 多轮生成、批判和修正 |
| Self-Consistency | 生成多条独立推理路径并比较结果 |
| Multi-Agent Reasoning | 多个 Agent 独立分析、辩论和交叉验证 |
| Adaptive Stopping | 根据边际收益决定是否继续思考 |

这些研究并不完全等价。

尤其需要区分：

- 增加单条推理链的长度。
- 增加独立推理路径的数量。
- 增加推理搜索深度。
- 增加答案验证和修正次数。

不同方法适合不同任务，不能简单地将它们统称为增加思考次数。

---

## 2. 近一年代表性论文

### 2.1 Plan and Budget: Effective and Efficient Test-Time Scaling on Reasoning Large Language Models

**发表信息**

- 作者：Junhong Lin、Xinyue Zeng、Jie Zhu、Song Wang、Julian Shun、Jun Wu、Dawei Zhou
- 会议：ICLR 2026
- 论文：[ICLR 官方页面](https://proceedings.iclr.cc/paper_files/paper/2026/hash/ae8d4084f418bb51575c2ca6c658a05b-Abstract-Conference.html)
- 代码：[GitHub - P-and-B](https://github.com/junhongmit/P-and-B)

**研究问题**

LLM 在推理过程中存在两种问题：

- Overthinking：简单问题产生过长、偏离主题的推理。
- Underthinking：复杂问题没有获得足够的推理资源。

固定 Token Budget 虽然可以控制计算成本，但可能导致简单任务浪费预算、复杂任务推理不足。

**核心方法：Plan-and-Budget**

作者提出 BAM（Budget Allocation Model），将任务推理过程视为一系列具有不同不确定性的子问题。

具体流程：

1. 将复杂问题拆分为多个子问题。
2. 估计不同子问题的复杂程度。
3. 根据复杂度为子问题分配 Token Budget。
4. 通过自适应调度调整计算资源。
5. 在保证答案质量的同时优化计算效率。

**论文报告的实验结果**

- 最高准确率提升：70%。
- Token 消耗减少：39%。
- E3 效率指标最高提升：193.8%。
- 在特定实验中，小模型 DS-Qwen-32B 的效率可以达到大模型 DS-LLaMA-70B 的水平。

上述数字均为论文报告的最高结果，不代表所有任务均可获得相同比例的提升。

**研究意义**

这篇论文最重要的贡献是：

> 不应该对整个任务统一分配计算资源，而应该根据不同子任务的复杂度动态分配预算。

**对 Agent Skill 的启发**

应在任务执行前进行复杂度分析，并将复杂任务拆分为多个阶段，为不同阶段设置独立的思考预算。

---

### 2.2 Adaptive Thinking: Large Language Models Know When to Think in Latent Space

**发表信息**

- 作者：Pingzhi Li 等
- 会议：ICLR 2026
- 论文：[ICLR 官方页面](https://proceedings.iclr.cc/paper_files/paper/2026/hash/96dd7b14cde7e0a5194eceedafb2f249-Abstract-Conference.html)

**研究问题**

不同问题需要不同程度的推理。

能否在正式开始推理之前，判断一个问题需要多少额外计算？

**核心方法：Sonata**

作者利用 Self-Consistency（多条推理路径之间的一致性）作为推理需求的代理指标。

主要思路：

1. 观察不同推理路径之间的一致程度。
2. 一致性较低，说明问题可能需要更深入的推理。
3. 训练轻量级 Adapter 预测问题的一致性。
4. 在推理开始之前，根据预测结果动态分配预算。

Adapter 可以利用模型最后一层的隐藏表示，在 Prefill 阶段预测推理需求。

**论文报告的实验结果**

在 Qwen3、GPT-OSS 等模型及 AIME25、GSM8K、MATH500、GPQA、LiveCodeBench 等基准上：

- 减少 20%–60% 的思考 Token，同时保持相同准确率。
- 在相同 Token 成本下，准确率最高提升约 2%。
- Adapter 的额外推理开销低于 0.1%。

**研究意义**

它解决的是：

> 什么时候应该增加思考预算？

而不是简单地要求模型始终进行更长的推理。

**局限性**

Sonata 需要使用校准数据训练 Adapter，并访问模型内部隐藏状态。

因此，它不能直接等同于一个只依赖 Prompt 的通用 Agent Skill。

---

### 2.3 EAGer: Entropy-Aware Generation for Adaptive Inference-Time Scaling

**发表信息**

- 作者：Daniel Scalena、Leonidas Zotos、Elisabetta Fersini、Malvina Nissim、Ahmet Üstün
- 会议：ICML 2026
- 论文：[PMLR 官方页面](https://proceedings.mlr.press/v306/scalena26a.html)

**研究问题**

传统 Best-of-N 方法会为每个问题生成固定数量的候选推理路径。

但不同问题的复杂程度不同，统一分配计算资源并不经济。

**核心方法：EAGer**

通过 Token 级别的熵（Entropy）估计模型的不确定性：

1. 识别模型预测不确定性较高的位置。
2. 在这些位置扩展更多推理路径。
3. 对低不确定性位置避免冗余采样。
4. 将节省的计算资源分配给更需要探索的问题。

该方法不需要额外训练。

**论文报告的实验结果**

在 AIME 2025 等复杂推理基准上：

| 实验场景 | Pass@k 提升 | Token 减少 |
|---|---:|---:|
| 有目标标签的训练相关场景 | 最高 +37% | 59% |
| 测试时推理场景 | 最高 +12% | 64% |

这里的 +37% 和 +12% 是不同实验条件下的结果，不能混为一谈。

**研究意义**

该研究说明：

> 应该将额外计算集中在模型真正存在不确定性的地方，而不是平均增加每个问题的推理次数。

**对 Agent Skill 的启发**

可以将模型置信度、不同方案之间的分歧、证据不足等作为触发深度思考的信号。

但需要注意，普通 Agent Skill 通常无法直接读取模型内部的 Token Entropy，因此需要使用外部可观察的代理指标。

---

### 2.4 When More Thinking Hurts: Overthinking in LLM Test-Time Compute Scaling

**发表信息**

- 作者：Shu Zhou、Rui Ling、Junan Chen、Xin Wang、Tao Fan、Hao Wang
- 会议：Findings of ACL 2026
- 论文：[ACL Anthology](https://aclanthology.org/2026.findings-acl.1199/)

**研究问题**

增加思考长度是否一定能提高推理准确率？

这篇论文专门研究增加推理 Token 的边际收益，以及过度思考问题。

**主要发现**

1. 随着计算预算增加，额外推理 Token 的边际收益会明显下降。
2. 模型可能出现 Overthinking。
3. 延长推理过程可能导致模型放弃之前已经正确的答案。
4. 不同难度的问题具有不同的最优思考长度。
5. 在适当预算下停止思考，可以显著降低计算成本，同时维持相近的准确率。

**研究意义**

这篇论文对自适应推理系统具有重要意义：

> 目标不是让模型尽可能多地思考，而是找到足以解决任务的最优思考预算。

**对 Agent Skill 的启发**

必须建立明确的停止条件，避免无限制地进行自我反思。

---

### 2.5 Think Deep, Not Just Long: Measuring LLM Reasoning Effort via Deep-Thinking Tokens

**发表信息**

- 作者：Wei-Lin Chen、Liqian Peng、Tian Tan、Chao Zhao、Jianhang Chen、Ziqian Lin、Alec Go、Yu Meng
- 会议：ICML 2026
- 论文：[PMLR 官方页面](https://proceedings.mlr.press/v306/chen26di.html)

**研究问题**

思考 Token 数量是否能够代表实际思考深度？

作者认为，单纯统计推理 Token 数量并不能准确衡量模型的推理质量。

例如：

- 模型使用大量 Token 重复同一个观点。
- 模型使用较少 Token 完成关键假设修正。

前者 Token 更多，但不一定具有更深的推理。

**核心方法**

论文提出 Deep-Thinking Tokens 的概念。

它关注模型内部预测在更深网络层中发生显著修正、随后逐渐收敛的 Token。

进一步定义 Deep-Thinking Ratio，用于衡量一段推理中这类 Token 的比例。

**实验结果**

研究使用 AIME 24/25、HMMT 25、GPQA-Diamond 等基准，以及 GPT-OSS、DeepSeek-R1、Qwen3 等模型。

主要发现：

- Deep-Thinking Ratio 与准确率具有稳定的正相关。
- 相较单纯 Token 长度和模型置信度，该指标具有更强的准确率预测能力。
- Think@$n$ 利用该指标筛选推理路径，可以在显著降低推理成本的同时，达到或超过标准 Self-Consistency 的表现。

**研究意义**

> 思考质量比思考数量更重要。

**局限性**

Deep-Thinking Ratio 依赖模型内部信息，普通外部 Agent Skill 无法直接计算。

但可以借鉴其思想，使用外部可观察的有效信息增量、假设修正次数和独立验证结果来估计思考质量。

---

### 2.6 Multi-Agent Reasoning Improves Compute Efficiency: Pareto-Optimal Test-Time Scaling

**发表信息**

- 作者：Florian Valentin Wunderlich、Lars Benedikt Kaesberg、Jan Philip Wahle、Terry Ruas、Bela Gipp
- 会议：ACL 2026 Student Research Workshop
- 论文：[ACL Anthology](https://aclanthology.org/2026.acl-srw.1/)

**研究问题**

在有限计算预算下，哪一种推理扩展策略可以获得更好的准确率？

作者系统比较了：

1. Self-Consistency
2. Self-Refinement
3. Multi-Agent Debate
4. Mixture-of-Agents

实验使用 MMLU-Pro 和 BBH 两个推理基准，比较了 34 种配置和 100 多次评估。

**论文报告的实验结果**

- 在最高计算预算（CoT 的 20 倍）下，MMLU-Pro 准确率最高提升 7.1 个百分点。
- 相同计算预算下，Debate 比 Self-Consistency 高 1.3 个百分点。
- 相同计算预算下，Mixture-of-Agents 比 Self-Consistency 高 2.7 个百分点。
- Self-Consistency 较早出现收益饱和，而多 Agent 方法在复杂任务上仍能持续获得收益。

**研究意义**

增加计算量不一定意味着让同一个模型反复思考。

在某些复杂任务中，增加独立思考视角、让不同 Agent 交叉审查，可能比单纯延长同一条推理链更有效。

**局限性**

这是一篇 ACL Student Research Workshop 论文，证据成熟度应与主会论文区别对待。

---

### 2.7 AERA: Adaptive Evidence Residual Allocation for Efficient Test-Time Reasoning

**发表信息**

- 作者：Ziming Wang、Ivor Tsang、Hangwei Qian
- 时间：2026 年 8 月
- 状态：arXiv 预印本
- 论文：[arXiv:2608.27964](https://arxiv.org/abs/2608.27964)

**研究问题**

当前答案看起来已经足够确定，是否意味着没有必要继续计算？

作者指出，推理过程中的答案正确性可能呈现非单调变化：

- 证据看起来越来越充分，但后续可能出现错误。
- 当前答案看起来不稳定，但继续推理可能恢复正确。

因此，单纯依据当前置信度或答案稳定性决定停止，可能并不可靠。

**核心方法：AERA**

通过分析：

- 答案分布
- 时间序列特征
- 重新求解结果
- 语义特征
- 已消耗计算量

预测继续投入计算是否仍有价值。

系统可以逐步决定：

- 停止当前问题的计算。
- 为当前问题追加计算预算。
- 将节省的计算资源分配给其他问题。

**论文报告的实验结果**

在 300 道未参与训练的 GSM8K 测试问题上：

| 方法 | 准确率 | Completion Tokens |
|---|---:|---:|
| 128 次响应 | 93.01% | 基准 |
| AERA | 92.61% | 减少 95.99% |

AERA 在准确率仅有 0.40 个百分点差距的情况下，大幅减少了计算量。

**研究意义**

它将停止策略从：

> 当前答案是否足够确定？

进一步推进为：

> 继续投入计算，是否还有机会改善答案？

这对设计 Agent 的动态停止机制具有参考价值。

**局限性**

目前仍是预印本，且实验不能直接证明其适用于所有类型的 Agent 任务。

---

### 2.8 The Shadow Price of Reasoning: Economic Perspective on Optimal Budget Allocation for LLMs

**发表信息**

- 作者：Xu Wan、Speed Zhu、Jianwei Cai、Guang Chen、Ximing Huang、Wiggin Zhou、Mingyang Sun
- 会议：ICML 2026
- 论文：[PMLR 官方页面](https://proceedings.mlr.press/v306/wan26p.html)

**研究问题**

当系统同时处理大量问题，而总计算预算有限时，应该如何分配计算资源？

**核心方法：CLEAR**

作者将推理预算分配建模为全局约束优化问题。

核心思想：

1. 估计不同问题继续推理的潜在收益。
2. 计算额外推理资源的边际价值。
3. 将计算资源从收益较低的问题转移到更有可能通过额外推理解决的问题。
4. 在有限预算下优化整体准确率。

**论文报告的实验结果**

在资源紧张的实验场景中，CLEAR 相较统一分配策略，整体准确率最高提高到原来的 3 倍。

这是全局资源分配场景中的实验结果，不代表单个问题的准确率提高三倍。

**研究意义**

它将自适应思考从单个问题的推理策略扩展到了整个 Agent 系统的资源调度。

---

## 3. 论文横向比较

| 论文 | 核心方法 | 动态预算 | 训练要求 | 主要价值 |
|---|---|---|---|---|
| Plan and Budget | 子任务分解、预算调度 | 是 | 无需模型重新训练 | 按子任务分配计算 |
| Adaptive Thinking | 一致性预测、Adapter | 是 | 需要训练 Adapter | 判断何时深入思考 |
| EAGer | Token 熵、自适应分支 | 是 | 无需额外训练 | 按不确定性扩展推理 |
| When More Thinking Hurts | 边际收益、过度思考分析 | 是 | 研究分析 | 避免无效思考 |
| Think Deep, Not Just Long | Deep-Thinking Ratio | 是 | 依赖模型内部信息 | 衡量实际推理深度 |
| Multi-Agent Reasoning | Debate、MoA | 部分 | 无需专门训练 | 增加独立视角 |
| AERA | 剩余计算价值预测 | 是 | 需要训练控制器 | 动态停止与资源追加 |
| Shadow Price of Reasoning | 全局预算优化 | 是 | 需要系统级控制器 | 多任务资源调度 |

注：表中的训练要求指论文提出方法的主要实现要求，不等于这些研究都可以直接通过普通 Prompt 实现。

---

## 4. 综合研究结论

### 4.1 增加推理计算可以提高任务准确率

近一年研究进一步支持了 Test-Time Compute Scaling 的有效性。

但需要区分不同的计算扩展方式：

- 延长单条推理链。
- 生成更多独立候选答案。
- 扩展多条推理路径。
- 增加独立验证与修正。
- 增加多个 Agent 的交叉分析。

不同方法在不同任务上的效果并不相同。

### 4.2 动态分配计算预算优于统一增加计算

这是近一年研究中非常明显的发展趋势。

Plan-and-Budget、Sonata、EAGer 和 CLEAR 都在研究如何根据任务难度或不确定性分配计算资源。

因此，不能简单地给所有任务设置相同的反思轮数。

### 4.3 思考质量比思考数量更重要

When More Thinking Hurts 和 Think Deep, Not Just Long 给出了重要补充：

- 过长的推理可能降低准确率。
- 重复相同观点不等于深入思考。
- 新一轮思考应该产生新的信息或修正。
- 应该根据边际收益决定是否继续。

### 4.4 反思、搜索和验证是不同的计算策略

对于复杂问题：

- Self-Refinement 更适合修正已有答案。
- Self-Consistency 更适合通过多个独立结果降低偶然错误。
- Tree Search 更适合探索不同解题路径。
- Multi-Agent Debate 更适合交叉论证。
- Verification 更适合检查最终结果及其依据。

不应该把所有问题都交给同一种反思机制。

---

## 5. 对通用 AI Agent Skill 的设计建议

### 5.1 设计目标

建议设计一个通用 Skill：

**Adaptive Deliberation（自适应审慎思考）**

它不是要求模型无条件多思考，而是根据任务特点动态调整推理预算。

核心目标：

1. 简单任务快速完成。
2. 复杂任务主动增加思考深度。
3. 高风险任务增加验证次数。
4. 每轮思考都应带来有效信息或错误修正。
5. 当边际收益不足时及时停止。

### 5.2 四层架构

#### Layer 1：Task Complexity Assessment

任务复杂度评估。

输入任务后，判断：

- 任务复杂度。
- 问题不确定性。
- 错误代价。
- 是否存在多个合理答案。
- 是否需要外部证据。
- 是否涉及多个独立子任务。

输出任务等级：

| 等级 | 任务类型 | 建议策略 |
|---|---|---|
| L0 | 简单事实、翻译、格式转换 | 直接回答 |
| L1 | 一般分析、常规代码修改 | 一轮分析与检查 |
| L2 | 复杂技术问题、研究分析 | 多角度分析、反方检查 |
| L3 | 高风险、复杂决策、多步骤任务 | 分解任务、多轮验证、独立审查 |

等级只用于确定初始预算，执行过程中应允许动态升级或降级。

#### Layer 2：Adaptive Compute Allocation

动态计算预算分配。

根据任务等级、子任务复杂度和不确定性决定：

- 初始思考预算。
- 最大反思轮数。
- 是否需要独立推理路径。
- 是否需要独立验证。
- 是否需要调用外部工具。

对于复杂任务，优先进行任务分解，为每个子任务分配不同预算。

#### Layer 3：Critical Reasoning and Verification

批判性推理与验证。

每一轮应当执行有明确目标的分析：

1. 独立构建初始解决方案。
2. 检查关键假设。
3. 寻找反例。
4. 分析潜在失败路径。
5. 验证事实与推理依据。
6. 根据新证据修正答案。

如果某轮没有产生新的证据、有效推理路径或实质性修正，不应仅为了增加轮数而重复思考。

#### Layer 4：Adaptive Stopping

动态停止。

检查：

- 是否已经解决原始问题？
- 关键结论是否得到验证？
- 是否仍然存在重要的不确定性？
- 新一轮分析是否产生了有效信息？
- 是否还有合理的改进空间？
- 新增计算的成本是否超过预期收益？

满足停止条件后输出最终答案。

---

## 6. 边际思考收益模型

可以将推理过程抽象为一个计算资源优化问题。

定义：

- \(Q_n\)：第 \(n\) 轮推理后的答案质量。
- \(C_n\)：截至第 \(n\) 轮的累计计算成本。
- \(MRG_n\)：第 \(n\) 轮新增计算带来的边际收益。

\[
MRG_n =
\frac{Q_n-Q_{n-1}}{C_n-C_{n-1}}
\]

当新增计算带来的质量收益很低，或者质量出现下降时，应考虑停止。

更完整的优化目标可以表示为：

\[
\max_{\pi} \quad
\mathbb{E}[Q(\pi)]-\lambda \mathbb{E}[C(\pi)]
\]

其中：

- \(\pi\)：推理预算分配策略。
- \(Q\)：最终任务质量。
- \(C\)：Token、工具调用、模型调用和时间等计算成本。
- \(\lambda\)：计算成本的权重。

实际工程中，答案质量不能只依靠模型自身的主观置信度评估。

应优先使用：

- 单元测试与集成测试。
- 可验证的数学结果。
- 权威资料和原始证据。
- 结构化规则检查。
- 独立评估器。
- 人工评审或真实任务结果。

对于无法客观衡量的任务，可以结合多个评价维度，并明确剩余不确定性。

---

## 7. 工程实现路线

### 7.1 第一阶段：纯 Prompt Skill

优先实现不依赖模型内部状态的版本。

主要能力：

- 任务复杂度分级。
- 任务分解。
- 动态反思轮数。
- 反方观点检查。
- 事实和逻辑验证。
- 停止条件。

优点：

- 可用于 Codex、Claude Code 等支持 Agent Skills 的工具。
- 不需要训练模型。
- 实现成本较低。
- 可以快速验证实际效果。

### 7.2 第二阶段：Agent Runtime

在 Prompt Skill 的基础上增加运行时控制：

- 控制模型调用次数。
- 控制每次调用的 Token 预算。
- 并行生成多个候选答案。
- 支持独立验证 Agent。
- 根据执行结果追加计算预算。
- 记录每轮计算成本和验证结果。

### 7.3 第三阶段：自适应预算优化

通过实际任务数据建立预算分配模型。

收集：

- 任务类型。
- 初始复杂度评估。
- 推理轮数。
- Token 消耗。
- 工具调用次数。
- 每轮答案质量。
- 最终任务准确率。
- 错误类型。

逐步学习不同任务类型的最优计算预算。

最终实现从人工设定规则到数据驱动预算分配的升级。

---

## 8. Benchmark 与评估方案

建议不要仅使用模型自我评价来验证 Skill 是否有效。

应建立对照实验：

| 实验组 | 策略 |
|---|---|
| A | 普通一次性回答 |
| B | 固定两轮反思 |
| C | 固定三轮反思 |
| D | 动态复杂度分级 |
| E | 动态分级 + 反方验证 |
| F | 动态分级 + 多路径搜索 + 停止机制 |

建议记录以下指标：

| 指标 | 说明 |
|---|---|
| Task Accuracy | 任务准确率 |
| Task Success Rate | 任务完成率 |
| Token Consumption | Token 消耗 |
| Latency | 完成时间 |
| Tool Calls | 工具调用次数 |
| Error Rate | 错误率 |
| Correction Rate | 发现并修正错误的比例 |
| Cost per Success | 每次成功任务的平均成本 |

特别关注：

\[
\text{Cost per Success}
=
\frac{\text{Total Compute Cost}}
{\text{Successful Tasks}}
\]

这比单独比较 Token 数量或准确率更能反映实际工程价值。

实验还应该按照任务难度分组，并控制模型版本、工具权限和外部信息条件，避免将模型能力差异误认为 Skill 的效果。

---

## 9. 推荐阅读顺序

| 优先级 | 论文 | 重点 |
|---|---|---|
| 1 | Plan and Budget | 任务分解与预算分配 |
| 2 | When More Thinking Hurts | 过度思考与停止条件 |
| 3 | Adaptive Thinking | 推理需求预测 |
| 4 | EAGer | 不确定性驱动的计算扩展 |
| 5 | Think Deep, Not Just Long | 思考质量衡量 |
| 6 | AERA | 动态停止与剩余计算价值 |
| 7 | Multi-Agent Reasoning | 多 Agent 交叉验证 |
| 8 | Shadow Price of Reasoning | 全局预算优化 |

---

## 10. 最终结论

近一年相关研究的核心趋势，是从固定推理长度和统一计算预算，逐渐转向自适应测试时计算分配。

可以归纳为五项原则：

1. **按需思考**：根据任务复杂度决定初始推理预算。
2. **按需扩展**：将额外计算集中在困难和不确定的环节。
3. **多角度验证**：在必要时使用独立推理路径和不同 Agent 交叉审查。
4. **重视思考质量**：每轮推理都应该产生新的信息或实质性修正。
5. **及时停止**：当新增计算的边际收益不足时，结束推理。

因此，最值得工程化实现的方向不是简单的 `think-more`，而是：

**Adaptive Deliberation — 自适应审慎思考引擎。**

其本质是将模型的推理过程视为一个可以动态调度的计算过程，在任务准确率、推理深度、响应时间和计算成本之间寻找最优平衡。

需要强调：上述论文的实验主要集中在数学、科学和知识推理基准，不能直接证明该架构一定能提高所有编程、研究、写作和通用 Agent 任务的准确率。实际效果仍需要针对目标任务进行 Benchmark 验证。

---

## 参考文献

1. Lin, J., et al. (2026). *Plan and Budget: Effective and Efficient Test-Time Scaling on Reasoning Large Language Models*. ICLR 2026. https://proceedings.iclr.cc/paper_files/paper/2026/hash/ae8d4084f418bb51575c2ca6c658a05b-Abstract-Conference.html

2. Li, P., et al. (2026). *Adaptive Thinking: Large Language Models Know When to Think in Latent Space*. ICLR 2026. https://proceedings.iclr.cc/paper_files/paper/2026/hash/96dd7b14cde7e0a5194eceedafb2f249-Abstract-Conference.html

3. Scalena, D., et al. (2026). *EAGer: Entropy-Aware GEneRation for Adaptive Inference-Time Scaling*. ICML 2026. https://proceedings.mlr.press/v306/scalena26a.html

4. Zhou, S., et al. (2026). *When More Thinking Hurts: Overthinking in LLM Test-Time Compute Scaling*. Findings of ACL 2026. https://aclanthology.org/2026.findings-acl.1199/

5. Chen, W.-L., et al. (2026). *Think Deep, Not Just Long: Measuring LLM Reasoning Effort via Deep-Thinking Tokens*. ICML 2026. https://proceedings.mlr.press/v306/chen26di.html

6. Wunderlich, F. V., et al. (2026). *Multi-Agent Reasoning Improves Compute Efficiency: Pareto-Optimal Test-Time Scaling*. ACL 2026 Student Research Workshop. https://aclanthology.org/2026.acl-srw.1/

7. Wang, Z., Tsang, I., & Qian, H. (2026). *AERA: Adaptive Evidence Residual Allocation for Efficient Test-Time Reasoning*. arXiv:2608.27964. https://arxiv.org/abs/2608.27964

8. Wan, X., et al. (2026). *The Shadow Price of Reasoning: Economic Perspective on Optimal Budget Allocation for LLMs*. ICML 2026. https://proceedings.mlr.press/v306/wan26p.html

---

*文档说明：本报告为基于论文官方摘要、出版信息及公开研究结果整理的技术调研，论文报告的实验结果均保留其原始实验场景，不应直接视为通用任务的效果保证。*