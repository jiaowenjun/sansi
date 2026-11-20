# AI Agent 自适应深度思考 Skill 调研报告

> **研究主题**：通过结构化思考、多轮反思、任务分解和验证机制，以额外计算成本换取更高的任务准确率  
> **研究范围**：GitHub 开源 Agent Skills  
> **调研日期**：2026-11-20  
> **筛选数量**：3 个代表性 Skill  
> **目标平台**：Codex、Claude Code、Cursor、GitHub Copilot 等支持 Agent Skills 的工具  
> **核心目标**：为开发通用的 Adaptive Deliberation（自适应审慎思考）Skill 提供工程参考<br>
> **源码复核版**：本地仓库快照、测试结果与证据边界见[本地资料汇总](local_reference_summary.md)

---

# 1. 调研目标与筛选方法

## 1.1 目标

希望找到一种能够贯彻「三思而后行」原则的通用 Agent Skill。

它不只是要求模型认真思考，而是能够：

1. 根据任务复杂度决定思考深度。
2. 对复杂任务投入更多推理次数和计算资源。
3. 通过多角度分析减少单一思维路径的盲点。
4. 对初始答案进行批判、修正和重新验证。
5. 根据每轮思考的实际收益决定是否继续。
6. 在准确率、Token 消耗和响应时间之间取得平衡。

## 1.2 筛选标准

主要考察以下五个维度：

| 维度 | 权重 | 评价内容 |
|---|---:|---|
| 目标相关性 | 35% | 是否直接实现深度思考、反思或验证 |
| 动态思考能力 | 25% | 是否能根据任务难度调整思考预算 |
| 实现成熟度 | 15% | 是否有完整 SKILL.md、脚本、Hooks、测试 |
| 通用性 | 10% | 是否适用于不同 Agent 和非编程任务 |
| GitHub Stars | 15% | 社区关注度和实际采用情况 |

Stars 仅作为社区关注度的参考，不代表实际准确率或工程质量。

本报告优先选择与研究目标直接相关的项目，而不是单纯选择 Stars 最高的通用 Skill 集合。

## 1.3 最终入选项目

截至 2026-11-20 检索到的 GitHub 页面数据：

| 排名 | 项目 | GitHub Stars | 主要方向 |
|---:|---|---:|---|
| 1 | unlazy | 约 4.4k | 深度任务分解、完成验证 |
| 2 | cc-thinking-skills | 约 1.8k | 思维模型路由、组合分析 |
| 3 | Self-Refine Skill | 约 12 | 多轮反思、自我修正 |

项目地址：

