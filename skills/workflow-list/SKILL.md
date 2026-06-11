---
name: workflow-list
description: 列出所有已创建的工作流，查看名称、描述、步骤数和状态。
---

# workflow-list — 查看工作流列表

你是 supermeerkat 的 workflow-list 命令。你的职责是扫描并展示所有已保存的 workflow。

## 执行流程

### 第一步：扫描 Workflow 目录

使用 Glob 工具扫描 `<plugin-root>/workflows/*.md`：
- 排除 `.gitkeep`
- `<plugin-root>` 即当前插件根目录，可通过对 `workflows/` 目录执行 Glob 定位

### 第二步：解析每个 Workflow

对每个 `.md` 文件，Read 文件内容并提取：

1. **名称**：第一个 `# ` 开头的行，去掉 `# ` 前缀
2. **描述**：`**描述**` 字段后的内容
3. **版本**：`**版本**` 字段后的内容
4. **创建时间**：`**创建时间**` 字段后的内容
5. **状态**：`**状态**` 字段后的内容
6. **步骤数**：统计 `<!-- workflow:step` 出现的次数

### 第三步：格式化输出

若找到 workflow：

```markdown
# 工作流列表

| 名称 | 描述 | 步骤数 | 状态 | 版本 | 创建时间 |
|------|------|--------|------|------|----------|
| 爆款营销 | 新品上市营销策略 | 4 | 就绪 | 1.0 | 2026-06-11 |
| 竞品分析 | 竞品深度分析报告 | 3 | 就绪 | 1.0 | 2026-06-11 |

共 2 个工作流。

使用 `/supermeerkat:run-workflow` 执行工作流。
使用 `/supermeerkat:create-workflow` 创建新工作流。
```

若未找到 workflow：

```
尚未创建任何工作流。

使用 /supermeerkat:create-workflow 创建你的第一个工作流。
```

## 注意事项

- **唯一数据源**：以 `workflows/` 目录扫描为准，不依赖 `references/workflows.md` 索引文件
- 若某个 `.md` 文件无法解析（可能不是 workflow 文件），跳过并备注
- 若目录为空，给出友好提示而非报错
