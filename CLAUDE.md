# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

supermeerkat 是一个 Claude Code 插件，提供多 Agent 协作工作流引擎。目标用户是消费品行业的非技术人员。MVP 阶段通过 4 个内置 Agent 和 3 个斜杠命令实现工作流的创建、执行和查询。

## 核心架构

### 三层结构

```
Skill Layer（用户入口）→ Orchestrator Agent（调度核心）→ Specialist Agents（执行单元）
```

- **Skill Layer** (`skills/*/SKILL.md`)：处理用户交互，委托给 Orchestrator
- **Orchestrator** (`agents/orchestrator.md`)：解析 DAG、管理 run-state.json、调度 Agent、生成最终报告
- **Specialists**：Research（搜索）、Analysis（分析）、Critique（审查），各司其职

### Agent 间通信

**唯一通信方式：文件系统共享存储**，绝对路径。Agent 不直接对话，通过读写 `shared/artifacts/<run-id>/` 下的文件传递上下文。不使用 `isolation: worktree`（避免跨 worktree 路径冲突）。

### Workflow 格式

Markdown + JSON 注释混合。用户通过对话生成（`/supermeerkat:create-workflow`），永远不需要手动编辑。步骤定义嵌入在 `<!-- workflow:step { ... } -->` HTML 注释中。

JSON 步骤字段：`id` / `agent` / `task` / `dependsOn` / `output`（必填），`input` / `pauseAfter` / `timeout`（选填）。

### 状态管理

Orchestrator 在 `shared/artifacts/<run-id>/run-state.json` 中持久化步骤状态。状态流转：`pending` → `in_progress` → `completed` / `failed` / `skipped`。这使上下文窗口消失后状态不丢失，支持断点续跑。

## 关键设计决策

- **MVP 仅串行执行**：即使 DAG 中有无依赖的并行步骤，也按拓扑排序依次调度。并行是后续迭代方向。
- **Critique 非迭代闭环**：Critique Agent 审查后输出意见，Orchestrator 在最终报告中逐条回应（已采纳/暂不采纳），但不会"修订→再审查"循环。
- **timeout 是协作式软超时**：Orchestrator 将时限写入 Agent prompt，不强制中断。Agent 工具无原生 timeout 参数。
- **数据源唯一**：`workflow-list` 和 `run-workflow` 均以 `workflows/` 目录扫描为准，不依赖 `references/workflows.md`（后者仅由 `create-workflow` 写入维护）。

## 插件开发要点

### 文件约定

- **Manifest**：`.claude-plugin/plugin.json`（仅包含元数据：name, version, description, author, license, keywords）
- **Agent 系统提示**：`agents/<name>.md`，通过 YAML frontmatter 声明 `name`、`description`、`model`。Agent 由目录扫描自动发现，无需在 manifest 中显式注册
- **Skill 实现**：`skills/<name>/SKILL.md`，YAML frontmatter 包含 `name`、`description`、`version`。Skill 也由目录扫描自动发现
- Skill 通过 Glob 搜索 `workflows/` 目录定位插件根路径（而非依赖环境变量）

### Agent 设计原则

- 遵循最小权限：Critique 只有 `Read`/`Write`，Research 无 `Grep`，仅 Orchestrator 拥有 `Agent` 工具
- Specialist Agent 的 systemPrompt 必须包含：可用工具声明、工作方式（含输入文件不存在处理）、输出格式模板、铁律（硬约束）
- Orchestrator 调用 Specialist 时必须传入绝对路径的 prompt 模板，明确规定"必须写入输出文件、不修改其他文件、不输出空文件"

### 错误处理模式

Agent 调用失败 / 超时 / 输出未生成 → Orchestrator 提供 重试/跳过/中止 三选项。用户中断（Ctrl+C）→ 产物保留，下次执行时检测 run-state.json 提供断点续跑。

## 开发命令

本地开发测试：

```
cc --plugin-dir /path/to/supermeerkat
```

安装后运行 `/reload-plugins` 使修改生效。

验证 manifest 合法性：

```powershell
Get-Content .claude-plugin/plugin.json | ConvertFrom-Json | Out-Null
```

Git 历史清晰：每个 Task 一个 commit，feat/fix 前缀，中文 commit message。

## 参考文档

- `docs/superpowers/specs/2026-06-11-supermeerkat-plugin-design.md` — 完整设计规格
- `docs/superpowers/plans/2026-06-11-supermeerkat-plugin-implementation.md` — 实施计划（含所有文件的完整内容）
