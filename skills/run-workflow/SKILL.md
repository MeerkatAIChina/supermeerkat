---
name: run-workflow
description: 执行一个已创建的工作流，按步骤依次运行并生成最终报告。
version: 1.0.0
---

# run-workflow — 执行工作流

你是 supermeerkat 的 run-workflow 命令。你的职责是让用户选择并执行 workflow，执行逻辑委托给 Orchestrator Agent。

## 执行流程

### 第一步：扫描可用 Workflow

使用 Glob 工具搜索 `<plugin-root>/workflows/*.md`：
- 扫描 `workflows/` 目录下所有 `.md` 文件
- 排除 `.gitkeep`
- 若目录下无 workflow 文件，告知用户"尚未创建任何 workflow，请先使用 /supermeerkat:create-workflow 创建"
- `<plugin-root>` 即当前插件根目录，可通过对 `workflows/` 目录执行 Glob 定位

### 第二步：解析 Workflow 元数据

对每个 workflow 文件，Read 前 5 行提取：
- 标题（第一个 `# ` 开头的行）
- 描述（`**描述**` 字段）
- 状态（`**状态**` 字段）

### 第三步：展示列表让用户选择

以表格形式展示：

```
可用 Workflow：

| # | 名称 | 描述 | 状态 |
|---|------|------|------|
| 1 | 爆款营销 | 新品上市营销策略 | 就绪 |
| 2 | 竞品分析 | 竞品深度分析报告 | 就绪 |

请输入编号或名称选择要执行的 workflow：
```

- 用户可通过编号或名称选择
- 验证用户输入有效（编号在范围内或名称匹配）
- 如用户直接运行 `/supermeerkat:run-workflow <名称>`，跳过选择步骤

### 第四步：断点续跑检查

在启动执行前，检查是否存在未完成的执行：

1. 使用 Glob 搜索 `shared/artifacts/run-*/run-state.json`
2. 对每个找到的 `run-state.json`，Read 内容检查：
   - `steps` 中是否存在 `in_progress` 或 `failed` 状态
   - `workflow` 字段是否与当前选择的工作流名称匹配
3. 若存在匹配的未完成执行：
   ```
   检测到未完成的执行：

   | Run ID | 开始时间 | 进度 |
   |--------|----------|------|
   | run-20260611-143022 | 2026-06-11T14:30:22Z | 2/4 步骤完成，step-3 失败 |

   请选择：
   A. 从断点继续 — 跳过已完成步骤，从失败处恢复
   B. 重新执行 — 创建新的执行，保留旧产物
   C. 查看详情后决定 — 展示 run-state.json 完整内容
   ```
4. 若用户选择 A（断点续跑）：复用已有 run-id
5. 若用户选择 B（重新执行）：生成新 run-id

### 第五步：生成 Run ID 和路径

- **断点续跑**：复用已有 `run-id`
- **新执行**：生成 `run-id = "run-YYYYMMDD-HHmmss"`（当前 UTC 时间）
- 共享存储绝对路径：`<plugin-root>/shared/artifacts/<run-id>/`

### 第六步：调用 Orchestrator 执行

使用 Agent 工具调用 Orchestrator agent：

```
agent: orchestrator
prompt: |
  请执行以下 workflow 文件。

  Workflow 文件绝对路径: <plugin-root>/workflows/<name>.md
  Run ID: <run-id>
  共享存储绝对路径: <plugin-root>/shared/artifacts/<run-id>/
  断点续跑: <true 或 false>

  执行说明：
  1. 解析 workflow 文件中的步骤
  2. 若断点续跑=true，读取已有 run-state.json，跳过 completed 步骤
  3. 若断点续跑=false，创建新的 run-state.json
  4. 按依赖关系串行执行每个步骤
  5. pauseAfter=true 的步骤完成后暂停并等待用户确认
  6. 所有步骤完成后，综合产物和 Critique 审查意见生成最终报告

  请在每一步完成后报告进度。
```

### 第七步：展示执行结果

Orchestrator 返回后：
- 确认 `final-report.md` 已生成
- 展示执行摘要（从 run-state.json 读取各步骤状态）
- 告知用户中间产物位置和最终报告位置

```
✅ Workflow "<名称>" 执行完成

📊 执行摘要:
  step-1 (竞品调研): ✅ 完成
  step-2 (卖点分析): ✅ 完成
  step-3 (策略制定): ✅ 完成
  step-4 (Critique审查): ✅ 完成

📁 最终报告: shared/artifacts/<run-id>/final-report.md
📁 中间产物: shared/artifacts/<run-id>/

是否保留中间产物？ [保留 / 仅保留最终报告]
```

## 错误处理

- workflow 文件解析失败 → 告知用户哪个文件有问题，建议重新 create-workflow
- Orchestrator 执行中断 → 告知用户断点 run-id，说明可通过重新执行 /supermeerkat:run-workflow 恢复
- 共享存储目录创建失败 → 报告错误，检查磁盘空间和权限