- [Leonxlnx/unlazy](https://github.com/Leonxlnx/unlazy)
- [tjboudreaux/cc-thinking-skills](https://github.com/tjboudreaux/cc-thinking-skills)
- [Thomaszhou22/self-refine-skill](https://github.com/Thomaszhou22/self-refine-skill)

选择第三个项目时，虽然其 Stars 明显较少，但它与「通过增加思考轮数提高答案质量」这一目标高度吻合，因此在相关性优先的筛选策略下仍然入选。

本次源码结论固定到 `unlazy@1667149`、`cc-thinking-skills@7b8fece` 和 `self-refine-skill@620cb9b`。Stars 是调研日页面快照，不参与源码能力或效果证据判断。

---

# 2. Skill 一：unlazy

## 2.1 项目概况

**GitHub：** https://github.com/Leonxlnx/unlazy

**Stars：** 约 4.4k

**License：** MIT

**主要定位：** Completion Discipline for AI Agents

项目的核心理念：

> Make incomplete work visible and make completion testable.

即：让未完成的工作显性化，让任务完成状态能够被验证。

它主要针对 AI Agent 在长时间、多步骤任务中出现的问题：

- 任务尚未真正完成就提前结束。
- 只实现主要功能，遗漏边缘需求。
- 没有充分检查自己的实现。
- 多步骤任务中丢失原始要求。
- 在复杂项目中出现局部正确、整体错误。
- Agent 声称任务完成，但缺乏客观证据。

它不是传统意义上的思维模型集合，而是将深度任务分解、执行约束和完成验证结合起来的工程化 Skill。

## 2.2 核心实现逻辑

unlazy 最重要的设计是 Depth Tree（深度任务树）。

### 2.2.1 Depth Tree：递归分解任务

传统 Agent 通常采用线性执行：

```text
接收任务
   |
分析任务
   |
执行任务
   |
检查结果
   |
结束
```

unlazy 则要求复杂任务先进行结构化分解：

```text
                    Root Task
                        |
          +-------------+-------------+
          |                           |
       Branch A                    Branch B
          |                           |
     +----+----+                 +----+----+
     |         |                 |         |
   Leaf A1   Leaf A2           Leaf B1   Leaf B2
     |         |                 |         |
   验证      验证                验证      验证
     |         |                 |         |
     +---------+-----------------+---------+
                        |
                  集成验证
                        |
                   最终验收
```

具体规则：

1. 首先保留用户的原始任务作为 Root。
2. 按照自然的业务边界、模块边界和验证边界拆分。
3. 每个 Leaf 必须是一个完整、独立、可交付的工作单元。
4. 每个 Leaf 有明确的输入、输出、依赖关系和验收条件。
5. Branch 负责检查子任务之间的集成。
6. Root 最后重新核对所有原始要求。

一个重要细节：

Depth Tree 的深度不是简单的二叉树深度，也不意味着 Token 数量一定按照 \(2^N\) 增长。

项目目前明确要求：只有在真实工作边界存在时才继续拆分，不能为了达到指定深度而制造无意义的子任务。

### 2.2.2 Acceptance Gates：验收门槛

这是 unlazy 最有工程价值的设计。

在真正开始执行之前，先创建 `GATES.md`，定义任务完成必须满足的条件。

例如：

```markdown
# Gates: User Authentication

- [ ] G1: Login API works
  CHECK: pnpm test login.test.ts
  EXPECT: login tests passed

- [ ] G2: Invalid password is rejected
  CHECK: pnpm test login-security.test.ts
  EXPECT: security tests passed

- [ ] G3: Refresh token works
  CHECK: pnpm test refresh.test.ts
  EXPECT: refresh tests passed

- [ ] G4: No TypeScript errors
  CHECK: pnpm typecheck
  EXPECT: TypeScript compilation passed
```

每个 Gate 包含：

| 字段 | 作用 |
|---|---|
| Gate ID | 唯一标识 |
| Description | 可观察的完成条件 |
| CHECK | 验证命令 |
| EXPECT | 预期成功结果 |
| EVIDENCE | 验证证据 |

核心逻辑：

```text
定义验收条件
     |
执行任务
     |
运行 CHECK
     |
检查退出码和 EXPECT
     |
记录验证证据
     |
全部通过？
   /     \
  否      是
  |        |
继续修正  进入下一阶段
```

与普通 Prompt 最大的区别是：

普通 Prompt 依赖模型自己判断「我已经完成了」。

unlazy 则将完成条件外部化，要求 Agent 提供能够检查的证据。

### 2.2.3 四轮工作流程

unlazy 要求每个 Leaf 至少经过以下四个工作阶段：

| 阶段 | 操作 | 目标 |
|---|---|---|
| Pass 1 | Implement | 完整实现，不留占位内容 |
| Pass 2 | Expert Reread | 重新以领域专家视角审查 |
| Pass 3 | Defect Hunt | 检查正确性、集成、性能、兼容性和证据 |
| Pass 4 | Polish | 完成低成本优化，再检查是否仍有问题 |

如果最后一轮发现实质性问题，就继续修正。

如果完整改进检查没有发现问题，且所有 Gate 都有有效证据，才允许完成。

这相当于把「反复检查」变成一个有明确目标的工作流程，而不是让模型漫无目的地重复思考。

### 2.2.4 多 Agent 并行执行

对于大型任务，unlazy 支持将 Depth Tree 中的 Leaf 分配给独立 Agent。

主要机制：

- 每个 Leaf 有明确文件所有权。
- 每个 Leaf 声明依赖关系。
- 只有依赖满足的 Leaf 才能启动。
- 使用 Dispatch Wave 分批启动 Agent。
- 记录 Agent 的执行状态。
- 子 Agent 完成后，由父 Agent 重新验证。
- Branch 最后执行集成验证。

验证分为四层：

1. Leaf 自身检查。
2. Parent 对 Leaf 的重新验证。
3. Branch 集成检查。
4. 可选的 Stop Hook 结构性检查。

这解决了多 Agent 场景下局部正确但整体不一致的问题。

## 2.3 工程实现

unlazy 不仅包含 SKILL.md，还包含完整的运行时辅助工具。

核心目录：

```text
unlazy/
├── SKILL.md
├── references/
│   ├── method.md
│   ├── gates.md
│   ├── orchestration.md
│   ├── dispatch.md
│   ├── parallel.md
│   └── token-economy.md
├── templates/
│   ├── PLAN.md
│   ├── gates-leaf.md
│   └── gates-node.md
├── scripts/
│   ├── gate-check.mjs
│   ├── gate-lint.mjs
│   ├── install-hooks.mjs
│   └── stop-hook.mjs
└── tests/
```

本地执行 `npm test` 时，主要测试套件 34/34、27/27、51/51、24/24、29/29、8/8 均通过，自检 15/15 通过。这证明 Gate 解析、授权与证据绑定、租约调度和 Stop Hook 符合仓库定义的软件契约；它不等于“模型思考能力”或通用任务准确率已经提升。

其中最值得研究的组件：

**1. gate-check.mjs**

负责：

- 解析 GATES.md。
- 检查 Gate 格式。
- 运行已经获得授权的验证命令。
- 检查命令退出码。
- 检查 EXPECT。
- 记录验证证据。
- 检查之前的证据是否仍然有效。

它不会无条件执行任意命令。需要先检查命令内容，并通过明确授权才能执行。

**2. gate-lint.mjs**

检查验收条件是否存在明显问题，例如：

- Gate 没有明确的验证命令。
- EXPECT 无法有效区分成功和失败。
- 验收条件过于模糊。
- 命令无法真正验证对应目标。

**3. stop-hook.mjs**

在 Claude Code 中阻止 Agent 在必要验收条件尚未完成时提前结束。

Hook 不负责重新执行验证，而是根据结构化状态判断是否允许结束。

项目还提供进度保护，避免在无法取得进展时无限阻止退出。

**4. PLAN.md**

大型任务的统一计划文件，记录：

- 任务树。
- 子任务所有权。
- 依赖关系。
- 执行波次。
- 当前状态。
- 推理等级。
- 完成证据。

## 2.4 优点与局限

**优点**

1. 将任务完整性从主观判断转变为客观验收。
2. 能够处理复杂、多步骤和长时间任务。
3. 有明确的任务拆分与集成机制。
4. 支持多 Agent 并行。
5. 验证流程具有较高工程成熟度。
6. 能够减少遗漏和提前结束。

**局限**

1. 主要面向编程和工程执行，不是通用知识推理 Skill。
2. 对简单问题使用成本过高。
3. 依赖 Agent 能够创建文件、执行命令和读取验证结果。
4. Depth Tree 并不直接控制模型内部推理 Token。
5. Gate 只能证明其验证命令所测量的内容，不能保证验收条件本身完整或正确。
6. 过度分解会产生额外协调成本。

特别注意：项目明确说明，早期版本关于任务深度可以按二叉树指数倍增加努力的说法，缺少可复现的原始实验材料，不能将其视为已经验证的计算扩展规律。

## 2.5 如何借鉴并落地

我认为不应该完整照搬 unlazy，而应该提取三个核心机制。

**机制 A：复杂任务分解**

对于复杂任务，自动创建任务树。

但应增加一个约束：

只有当子任务具有独立目标、不同知识领域或独立验证方式时才进行拆分。

例如：

```text
研究一个数据库选型问题
       |
       +-- 性能需求分析
       |
       +-- 数据一致性分析
       |
       +-- 运维成本分析
       |
       +-- 扩展能力分析
       |
       +-- 综合决策
```

**机制 B：任务完成检查清单**

将用户需求转成结构化验收条件。

非编程任务也可以使用：

```markdown
# Research Gates

- [ ] 是否覆盖所有核心问题
- [ ] 是否区分事实与推测
- [ ] 是否检查关键数据来源
- [ ] 是否考虑反方证据
- [ ] 是否存在逻辑矛盾
- [ ] 是否给出可执行结论
```

对于研究类任务，可以让 Gate 对应每一项待验证的结论。

**机制 C：分层验证**

将自我检查、独立检查和最终验收分开。

例如：

```text
初始答案
   |
自身检查
   |
修正答案
   |
独立视角审查
   |
验证事实与证据
   |
最终验收
```

推荐将 unlazy 作为复杂任务执行层的参考，而不是将它作为所有任务的默认思考机制。

---

# 3. Skill 二：cc-thinking-skills

## 3.1 项目概况

**GitHub：** https://github.com/tjboudreaux/cc-thinking-skills

**Stars：** 约 1.8k

**License：** MIT

**主要定位：** 结构化思考模型集合。

这是一个包含 28 个独立 Agent Skills 的思维框架集合，支持：

- Claude Code
- GitHub Copilot
- Codex
- Cursor
- 其他兼容 Agent Skills 的工具

与 unlazy 不同，它不主要关注任务执行是否完整，而是关注：

**面对复杂问题时，应该采用什么思考方法，以及如何组合不同的思考方法。**

## 3.2 核心实现逻辑

### 3.2.1 28 个 active Skill

项目将不同领域的思维方法封装为独立 Skill。

主要分为五类。

**A. 思维模型路由与组合**

| Skill | 作用 |
|---|---|
| thinking-model-router | 根据问题选择合适的思维模型 |
| thinking-model-combination | 组合多个互补的思维模型 |

**B. 问题诊断与理解**

| Skill | 作用 |
|---|---|
| thinking-scientific-method | 假设建立与科学验证 |
| thinking-five-whys-plus | 连续追问根本原因 |
| thinking-kepner-tregoe | 对比问题的发生与未发生情况 |
| thinking-systems | 系统结构、反馈与因果关系 |
| thinking-map-territory | 识别认知模型与实际情况之间的差异 |
| thinking-cynefin | 根据问题性质选择解决方式 |
| thinking-socratic | 通过提问发现隐藏假设 |

**C. 决策与评估**

| Skill | 作用 |
|---|---|
| thinking-reversibility | 分析决策可逆性 |
| thinking-opportunity-cost | 分析机会成本 |
| thinking-probabilistic | 概率推理与置信度更新 |
| thinking-second-order | 分析二阶影响 |
| thinking-steel-manning | 构建最强反方论证 |
| thinking-bounded-rationality | 设置合理停止条件 |
| thinking-circle-of-competence | 识别证据和认知能力边界 |
| thinking-lindy-effect | 判断事物的长期持续性 |

**D. 创造与优化**

| Skill | 作用 |
|---|---|
| thinking-first-principles | 第一性原理 |
| thinking-triz | 解决矛盾和冲突 |
| thinking-via-negativa | 通过删除不必要因素改善系统 |
| thinking-thought-experiment | 思想实验 |
| thinking-jobs-to-be-done | 识别用户真正需求 |
| thinking-effectuation | 在不确定条件下利用现有资源行动 |

**E. 风险与执行**

| Skill | 作用 |
|---|---|
| thinking-pre-mortem | 事前分析失败原因 |
| thinking-red-team | 对抗性安全分析 |
| thinking-margin-of-safety | 安全边际 |
| thinking-theory-of-constraints | 识别系统瓶颈 |
| thinking-ooda | 观察、判断、决策、行动 |

### 3.2.2 Model Router：思维模型路由

这是整个项目最重要的设计。

如果有 28 个思维模型，却让 Agent 每次都执行全部 28 个模型，不仅成本巨大，而且容易产生重复分析。

因此项目提供 `thinking-model-router`。

其主要流程：

```text
接收问题
   |
判断领域
   |
判断问题类型
   |
分析时间、信息、风险和复杂度
   |
筛选候选思维模型
   |
计算适配程度
   |
选择最合适的模型
   |
是否需要多个模型？
   |
输出 NONE / ONE / MULTI
```

其中：

- NONE：不需要额外思维框架。
- ONE：选择一个主要思维模型。
- MULTI：选择最多三个互补模型。

Router 有几个非常重要的约束：

1. 优先使用一个思维模型。
2. 不因为任务复杂就无条件增加模型数量。
3. 多个模型必须具有不同的分析职责。
4. 如果模型不能提供新的视角，就不应该组合。
5. 组合多个模型之前，必须明确它们之间的冲突处理规则。

例如，分析一个复杂技术架构：

```text
问题：是否应该将单体应用拆分为微服务？

        |
        v

Router
        |
        +-- thinking-reversibility
        |      分析迁移决策的可逆性
        |
        +-- thinking-systems
        |      分析系统复杂度和依赖关系
        |
        +-- thinking-pre-mortem
               分析迁移失败路径
        |
        v

综合分析与决策
```

### 3.2.3 Model Combination：思维模型组合

项目提供独立的 `thinking-model-combination`。

它不是简单地将多个模型的结论放在一起，而是要求：

1. 明确当前问题。
2. 找出主要思维模型尚未解决的盲点。
3. 为第二个、第三个模型指定不同的职责。
4. 预先确定模型之间的关系。
5. 分别执行各自的分析。
6. 最后进行一次综合。

例如：

```text
架构选型
   |
   +-- 第一性原理
   |      分析最基本的技术约束
   |
   +-- 系统思维
   |      分析长期结构性影响
   |
   +-- 反方论证
          寻找当前方案的最强反例
   |
   v
综合决策
```

项目特别强调，不应该为了显得全面而组合思维模型。

每一个模型都必须回答一个独立的问题。

## 3.3 关键实现细节

### 3.3.1 Skill 结构

每个思维模型都有独立的 SKILL.md。

典型结构：

```text
skills/
├── thinking-model-router/
│   └── SKILL.md
├── thinking-model-combination/
│   └── SKILL.md
├── thinking-scientific-method/
│   └── SKILL.md
├── thinking-steel-manning/
│   └── SKILL.md
├── thinking-pre-mortem/
│   └── SKILL.md
└── ...
```

每个 Skill 负责：

- 触发条件。
- 不适用条件。
- 具体分析步骤。
- 结构化输出。
- 停止条件。
- 反例检查。

### 3.3.2 Router 的评分机制

项目的 Router 对多个候选模型进行评分。

评分维度：

| 维度 | 权重 |
|---|---:|
| Problem Fit | 30% |
| Available Inputs | 20% |
| Time to Apply | 15% |
| Stakeholder Usability | 15% |
| Competence to Apply Correctly | 20% |

如果只有一个模型明显适合，就直接选择它。

如果多个模型评分接近，则优先选择输入要求更少、执行更简单的模型。

如果所有模型都不适合，就返回 NONE。

### 3.3.3 Evidence：项目的效果评估

这个项目值得肯定的一点是，它没有直接声称 28 个思维模型一定能提高准确率。

项目提供：

```text
evals/
analysis/
experiments/
scripts/
```

包括：

- Skill 结构检查。
- Router 路由评估。
- 结果评估。
- 证据登记。
- 实验报告。

本地 `validate-skills.js` 显示 28/28 个 active Skill 结构通过，structural eval 通过。这里的 28 个条目包含 router 和 combination，不能全部称为相互独立的思维模型。

当前证据审计指出：

- 28 个 active Skill 全部处于 `manual-only quarantine`。
- 自动确认提升的 Skill 数量为 0。
- `thinking-scientific-method` 有一个初步 +4.0 个百分点的实验结果。
- 该结果低于项目预设的 +5 个百分点效用门槛，且存在证据缺口。

因此不能将它当作已证明能够提高准确率的通用框架。

## 3.4 优点与局限

**优点**

1. 思维模型种类丰富。
2. 具有明确的模型路由机制。
3. 支持多个模型组合。
4. 有明确的适用条件和不适用条件。
5. 具备结构化评估框架。
6. 跨平台支持较好。
7. 可以按需加载，不必每次使用所有模型。

**局限**

1. 主要解决分析方法选择，不直接管理 Token Budget。
2. 大部分 Skill 需要显式触发。
3. 不负责对最终答案进行全面事实验证。
4. 思维模型可能被错误应用。
5. 多模型分析仍然可能出现共同偏见。
6. 缺少充分的独立实验来证明整体准确率提升。

## 3.5 如何借鉴并落地

建议将其 Router 设计直接借鉴到 Adaptive Deliberation。

但是需要将原本的「思维模型路由」升级为「思考策略 + 计算预算路由」。

例如：

```text
                Task Assessment
                       |
          +------------+------------+
          |            |            |
       Diagnose      Decide       Create
          |            |            |
       Scientific   Probabilistic First Principles
       Method       Thinking       Thinking
          |            |            |
          +------------+------------+
                       |
                Complexity Score
                       |
                 Compute Budget
                       |
                 Reasoning Loop
```

我建议采用以下设计：

**第一步：判断任务类型**

- 事实查询
- 问题诊断
- 技术决策
- 研究分析
- 创意设计
- 风险评估
- 多步骤执行

**第二步：选择思维框架**

每个任务只选择最适合的 1–3 个模型。

**第三步：确定分析顺序**

例如：

```text
Scientific Method
       |
       v
Steel Manning
       |
       v
Probabilistic Thinking
       |
       v
Final Decision
```

**第四步：记录不同模型的独立贡献**

不只是要求模型给出多个答案，而是要求每个模型明确回答自己的独立问题。

**第五步：综合并停止**

如果新模型没有产生新证据、新结论或新风险，就停止增加分析框架。

---

# 4. Skill 三：Self-Refine Skill

## 4.1 项目概况

**GitHub：** https://github.com/Thomaszhou22/self-refine-skill

**Stars：** 约 12

**License：** MIT

**主要定位：** Systematic Self-Reflection for AI Agents

这是三个项目中，与「增加思考次数以改善最终答案」最直接相关的项目。

它的核心理念：

> Make every AI response better — before the user sees it.

即：在向用户交付之前，对模型生成的答案进行自我检查和改进。

它将整个过程抽象为：

```text
GENERATE
    |
    v
CRITIQUE
    |
    v
REFINE
    |
    v
CHECK
    |
    +---- 有实质问题？---- YES
    |                       |
    |                       v
    |                    CRITIQUE
    |
    NO
    |
    v
DELIVER
```

## 4.2 核心实现逻辑

### 4.2.1 四阶段反思循环

**Stage 1：Generate**

模型先按照正常方式完成任务，生成初始答案。

这里不要求模型一开始就进行无限制的深度分析。

**Stage 2：Critique**

模型从多个维度批判自己的初始答案。

Skill 定义了六个检查维度：

| 维度 | 检查内容 |
|---|---|
| Logical Completeness | 推理链是否完整 |
| Factual Accuracy | 事实是否准确 |
| Response Completeness | 是否回答全部问题 |
| Conciseness | 是否存在冗余 |
| Actionability | 是否具有可执行性 |
| Internal Consistency | 是否存在内部矛盾 |

其中事实准确性尤其重要。

例如：

```text
初始答案：
Redis 的延迟一定低于 PostgreSQL。

Critique：
该结论是否有适用条件？
是否考虑本地缓存、网络延迟、数据量？
是否存在反例？

Refine：
Redis 通常适合低延迟缓存场景，
但不能脱离部署、查询模式和数据规模
直接断言所有情况下 Redis 更快。
```

**Stage 3：Refine**

根据 Critique 发现的问题修改答案。

这里有一个重要要求：

不能只对原答案进行文字润色，而应该修正真正影响正确性和可用性的问题。

例如：

- 修正事实错误。
- 补充遗漏条件。
- 删除没有证据的结论。
- 修复推理漏洞。
- 改善执行步骤。
- 消除内部矛盾。

**Stage 4：Check**

重新检查修正后的答案。

如果仍然存在重要问题，且预算允许，则再次进入 Critique。

如果没有实质性问题，或者达到最大轮数，则输出答案。

### 4.2.2 动态反思深度

这是 Self-Refine Skill 最值得借鉴的机制。

项目定义四个反思等级：

| Level | 适用任务 | 最大反思轮数 | Token 开销 |
|---|---|---:|---:|
| 0 | 简单问答 | 0 | 0% |
| 1 | 普通任务 | 1 | 约 15% |
| 2 | 复杂技术任务 | 2 | 约 30% |
| 3 | 高风险任务 | 3 | 约 50% |

这些 Token 开销是项目给出的估算，不是跨模型的保证值。

具体逻辑：

**Level 0：Quick Scan**

适用于：

- 问候。
- 简单事实查询。
- 简单格式转换。
- 单句回答。

不执行正式反思。

**Level 1：Standard Review**

适用于：

- 普通解释。
- 常规技术建议。
- 多段落回答。
- 简单代码示例。

只进行一轮快速检查。

**Level 2：Deep Audit**

适用于：

- 复杂技术架构。
- 多步骤计划。
- 研究报告。
- 包含多个关键事实和结论的回答。

逐项检查六个维度，记录问题严重程度：

```text
Critical
Minor
Pass
```

发现 Critical 问题必须修正。

**Level 3：Adversarial Review**

适用于：

- 高风险决策。
- 生产环境代码。
- 安全设计。
- 重要研究结论。
- 用户明确要求严格检查的任务。

在普通检查之外，增加对抗性审查：

> If I wanted to prove this response wrong, what would I attack?

即：如果我要证明这个答案是错误的，最应该攻击哪些地方？

然后检查这些攻击点，修正有效问题。

### 4.2.3 收敛和停止机制

Self-Refine 没有采用无限反思。

它明确规定：

1. 最多三轮反思。
2. 某轮没有发现 Critical 或 Minor 问题时提前停止。
3. 修改仅涉及文字风格时提前停止。
4. 新一轮修正的问题数量少于上一轮时，可以停止。
5. 不允许为了证明自己认真而制造新的修改。

这是非常重要的设计。

因为多轮反思并不一定持续改善答案，模型可能在后续轮次中引入新错误。

### 4.2.4 Reflexion Memory：跨任务学习

Self-Refine 还引入了跨会话反思记忆。

例如：

```text
Self-reflection:

When writing TypeScript examples,
I tend to forget browser/Node compatibility.

Fix:
Prefer ReturnType<typeof setTimeout>
instead of NodeJS.Timeout when examples
should work in both environments.
```

当模型发现重复出现的错误模式时，将其记录到记忆文件中。

后续任务可以读取这些经验，避免重复犯错。

这使 Skill 不再只是一次性答案修正，而是形成持续改进机制。

## 4.3 工程实现结构

项目的核心实现比较轻量：

```text
self-refine-skill/
├── SKILL.md
├── README.md
├── BENCHMARK.md
├── CHANGELOG.md
├── LICENSE
└── references/
    ├── sources.md
    └── reflection-templates.md
```

核心逻辑主要由 SKILL.md 中的指令完成。

它不依赖额外 API，也不要求额外部署一个模型。

宿主 Agent 自己承担：

- 初始答案生成。
- 自我批判。
- 答案修正。
- 最终检查。

需要注意：

Skill 声称 Zero API Cost，准确理解应当是没有额外的外部 API 或独立服务依赖。多轮模型推理仍然会产生额外 Token 消耗、延迟或平台计费。

## 4.4 项目提供的 Benchmark

项目提供了自己的测试报告：

[Self-Refine BENCHMARK.md](https://github.com/Thomaszhou22/self-refine-skill/blob/main/BENCHMARK.md)

实验设置：

- 20 道合成测试题。
- 5 个任务类别。
- 使用同一模型比较单次生成与 Self-Refine。
- 六个评价维度。
- 使用 1–10 分的加权评分。
- 单次运行，没有多随机种子复现。

论文报告：

| 指标 | 单次生成 | Self-Refine |
|---|---:|---:|
| 平均加权分 | 6.42 | 7.78 |
| 中位数 | 6.45 | 7.80 |
| 最低分 | 4.50 | 5.90 |
| 最高分 | 8.30 | 9.60 |

项目报告的总体质量评分提升约 21.2%。

各维度：

| 评价维度 | 提升 |
|---|---:|
| 逻辑完整性 | +23.7% |
| 事实准确性 | +14.8% |
| 回答完整性 | +24.8% |
| 简洁性 | +27.7% |
| 可执行性 | +25.0% |
| 内部一致性 | +11.9% |

但必须注意实验局限：

1. 只有 20 道合成题。
2. 测试集主要是开发者常见任务。
3. 生成与评价使用同一模型。
4. 评分具有一定主观性。
5. 没有进行多随机种子重复实验。
6. 没有测量实际响应延迟。
7. 没有通过消融实验单独评估六个反思维度的贡献。

因此，+21.2% 只能视为项目自身小规模评测结果，不能当作经过独立验证的通用准确率提升。

## 4.5 优点与局限

**优点**

1. 与目标最直接吻合。
2. 有明确的动态反思等级。
3. 反思过程简单、易于理解。
4. 可以用于非编程任务。
5. 不需要额外服务或 API。
6. 有明确的停止机制。
7. 可以通过记忆机制积累错误经验。

**局限**

1. 主要依赖模型自我评价。
2. 自我批判可能与初始推理共享同样的错误。
3. 不能保证发现事实性错误。
4. 多轮反思可能导致答案退化。
5. 不能真正控制底层模型的推理计算预算。
6. 反思等级和 Token 开销是预设估算，不是动态测量的实际计算量。
7. 项目社区规模较小。

## 4.6 如何借鉴并落地

我建议将 Self-Refine 作为整个 Adaptive Deliberation 的核心反思引擎。

但是需要进行三个重要改造。

**改造 A：将固定等级升级为动态预算**

不只是依据任务类型设置 Level 0–3，还要在每轮反思之后重新评估是否需要增加预算。

例如：

```text
Task Assessment
      |
      v
Initial Draft
      |
      v
Critique
      |
      v
发现关键问题？
   /       \
  YES       NO
   |         |
   v         v
追加预算    停止
   |
   v
Refine
   |
   v
再次检查
```

**改造 B：将自我评价与客观验证分开**

例如，代码任务：

- 自我检查：分析逻辑是否正确。
- 客观检查：执行测试。
- 独立检查：让另一个 Agent 审查关键实现。

研究任务：

- 自我检查：检查论证是否完整。
- 外部验证：检查论文、数据和引用。
- 对抗性检查：寻找反方证据。

**改造 C：引入有效信息增量**

每轮 Critique 必须明确指出：

- 新发现了什么问题？
- 是否产生了新的证据？
- 是否修正了原有假设？
- 是否发现了反例？
- 是否真正改善了最终答案？

如果没有产生任何实质性变化，就应该停止。

---

# 5. 三个 Skill 的综合比较

## 5.1 核心能力对比

| 能力 | unlazy | cc-thinking | Self-Refine |
|---|---|---|---|
| 任务复杂度判断 | 间接 | 强 | 强 |
| 思考预算分配 | 间接 | 弱 | 有等级 |
| 多轮自我反思 | 有改进 Pass | 视模型而定 | 核心机制 |
| 多角度分析 | 通过任务分解 | 核心机制 | 通过反方检查 |
| 独立推理路径 | 支持多 Agent | 支持模型组合 | 不提供 |
| 客观验证 | 强 | 弱 | 主要依赖工具能力 |
| 动态停止 | Gate 驱动 | 模型路由停止 | 反思收敛 |
| 跨任务记忆 | 任务状态 | 无核心机制 | 支持反思记忆 |
| 非编程任务 | 有限 | 强 | 强 |
| 工程化程度 | 高 | 中高 | 轻量 |

## 5.2 三个项目解决的是不同问题

可以将它们理解为三个互补层次。

### unlazy：确保任务真正完成

解决：

- 是否遗漏需求？
- 是否完成所有子任务？
- 是否经过实际验证？
- 是否可以证明任务已经完成？

### cc-thinking-skills：确保采用正确的思考方法

解决：

- 应该从什么角度分析？
- 是否需要多个思维模型？
- 当前分析是否存在盲点？
- 是否需要反方论证？

### Self-Refine：确保最终答案经过充分修正

解决：

- 初始答案是否存在错误？
- 是否有遗漏？
- 是否需要进一步反思？
- 是否已经达到收敛状态？

因此，三者不应该被视为竞争关系。

它们实际上可以组成一个完整的自适应推理系统。

---

# 6. 综合设计：Adaptive Deliberation Skill

结合三个项目，我建议设计一个独立的通用 Skill：

**Adaptive Deliberation**

其目标：

> 根据任务复杂度、风险和不确定性，动态分配思考预算，通过结构化推理、反思、独立验证和停止机制，提高最终任务质量。

## 6.1 整体架构

```text
                   User Request
                        |
                        v
              1. Task Assessment
                        |
          +-------------+-------------+
          |             |             |
       Simple         Medium         Complex
          |             |             |
          v             v             v
        Direct       One-pass      Deep Analysis
        Answer       + Review      + Verification
                                      |
                                      v
                           2. Thinking Router
                                      |
                            Select 1-3 Models
                                      |
                                      v
                            3. Task Decomposition
                                      |
                             Depth Tree if needed
                                      |
                                      v
                             4. Initial Reasoning
                                      |
                                      v
                                5. Critique
                                      |
                         +------------+------------+
                         |            |            |
                       Logic        Facts       Coverage
                         |            |            |
                         +------------+------------+
                                      |
                                      v
                              6. Refinement
                                      |
                                      v
                             7. Verification
                                      |
                                      v
                            8. Stopping Decision
                                      |
                         +------------+------------+
                         |                         |
                     More Work                  Converged
                         |                         |
                         v                         v
                    Add Budget                Final Answer
                         |
                         +------> Critique
```

## 6.2 四层架构

### Layer 1：Task Assessment

借鉴 Self-Refine 的动态等级和 cc-thinking 的 Router。

输入任务后，评估：

| 维度 | 说明 |
|---|---|
| Complexity | 问题本身的复杂程度 |
| Uncertainty | 关键结论的不确定性 |
| Stakes | 答案错误的代价 |
| Scope | 任务涉及的子问题数量 |
| Verifiability | 是否可以客观验证 |
| Reversibility | 错误决策是否容易撤销 |

注意：

复杂度与风险不完全相同。

一个问题可能很复杂，但错误代价很低；也可能非常简单，但错误代价极高。

建议分别评估，不要简单相加后只得到一个分数。

初始思考等级：

| 等级 | 适用场景 | 初始策略 |
|---|---|---|
| L0 | 简单查询 | 直接回答 |
| L1 | 一般分析 | 一轮自我检查 |
| L2 | 复杂问题 | 多角度分析 + 反思 |
| L3 | 高风险问题 | 多轮反思 + 独立验证 |

### Layer 2：Reasoning Strategy Router

根据任务类型选择合适的思维框架。

例如：

| 任务类型 | 推荐思维策略 |
|---|---|
| Bug 定位 | Scientific Method |
| 架构选型 | First Principles + Reversibility |
| 技术研究 | Scientific Method + Steel Manning |
| 风险决策 | Pre-mortem + Probabilistic Thinking |
| 系统优化 | Systems Thinking + Theory of Constraints |
| 复杂方案设计 | First Principles + Thought Experiment |
| 事实核查 | Evidence Verification + Adversarial Review |

默认只使用一个主要模型。

只有当主要模型留下明确盲点时，才增加第二个模型。

### Layer 3：Iterative Deliberation

借鉴 Self-Refine。

每轮执行：

1. 形成当前解决方案。
2. 检查逻辑漏洞。
3. 检查事实和证据。
4. 检查遗漏。
5. 寻找反例。
6. 修正答案。
7. 判断是否值得继续。

对于复杂任务，可以增加独立 Agent：

- Agent A：提出解决方案。
- Agent B：寻找反例和失败路径。
- Agent C：独立验证关键结论。

但应保证不同 Agent 有独立职责，而不是让多个 Agent 重复生成相同答案。

### Layer 4：Adaptive Stopping

借鉴 Self-Refine 的收敛机制和 unlazy 的完成验收。

停止条件：

- 原始需求已经得到满足。
- 关键结论具有足够的证据。
- 没有未解决的重大逻辑漏洞。
- 最近一轮没有产生实质性新信息。
- 进一步计算的预期收益不足。
- 达到预算上限。

停止时应明确区分：

- 已经验证的结论。
- 尚未验证的结论。
- 由于预算限制而没有继续检查的内容。

---

# 7. 推荐的 SKILL.md 实现

建议采用一个核心 Skill，而不是将所有功能写成一个无限膨胀的 Prompt。

目录结构：

```text
adaptive-deliberation/
├── SKILL.md
├── references/
│   ├── task-assessment.md
│   ├── reasoning-router.md
│   ├── reflection-framework.md
│   ├── verification.md
│   └── stopping-rules.md
├── templates/
│   ├── reasoning-plan.md
│   └── review-report.md
├── evals/
│   ├── benchmark.md
│   └── evaluation-rubric.md
└── scripts/
    └── evaluate-results.py
```

其中 SKILL.md 只保留主流程和关键约束，具体方法按需加载。

## 7.1 建议的核心指令

以下是核心实现草案：

```markdown
---
name: adaptive-deliberation
description: >
  Dynamically allocate reasoning effort according to task
  complexity, uncertainty, and error cost. Use structured
  reasoning, self-critique, adversarial verification, and
  adaptive stopping to improve task quality.
---

# Adaptive Deliberation

## Core Principle

Do not maximize thinking time.

Allocate sufficient reasoning effort to achieve the
required level of correctness at reasonable cost.

## Phase 1: Assess

Evaluate:

- Task complexity
- Uncertainty
- Error cost
- Scope
- Verifiability

Choose an initial reasoning level.

## Phase 2: Route

Select the smallest set of reasoning methods
that covers the task's actual uncertainties.

Default: one primary method.

Maximum: three complementary methods.

## Phase 3: Decompose

For complex tasks:

- Split into coherent subproblems.
- Define dependencies.
- Define expected outcomes.
- Identify independent verification methods.

Do not create artificial subtasks.

## Phase 4: Reason

Develop an initial solution.

For each critical conclusion:

- Identify assumptions.
- Examine supporting evidence.
- Consider alternatives.
- Identify possible failure modes.

## Phase 5: Critique

Review:

- Logical correctness
- Factual accuracy
- Completeness
- Internal consistency
- Actionability

For high-risk tasks, perform adversarial review.

## Phase 6: Refine

Fix substantive issues.

Do not rewrite merely to create another
thinking round.

## Phase 7: Verify

Use objective checks whenever available:

- Tests
- Calculations
- External sources
- Tool results
- Independent reviewers

## Phase 8: Stop

Stop when:

- Requirements are satisfied.
- Critical claims are verified.
- No material issues remain.
- Additional reasoning has insufficient expected value.

If the budget is exhausted, report unresolved issues.
```

这只是核心骨架。真正落地时，还需要将任务分级、反思模板、验证规则和停止条件写入独立参考文件。

## 7.2 需要区分 Skill 与 Runtime

一个重要的工程边界：

**SKILL.md 只能指导 Agent 如何思考，不能保证模型底层真正增加推理计算量。**

例如：

- 要求多反思一轮，可以增加额外模型调用。
- 要求生成多个候选答案，可以通过 Runtime 调用多次模型。
- 要求选择更高 Reasoning Effort，需要宿主或 API 支持。
- 要求启动独立 Agent，需要 Agent Runtime 支持。
- 要求执行测试，需要工具权限。

因此建议分成两部分：

| 部分 | 职责 |
|---|---|
| Adaptive Deliberation Skill | 任务评估、策略选择、反思和停止规则 |
| Agent Runtime | 模型调用、Token Budget、并行 Agent、工具执行和计量 |

不要把 Prompt 层的思考轮数直接等同于模型内部的推理计算预算。

---

# 8. Benchmark 设计

开发这个 Skill 时，最重要的是证明它确实提高了任务质量，而不是只让回答变长。

建议至少设置六组实验：

| 实验组 | 方法 |
|---|---|
| A | 普通一次性回答 |
| B | 固定一轮 Self-Refine |
| C | 固定三轮 Self-Refine |
| D | 动态思考等级 |
| E | 动态等级 + 思维模型路由 |
| F | 动态等级 + 路由 + 独立验证 + 停止机制 |

## 8.1 测试任务分类

建议覆盖：

1. 事实问答。
2. 数学推理。
3. 代码生成。
4. Bug 定位。
5. 技术架构设计。
6. 技术研究。
7. 复杂决策。
8. 多步骤任务。

每类任务再划分简单、中等和困难三个等级。

## 8.2 评估指标

| 指标 | 目的 |
|---|---|
| Accuracy | 最终答案准确率 |
| Task Success Rate | 实际任务完成率 |
| Completeness | 是否覆盖全部需求 |
| Hallucination Rate | 幻觉和错误事实比例 |
| Regression Rate | 反思后反而变差的比例 |
| Token Consumption | 计算成本 |
| Latency | 响应时间 |
| Cost per Success | 每次成功任务的平均成本 |

尤其需要关注：

```text
Accuracy Gain
    = Accuracy(Skill) - Accuracy(Baseline)

Relative Cost
    = Cost(Skill) / Cost(Baseline)

Cost per Success
    = Total Cost / Successful Tasks
```

实验应固定模型版本、工具权限和输入条件，并采用相同测试集进行配对比较。

对于生成式任务，应使用独立评价器或人工盲评，而不是完全依赖模型自身评价。

---

# 9. 最终建议

## 9.1 三个 Skill 的借鉴优先级

| 优先级 | 项目 | 应借鉴的核心机制 |
|---|---|---|
| 1 | Self-Refine | 多轮反思、动态深度、收敛停止 |
| 2 | cc-thinking-skills | 思维模型路由、组合分析 |
| 3 | unlazy | 复杂任务分解、验收门槛、客观验证 |

这里的借鉴优先级与前面的综合排名不同：前者考虑的是开发目标的直接匹配度，后者还考虑了项目关注度。

## 9.2 推荐组合

我不建议直接安装三个 Skill 后要求 Agent 每次全部执行。

更合理的组合方式：

```text
              Adaptive Deliberation
                       |
                       v
               Task Assessment
                       |
                       v
              cc-thinking-skills
                 思维策略路由
                       |
                       v
                Self-Refine
                多轮反思修正
                       |
                       v
                  unlazy
               任务完成验证
                       |
                       v
                Final Answer
```

具体职责：

- cc-thinking-skills 决定如何分析。
- Self-Refine 决定如何批判和改进答案。
- unlazy 决定如何验证复杂任务是否真正完成。

对于简单任务，不启动完整流程。

对于复杂研究任务，重点使用思维模型路由和多轮反思。

对于复杂编程任务，增加任务树和可执行验收门槛。

## 9.3 最终结论

三个项目分别提供了自适应深度思考系统所需要的三个核心能力：

1. **Thinking Strategy**：选择正确的思考方法。
2. **Iterative Deliberation**：通过多轮反思改善答案。
3. **Objective Verification**：通过外部证据验证结果。

真正值得开发的不是简单的 `think-more`，而是：

**Adaptive Deliberation：根据任务复杂度、错误代价和边际思考收益，动态调度推理资源的通用 Agent Skill。**

建议先实现纯 Prompt 版本，使用固定模型和 Benchmark 验证收益；随后再增加 Runtime 层的 Token Budget、独立 Agent 和工具验证能力。

最终目标不是让 Agent 永远思考，而是让它在值得投入计算的地方充分思考，在已经得到可靠答案时及时停止。

---

# 10. 参考链接

## 开源项目

1. [Leonxlnx/unlazy](https://github.com/Leonxlnx/unlazy)
2. [unlazy/SKILL.md](https://github.com/Leonxlnx/unlazy/blob/main/SKILL.md)
3. [unlazy/Depth Tree Method](https://github.com/Leonxlnx/unlazy/blob/main/references/method.md)
4. [tjboudreaux/cc-thinking-skills](https://github.com/tjboudreaux/cc-thinking-skills)
5. [thinking-model-router/SKILL.md](https://github.com/tjboudreaux/cc-thinking-skills/blob/main/skills/thinking-model-router/SKILL.md)
6. [thinking-model-combination/SKILL.md](https://github.com/tjboudreaux/cc-thinking-skills/blob/main/skills/thinking-model-combination/SKILL.md)
7. [Thomaszhou22/self-refine-skill](https://github.com/Thomaszhou22/self-refine-skill)
8. [Self-Refine SKILL.md](https://github.com/Thomaszhou22/self-refine-skill/blob/main/SKILL.md)
9. [Self-Refine BENCHMARK.md](https://github.com/Thomaszhou22/self-refine-skill/blob/main/BENCHMARK.md)

## 相关学术研究

以下是这些 Skill 所引用或相关的基础研究。它们不属于本次近一年论文筛选范围，仅用于解释实现理念的学术来源。

1. Madaan et al. (2023). *Self-Refine: Iterative Refinement with Self-Feedback*. https://arxiv.org/abs/2303.17651
2. Shinn et al. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning*. https://arxiv.org/abs/2303.11366
3. Dhuliawala et al. (2023). *Chain-of-Verification Reduces Hallucination in Large Language Models*. https://arxiv.org/abs/2309.11495
4. Gou et al. (2023). *CRITIC: Large Language Models Can Self-Correct with Tool-Interactive Critiquing*. https://arxiv.org/abs/2305.11738

---

*调研说明：GitHub Stars 为调研时公开页面显示的近似值，会随时间变化。项目自身的 Benchmark、效果声明和设计理念均作为项目方提供的证据处理，不等同于独立学术验证。源码版本、复跑结果及 Prompt/Runtime/验证分层见[本地资料汇总](local_reference_summary.md)。*
