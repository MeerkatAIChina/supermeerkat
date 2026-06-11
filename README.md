# supermeerkat

> 多 Agent 协作工作流引擎 — 让 AI 帮你完成竞品调研、卖点分析、策略制定、质量审查。

面向**消费品行业**的非技术用户（老板、产品经理、销售），通过简单的对话命令，自动调度多个 AI 角色协同完成复杂的商业分析任务。

## 快速开始

### 安装

在 Claude Code 中安装插件：

```
/plugin install supermeerkat
```

### 三步上手

**1. 创建一个工作流**

```
/supermeerkat:create-workflow
```

按提示描述你的业务目标（例如："帮我分析新品上市的营销策略"），系统会自动生成包含多个步骤的工作流文件。

**2. 执行工作流**

```
/supermeerkat:run-workflow
```

选择要执行的工作流，系统会按步骤依次调度 Research → Analysis → Critique Agent 协同完成任务。

**3. 查看已有工作流**

```
/supermeerkat:workflow-list
```

查看所有已创建的工作流及其状态。

## 内置 Agent 团队

| Agent | 角色 | 能力 |
|-------|------|------|
| **Orchestrator** | 编排器 | 解析工作流、调度任务、管理状态、生成最终报告 |
| **Research** | 研究员 | 网络搜索、信息检索、资料收集 |
| **Analysis** | 分析师 | 数据分析、逻辑推理、策略推导 |
| **Critique** | 审查员 | 独立审查、漏洞发现、改进建议 |

## 工作流示例

一个典型的「新品上市营销策略」工作流包含 4 个步骤：

```
步骤 1: Research Agent  →  竞品调研（搜索竞品策略、定价、人群）
步骤 2: Analysis Agent  →  卖点分析（提炼核心卖点与差异化定位）
步骤 3: Analysis Agent  →  策略制定（制定完整营销方案）
步骤 4: Critique Agent  →  审查（四维度审查，找出漏洞和改进空间）
```

Orchestrator 整合所有产出和 Critique 审查意见，生成最终报告。

## 特性

- **零代码**：所有工作流通过对话创建，无需手写任何文件
- **断点续跑**：执行中断后可从断点恢复，不丢失已完成步骤的产物
- **关键点暂停**：可在关键步骤后暂停，查看中间结果后再决定是否继续
- **质量控制**：Critique Agent 独立审查其他 Agent 的产出，形成对抗性检验
- **可追溯**：每次执行产物独立保存，便于回溯和对比

## 目录结构

```
supermeerkat/
├── plugin.json              # 插件清单
├── agents/                  # Agent 系统提示
│   ├── orchestrator.md
│   ├── research.md
│   ├── analysis.md
│   └── critique.md
├── skills/                  # 斜杠命令
│   ├── create-workflow/
│   ├── run-workflow/
│   └── workflow-list/
├── workflows/               # 你的工作流文件
└── shared/artifacts/        # 运行时产物
```

## 系统要求

- Claude Code（最新版本）
- 需要网络搜索功能的步骤需确保 WebSearch 工具可用

## 许可

MIT
