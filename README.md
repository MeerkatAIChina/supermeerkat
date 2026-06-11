# supermeerkat

> 多 Agent 协作工作流引擎 —— 将商业问题分解为可执行的步骤，调度 AI Agent 团队协同完成调研、分析、策略制定与质量审查。

supermeerkat 是一个面向**消费品行业**的 Claude Code 插件。它内置了四个专业 AI 角色（Orchestrator、Research、Analysis、Critique），通过三个斜杠命令，让非技术用户也能创建和执行复杂的多步骤商业分析工作流。

---

## Overview

在消费品行业，一个完整的营销决策通常需要经过竞品调研 → 卖点提炼 → 策略制定 → 方案审查等多个环节。传统做法是人工逐一完成，或向 AI 多次提问、手动串联结果。

supermeerkat 将这个过程**自动化**：

1. 用自然语言描述你的业务目标
2. Orchestrator Agent 将目标拆解为工作流步骤
3. 按步骤调度 Research → Analysis → Critique Agent 协同执行
4. 整合 Critique 审查意见，生成最终报告

**用户永远不需要手写或编辑工作流文件。** 所有操作通过对话命令完成。

---

## Installation

### 从官方市场安装

```
/plugin install supermeerkat@claude-plugins-official
```

### 本地开发安装

```
cc --plugin-dir /path/to/supermeerkat
```

安装后运行 `/reload-plugins` 使插件生效。

### 系统要求

- Claude Code 最新版本
- 含网络搜索的步骤需要 `WebSearch` 和 `WebFetch` 工具可用

---

## Quick Start

### 1. 创建工作流

```
/supermeerkat:create-workflow
```

系统会询问两个问题：
- **工作流目标**：你想解决什么问题？
- **工作流名称**：给它起个名字

之后 Orchestrator 会追问最多 3 个关键问题来澄清需求，然后生成完整的工作流草稿供你确认。

```
示例输入：
  "帮我为即将上市的功能饮料制定完整的营销策略方案"

示例输出：
  生成 4 步工作流：竞品调研 → 卖点分析 → 策略制定 → Critique 审查
```

### 2. 执行工作流

```
/supermeerkat:run-workflow
```

选择要执行的工作流。系统按步骤依次调度 Agent，在执行过程中：
- 遇到 `pauseAfter` 标记的关键步骤，暂停展示中间结果等待确认
- 步骤失败时提供重试 / 跳过 / 中止三个选项
- 若执行中断，下次运行时可从断点继续

### 3. 查看工作流

```
/supermeerkat:workflow-list
```

列出所有已保存的工作流，展示名称、描述、步骤数、状态和创建时间。

---

## Commands

### `/supermeerkat:create-workflow`

创建新的工作流。

- **触发方式**：直接运行命令，或附带目标描述
- **交互流程**：收集需求 → Orchestrator 设计（≤ 3 问） → 生成草稿 → 用户确认 → 保存
- **输出**：`workflows/<name>.md`（Markdown + JSON 混合格式）
- **适用场景**：首次使用、新建营销方案、新建分析项目

### `/supermeerkat:run-workflow`

执行已创建的工作流。

- **触发方式**：直接运行命令进入选择列表，或附带名称直接执行：`/supermeerkat:run-workflow 爆款营销`
- **执行模式**：串行执行（MVP），按 DAG 拓扑排序依次调度
- **断点续跑**：自动检测未完成的执行，提供从断点继续 / 重新执行 / 查看详情三种选择
- **关键点暂停**：`pauseAfter: true` 的步骤完成后暂停，展示摘要并等待用户指令
- **错误恢复**：步骤失败时提供重试 / 跳过 / 中止选项
- **最终报告**：综合所有产物和 Critique 审查意见，逐条回应（已采纳修订 / 暂不采纳附理由）
- **适用场景**：执行已确认的工作流、从断点恢复失败执行

