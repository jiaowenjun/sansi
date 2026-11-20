# Adaptive Deliberation：自适应审慎思考研究

通过自适应推理预算（Adaptive Test-Time Compute Allocation）提高 AI Agent 任务准确率的研究项目：根据任务复杂度、错误风险和推理中的不确定性，动态决定思考深度、推理次数与验证轮数。

**核心闭环**：`任务评估 → 初始预算分配 → 执行与观察 → 必要时增加预算 → 独立复核 → 收敛停止`

## 目录结构

```
sansi/
├── docs/                  # 调研报告（入库）
│   ├── research_review.md # 论文综述：自适应推理预算方向 8 篇代表性论文
│   ├── skill_review_v1.md # Skill 调研（一）：unlazy / cc-thinking-skills 等 5 个候选，精选 3 个
│   └── skill_review_v2.md # Skill 调研（二）：三思而后行——Reasoning Workflow / Deep Research / Self-Refine
├── papers/                # 论文 PDF（本地资料，已 gitignore）
└── reference-skills/      # 参考的开源 Skill 仓库克隆（本地资料，已 gitignore）
```

## 文档导航

| 文档 | 内容 |
|---|---|
| [research_review.md](docs/research_review.md) | 近一年自适应测试时计算分配方向的论文综述，含 Plan and Budget、Adaptive Thinking、EAGER、AERA 等 8 篇 |
| [skill_review_v1.md](docs/skill_review_v1.md) | GitHub 开源 Agent Skill 调研：unlazy、cc-thinking-skills、self-refine-skill 的深度分解、验证与思维路由机制 |
| [skill_review_v2.md](docs/skill_review_v2.md) | 「三思而后行」理念调研：自适应思考深度分级、预算分配与迭代收敛停止机制 |

## 下一步

基于三份调研，将各项目核心机制整合为一个独立的 **Adaptive Deliberation Skill**（不简单叠加使用），用于 Codex、Claude Code、Cursor 等支持 Agent Skills 的工具。

## 本地资料说明

`papers/` 与 `reference-skills/` 为体积较大的本地研究资料，已在 `.gitignore` 中排除。克隆参考仓库：

```bash
git clone https://github.com/zy839971925-zyy/Agent--deep-research-Workflow-Skills reference-skills/Agent--deep-research-Workflow-Skills
```
