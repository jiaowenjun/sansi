# 三思（sansi）

用额外的任务级思考、替代路径和验证弥补模型智能的不足：简单任务不浪费计算，复杂、不确定或高代价任务主动增加思考深度。

**核心理念：** 三思而后行，按需思考。

**三思协议：** `思前评估 → 思中观察与追加 → 思后挑战、验证与停止`

“三思”不是机械地重复三次。Skill 会选择最低充分深度：简单请求直接完成；出现多步依赖、证据冲突、重要取舍或高失败代价时，才增加替代路径、复核和外部验证。

## 使用

Skill 名称为 `sansi`，正式标识是 `$sansi`；也支持 `$三思`、`三思`、`请三思`、`三思一下`、`请三思后再回答` 等中文自然语言显式触发。`三思而后行` 表示按当前方案开始执行，并在执行中启用三思。显式调用不会强制最高深度，也不会扩大模型、工具或外部操作权限。

```text
使用 $sansi 评审这个迁移方案，重点检查回滚和数据一致性。
$三思：评审这个迁移方案，重点检查回滚和数据一致性。
请三思后再回答：这个迁移方案最可能在哪里失败？
三思而后行
```

只有这些表达被用作当前任务的指令时才触发；例如“解释‘三思而后行’的出处”是在讨论词语，不是要求启用 Skill。“三思而后行”的执行语义只覆盖当前已明确且可安全推进的方案，不替代高风险操作所需的授权或关键用户选择。

## 目录结构

```
sansi/
├── SKILL.md               # Skill 入口：深度路由、三思协议和停止条件
├── agents/openai.yaml     # Codex 界面元数据
├── references/
│   └── verification.md   # 按任务类型选择验证方法
├── evals/
│   └── cases.yaml         # 行为评测场景与不变量
├── docs/                  # 调研与证据底稿
│   ├── research_review.md # 论文综述：自适应推理预算方向 8 篇代表性论文
│   ├── skill_review_v1.md # Skill 调研（一）：unlazy / cc-thinking-skills 等 5 个候选，精选 3 个
│   ├── skill_review_v2.md # Skill 调研（二）：三思而后行——Reasoning Workflow / Deep Research / Self-Refine
│   └── local_reference_summary.md # 本地 PDF 全文与 5 个参考仓库的证据汇总
├── papers/                # 论文 PDF（本地资料，已 gitignore）
└── reference-skills/      # 参考的开源 Skill 仓库克隆（本地资料，已 gitignore）
```

## 文档导航

| 文档 | 内容 |
|---|---|
| [research_review.md](docs/research_review.md) | 近一年自适应测试时计算分配方向的论文综述，含 Plan and Budget、Adaptive Thinking、EAGER、AERA 等 8 篇 |
| [skill_review_v1.md](docs/skill_review_v1.md) | GitHub 开源 Agent Skill 调研：unlazy、cc-thinking-skills、self-refine-skill 的深度分解、验证与思维路由机制 |
| [skill_review_v2.md](docs/skill_review_v2.md) | 「三思而后行」理念调研：自适应思考深度分级、预算分配与迭代收敛停止机制 |
| [local_reference_summary.md](docs/local_reference_summary.md) | 8 篇本地 PDF 与 5 个参考仓库的版本快照、逐项证据、源码边界、测试结果和设计映射 |

## 设计边界

- Skill 控制可观察的任务流程，不声称能测量或保证模型内部推理 Token。
- 每轮追加投入必须解决明确缺口；没有新证据或实质修正时及时停止。
- 验证强度与风险相称，优先测试、原始来源和独立证据，不以模型自评代替事实。
- 多 Agent、额外模型调用和更高 reasoning effort 仅在宿主支持、权限允许且收益值得时使用。

## 验证

使用 Skill Creator 的校验器检查目录、名称、frontmatter 和脚手架残留：

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" .
```

## 本地资料说明

`papers/` 与 `reference-skills/` 为体积较大的本地研究资料，已在 `.gitignore` 中排除。克隆参考仓库：

```bash
git clone https://github.com/zy839971925-zyy/Agent--deep-research-Workflow-Skills reference-skills/Agent--deep-research-Workflow-Skills
```

本次核查固定到以下本地提交：`unlazy@1667149`、`cc-thinking-skills@7b8fece`、`self-refine-skill@620cb9b`、`Agent--deep-research-Workflow-Skills@c4de05d`、`moonlight-agent-skills@e943e32`。完整测试与证据边界见[本地资料汇总](docs/local_reference_summary.md)。