### `/supermeerkat:workflow-list`

查看所有已保存的工作流。

- **触发方式**：直接运行命令
- **数据源**：扫描 `workflows/` 目录（唯一真实数据源）
- **输出格式**：表格展示（名称 | 描述 | 步骤数 | 状态 | 版本 | 创建时间）
- **适用场景**：查看已有工作流、确认工作流状态、执行前查阅

---

## Built-in Agents

### Orchestrator — 编排器

工作流执行的核心调度者。不直接回答业务问题，而是：

- 解析工作流文件的 DAG 结构（提取 JSON 步骤定义、校验依赖关系、检测循环依赖）
- 管理执行状态（`run-state.json` 持久化，状态流转：`pending → in_progress → completed/failed/skipped`）
- 调度 Specialist Agent（传入任务、输入/输出绝对路径、超时限制）
- 处理暂停点和错误恢复
- 整合最终报告（综合产物 + 逐条回应 Critique 审查意见）

**可用工具**：`Agent` `Read` `Write` `Glob` `Grep`

### Research — 研究员

信息检索与资料收集。

- 接收检索任务和文件路径
- 使用 `WebSearch` 搜索（宽泛关键词 → 精准深入，至少 3 个来源交叉验证）
- 使用 `WebFetch` 深入阅读关键页面
- 输出结构化调研报告（核心发现 → 详细内容 → 来源标注 → 信息空白）
- **铁律**：结论先行、来源必标注、不编造不推测

**可用工具**：`Read` `Write` `WebSearch` `WebFetch`

### Analysis — 分析师

数据分析与策略推导。

- 读取输入文件，进行结构化拆解和多角度审视（市场 / 用户 / 竞争 / 可行性）
- 每条结论附带支撑论据和置信度（高 / 中 / 低）
- 输出格式：核心结论（3-5 条）→ 详细分析 → 风险与不确定性
- **铁律**：证据驱动、不确定性必标注、空输入时明确说明局限性

**可用工具**：`Read` `Write` `Grep` `Glob`

### Critique — 独立审查员

质量控制的最后防线。唯一职责是**挑毛病**。

- 按四维度审查其他 Agent 的产出：
  1. **逻辑漏洞** — 推理链条是否有断裂？因果关系是否成立？
  2. **事实错误** — 陈述事实是否有误？是否存在反例？
  3. **遗漏风险** — 忽略了哪些场景、因素或替代方案？
  4. **可改进空间** — 哪里可以更好？具体方案是什么？
- 每条发现必须包含：问题描述 + 反驳论据 + 具体改进方案（三要素缺一不可）
- **铁律**：严禁附和原文、不能只说"可以更好"、不得输出空文件

**可用工具**：`Read` `Write`

---

## Architecture

```
                          ┌──────────────────────┐
                          │       用户命令         │
                          └──────────┬───────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Skill Layer         │
                          │   create-workflow     │
                          │   run-workflow        │
                          │   workflow-list       │
                          └──────────┬───────────┘
                                     │ Agent 工具
                          ┌──────────▼───────────┐
                          │   Orchestrator Agent   │
                          │   · 解析 DAG           │
                          │   · 管理 run-state     │
                          │   · 调度 Specialist    │
                          └──┬───────┬───────┬────┘
                             │       │       │
                    ┌────────▼┐ ┌────▼──┐ ┌─▼────────┐
                    │ Research │ │Analysis│ │ Critique  │
                    │  Agent   │ │ Agent  │ │  Agent    │
                    └────┬─────┘ └───┬───┘ └─────┬─────┘
                         │           │            │
                         └───────────┼────────────┘
                                     │ 读写
                          ┌──────────▼───────────┐
                          │   Shared Storage      │
                          │   shared/artifacts/    │
                          │   <run-id>/            │
                          │   · run-state.json    │
                          │   · research-*.md     │
                          │   · analysis-*.md     │
                          │   · critique-*.md     │
                          │   · final-report.md   │
                          └──────────────────────┘
```

**核心设计决策**：

| 决策 | 选择 | 理由 |
|------|------|------|
| 调度模型 | Orchestrator 中心化 | 职责清晰、集中调度、易于调试 |
| Agent 间通信 | 文件系统（绝对路径） | 简单可靠、可调试、对用户透明 |
| 状态持久化 | `run-state.json` | 上下文窗口消失后状态不丢失，支持断点续跑 |
| 工作流格式 | Markdown + JSON 注释 | 人可读 + 机器可精确解析 |
| Specialist 隔离 | 不使用 worktree | 避免跨 worktree 文件路径冲突 |

---

## Workflow File Format

工作流文件是由 `/supermeerkat:create-workflow` 自动生成的 Markdown 文档，在正文中嵌入 JSON 注释定义步骤。

**用户无需手动编辑。** 以下是生成文件的内部结构参考：

```markdown
# 工作流标题

**描述**: 工作流描述
**版本**: 1.0
**创建时间**: 2026-06-11

## 步骤

### 步骤 1：竞品调研

<!-- workflow:step
{
  "id": "step-1",
  "agent": "research",
  "task": "搜索并收集主要竞品的营销策略",
  "dependsOn": [],
  "output": "research-findings.md",
  "timeout": 120
}
-->

### 步骤 2：卖点分析

<!-- workflow:step
{
  "id": "step-2",
  "agent": "analysis",
  "task": "基于竞品调研结果提炼核心卖点",
  "dependsOn": ["step-1"],
  "input": "research-findings.md",
  "output": "analysis-sellpoints.md",
  "pauseAfter": true,
  "timeout": 60
}
-->
```

**JSON 步骤字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|:--:|------|
| `id` | string | ✅ | 步骤唯一标识，跨步骤 `dependsOn` 引用 |
| `agent` | string | ✅ | `research` / `analysis` / `critique` / `orchestrator` |
| `task` | string | ✅ | 传给 Agent 的具体任务描述 |
| `dependsOn` | string[] | ✅ | 前置依赖步骤 ID，空数组表示无依赖 |
| `input` | string | ❌ | 从共享存储读取的文件名 |
| `output` | string | ✅ | 输出到共享存储的文件名 |
| `pauseAfter` | boolean | ❌ | 完成后是否暂停等待确认（默认 `false`） |
| `timeout` | number | ❌ | 协作式软超时（秒），默认 `120` |

---

## Features

### 零代码工作流创建

所有工作流通过 `/supermeerkat:create-workflow` 命令创建。Orchestrator 通过 brainstorming 理解需求，自动设计步骤、分配 Agent、设定输出格式。用户只需用自然语言描述目标。

### 断点续跑

每次执行的状态写入 `run-state.json`。若执行中断（Ctrl+C、Agent 异常、系统崩溃），下次运行 `/supermeerkat:run-workflow` 时自动检测未完成的执行，提供三种恢复选项：
- **从断点继续** — 跳过已完成步骤，从失败处恢复
- **重新执行** — 创建新执行，保留旧产物
- **查看详情后决定** — 展示完整状态后再选择

### 关键点暂停

在工作流的关键决策步骤标记 `pauseAfter: true`，步骤完成后暂停：
- 展示中间结果摘要
- 提供继续 / 修改后重试 / 中止执行三个选项
- 确保人在关键判断点保持控制权

### 对抗性审查

Critique Agent 作为独立审查员，以"挑毛病"为唯一职责：
- 四维度审查（逻辑 / 事实 / 遗漏 / 改进）
- 每条发现必须包含问题描述 + 反驳论据 + 改进方案
- 审查意见在最终报告中逐条回应，形成可追溯的质量记录

### 执行可追溯

每次执行生成独立的 `shared/artifacts/<run-id>/` 目录，保存：
- `run-state.json` — 步骤执行状态
- 各步骤中间产物
- `final-report.md` — 整合审查意见的最终报告

中间产物默认保留，便于回溯、对比和审计。

### 数据源唯一

`workflow-list` 和 `run-workflow` 均以 `workflows/` 目录扫描为唯一数据源。不存在索引文件与实际文件不同步的风险。

---

## Design Philosophy

- **Orchestrator 负责元认知** — 收到用户输入后，不直接回答，而是先进行需求分析、方案设计，再调度执行
- **共享存储是通信总线** — Agent 之间不直接通话，通过读写共享文件传递上下文；简单、可调试、对用户透明
- **Critique 是质量锚点** — 独立审查形成对抗性检验，防止 AI 输出间的"回声室效应"
- **Harness 工程理念** — 遵循 brainstorming → spec → plan → implement → review 的流程化设计
- **渐进式披露** — MVP 仅开放核心闭环（创建 → 执行 → 审查），复杂功能（并行调度、自定义 Agent）按需延后

---

## Best Practices

### 工作流设计

✅ 将复杂任务拆分为 3-5 个聚焦的步骤，每步一个明确产出
✅ 最后一步始终设为 Critique Agent 审查
✅ 在重要决策步骤使用 `pauseAfter: true`，保持人工判断介入
✅ 为不同任务使用不同的 `timeout` 值（搜索密集型给 120s，分析推理给 60s）

### 执行管理

✅ 关注 `pauseAfter` 暂停点，认真审查中间结果后再继续
✅ 执行中断后优先使用断点续跑，避免重复工作
✅ 定期查看 `workflow-list` 了解工作流状态
✅ 中间产物保留一段时间后再清理，便于回溯对比

### 安全与隐私

✅ 工作流文件中可能包含业务敏感信息，注意 `.gitignore` 配置
✅ 运行时产物目录（`shared/artifacts/run-*/`）不会提交到版本控制
✅ Research Agent 的网络搜索会访问公开网页，注意搜索关键词中不含机密信息

---

## Directory Structure

```
supermeerkat/
├── plugin.json                             # 插件清单（agents + skills 注册）
├── README.md
├── LICENSE
├── .gitignore
├── agents/                                 # Agent 系统提示定义
│   ├── orchestrator.md                     # 编排器 — DAG 解析、状态管理、调度
│   ├── research.md                         # 研究员 — 信息检索、资料收集
│   ├── analysis.md                         # 分析师 — 数据分析、策略推导
│   └── critique.md                         # 审查员 — 四维度独立审查
├── skills/                                 # 斜杠命令实现
│   ├── create-workflow/
│   │   ├── SKILL.md                        # 创建工作流命令
│   │   └── references/
│   │       └── workflows.md                # 工作流索引（辅助缓存）
│   ├── run-workflow/
│   │   └── SKILL.md                        # 执行工作流命令（含断点续跑）
│   └── workflow-list/
│       └── SKILL.md                        # 列出工作流命令
├── workflows/                              # 用户创建的工作流文件
│   └── .gitkeep
├── shared/
│   └── artifacts/                          # 运行时产物（每次执行一个 run-id 目录）
│       └── .gitkeep
└── commands/                               # 预留：独立命令定义
    └── .gitkeep
```

---

## Contributing

欢迎贡献。流程如下：

1. Fork 本仓库
2. 创建功能分支：`git checkout -b feat/your-feature`
3. 提交变更：`git commit -m "feat: description"`
4. 推送到分支：`git push origin feat/your-feature`
5. 创建 Pull Request

开发前请阅读：
- `docs/superpowers/specs/2026-06-11-supermeerkat-plugin-design.md` — 设计规格文档
- `docs/superpowers/plans/2026-06-11-supermeerkat-plugin-implementation.md` — 实施计划

---

## License

MIT License — 详见 [LICENSE](LICENSE) 文件。
