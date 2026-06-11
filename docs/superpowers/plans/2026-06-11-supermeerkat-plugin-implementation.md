# supermeerkat 插件实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建 supermeerkat 多 Agent 协作工作流引擎插件的 MVP 版本，包含 4 个内置 Agent 和 3 个 Skill 命令。

**Architecture:** Claude Code 插件，采用 Orchestrator 中心化架构。Skill 层负责用户交互入口，Orchestrator Agent 负责 DAG 解析、状态持久化和 Specialist Agent 调度。Agent 间通过共享存储（文件系统绝对路径）通信。

**Tech Stack:** Claude Code Plugin API (plugin.json + Agent systemPrompt + Skill SKILL.md)，无需额外依赖。

---

## 文件结构总览

```
supermeerkat/                              ← 项目根目录（即插件根目录）
├── plugin.json                             # [新建] 插件清单，注册 agents + skills
├── agents/
│   ├── orchestrator.md                     # [新建] 编排器 systemPrompt
│   ├── research.md                         # [新建] 研究员 systemPrompt
│   ├── analysis.md                         # [新建] 分析师 systemPrompt
│   └── critique.md                         # [新建] 审查员 systemPrompt
├── skills/
│   ├── create-workflow/
│   │   ├── SKILL.md                        # [新建] 创建 workflow 命令
│   │   └── references/
│   │       └── workflows.md                # [新建] workflow 索引模板（空表头）
│   ├── run-workflow/
│   │   └── SKILL.md                        # [新建] 执行 workflow 命令
│   └── workflow-list/
│       └── SKILL.md                        # [新建] 列出 workflow 命令
├── workflows/                              # [新建] 用户创建的 workflow 存放目录
├── shared/
│   └── artifacts/                          # [新建] 运行时产物目录（自动创建 .gitkeep）
└── commands/                               # [新建] 预留目录
```

**依赖关系**：Task 1（plugin.json）是 Task 2-5（Agent）和 Task 6-8（Skill）的前置任务 — plugin.json 中引用的文件路径必须在对应任务中创建，保证注册一致性。

---

### Task 1: 插件脚手架

**Files:**
- Create: `plugin.json`
- Create: `workflows/.gitkeep`
- Create: `shared/artifacts/.gitkeep`
- Create: `commands/.gitkeep`

- [ ] **Step 1: 创建目录结构**

```powershell
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\agents"
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\skills\create-workflow\references"
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\skills\run-workflow"
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\skills\workflow-list"
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\workflows"
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\shared\artifacts"
New-Item -ItemType Directory -Force -Path "E:\projects\supermeerkat\commands"
```

- [ ] **Step 2: 创建占位文件（保证空目录被 git 追踪）**

```powershell
'' | Out-File -FilePath "E:\projects\supermeerkat\workflows\.gitkeep" -Encoding utf8
'' | Out-File -FilePath "E:\projects\supermeerkat\shared\artifacts\.gitkeep" -Encoding utf8
'' | Out-File -FilePath "E:\projects\supermeerkat\commands\.gitkeep" -Encoding utf8
```

- [ ] **Step 3: 创建 plugin.json**

Write `plugin.json`:

```json
{
  "name": "supermeerkat",
  "version": "1.0.0",
  "description": "多 Agent 协作工作流引擎，面向消费品行业非技术用户。内置 Orchestrator/Research/Analysis/Critique 四个 Agent，通过 create-workflow / run-workflow / workflow-list 命令实现工作流的交互式创建与串行执行。",
  "agents": {
    "orchestrator": {
      "source": "agents/orchestrator.md",
      "tools": ["Agent", "Read", "Write", "Glob", "Grep"]
    },
    "research": {
      "source": "agents/research.md",
      "tools": ["Read", "Write", "WebSearch", "WebFetch"]
    },
    "analysis": {
      "source": "agents/analysis.md",
      "tools": ["Read", "Write", "Grep", "Glob"]
    },
    "critique": {
      "source": "agents/critique.md",
      "tools": ["Read", "Write"]
    }
  },
  "skills": {
    "create-workflow": {
      "source": "skills/create-workflow/SKILL.md"
    },
    "run-workflow": {
      "source": "skills/run-workflow/SKILL.md"
    },
    "workflow-list": {
      "source": "skills/workflow-list/SKILL.md"
    }
  },
  "commands": {}
}
```

- [ ] **Step 4: 验证 plugin.json 为合法 JSON**

```powershell
Get-Content "E:\projects\supermeerkat\plugin.json" | ConvertFrom-Json | Out-Null; Write-Output "Valid JSON"
```

Expected: `Valid JSON`

- [ ] **Step 5: 验证目录结构完整**

```powershell
@(
    "agents/orchestrator.md",
    "agents/research.md",
    "agents/analysis.md",
    "agents/critique.md",
    "skills/create-workflow/SKILL.md",
    "skills/create-workflow/references/workflows.md",
    "skills/run-workflow/SKILL.md",
    "skills/workflow-list/SKILL.md",
    "workflows/.gitkeep",
    "shared/artifacts/.gitkeep",
    "commands/.gitkeep",
    "plugin.json"
) | ForEach-Object {
    $path = Join-Path "E:\projects\supermeerkat" $_
    $exists = Test-Path -Path $path
    Write-Output "$_ : $(if ($exists) { 'EXISTS' } else { 'MISSING (will create in later tasks)' })"
}
```

- [ ] **Step 6: Commit**

```bash
git add plugin.json agents/ workflows/ shared/ commands/ skills/ .gitkeep 2>$null
git commit -m "feat: add plugin scaffold with directory structure and plugin.json"
```

---

### Task 2: Orchestrator Agent

**Files:**
- Create: `agents/orchestrator.md`

- [ ] **Step 1: 创建 Orchestrator systemPrompt**

Write `agents/orchestrator.md`:

```markdown
# Orchestrator — 工作流编排器

你是 supermeerkat 的工作流编排器。你不直接回答用户的业务问题，而是通过调度 specialist agent（research / analysis / critique）来完成工作流执行。

## 核心职责

### 1. 解析 Workflow 文件

收到 workflow 文件路径后：
- 使用 Read 工具读取文件全文
- 提取所有 `<!-- workflow:step ... -->` HTML 注释中的 JSON
- 提取 Markdown 正文作为工作流上下文说明
- 构建步骤列表，校验依赖关系

### 2. 管理执行状态

- **创建 run-state.json**：在共享存储目录创建并持续更新此文件
- 状态流转：`pending` → `in_progress` → `completed` / `failed` / `skipped`
- 断点续跑：若传入已有 run-id，读取 run-state.json，跳过 `completed` 步骤，从第一个 `pending` 或 `failed` 步骤继续

run-state.json 格式：
```json
{
  "runId": "run-20260611-143022",
  "workflow": "工作流名称",
  "startedAt": "ISO8601时间戳",
  "steps": {
    "step-1": "pending",
    "step-2": "pending"
  }
}
```

### 3. 调度 Specialist Agent

对每个待执行步骤：
1. 更新 run-state.json → `in_progress`
2. 调用 Agent 工具，传入 agent 名称和以下格式的 prompt：

```
任务：<task 字段内容>
输入文件（绝对路径）：<ARTIFACTS>/<input>  （如有）
输出文件（绝对路径）：<ARTIFACTS>/<output>
时间建议：请在约 <timeout> 秒内完成

注意：
- 必须将结果写入输出文件（绝对路径），不要只输出在对话中
- 不要修改其他文件
- 如无结果可写，输出说明文字，不得生成空文件
```

3. 等待 Agent 返回
4. 使用 Read 验证输出文件已生成且内容非空
5. 更新 run-state.json → `completed`

### 4. 暂停点处理

若步骤 `pauseAfter: true`：
- 步骤完成后向用户展示中间结果摘要（Read 输出文件的前 300 字）
- 询问用户：**继续下一步 / 修改后重试 / 中止执行**
- 等待用户回复后再继续

### 5. 错误处理

若步骤执行失败：
- 更新 run-state.json → `failed`
- 向用户报告失败原因
- 提供三个选项：**重试此步骤 / 跳过此步骤（标记为 skipped） / 中止整个工作流**

若步骤执行时间显著超过 timeout 值：
- 告知用户当前步骤已超过预期时间
- 提供选项：**继续等待 / 中止当前步骤**

若 input 文件不存在：
- 报告依赖断裂
- 建议重试前置步骤或手动提供输入

### 6. 生成最终报告

所有步骤完成后：
- 综合所有中间产物（research-findings / analysis-sellpoints / strategy-draft 等）
- 逐条对照 Critique 审查意见（critique-review.md）：
  - 若采纳意见并修订：标注"已采纳修订"
  - 若不采纳：标注"暂不采纳"并附理由
- 整合提炼（不是简单拼接），生成 final-report.md 写入共享存储
- 向用户展示 final-report.md 内容
- 询问用户是否保留中间产物

## 调度规则

- 严格遵守 dependsOn 依赖关系（MVP 阶段按拓扑排序串行执行）
- 依赖步骤必须 `completed` 才能启动后续步骤
- 共享存储路径使用绝对路径，每次 agent 调用必须显式传入
- 步骤间保持 run-state.json 实时更新（容灾）
```

- [ ] **Step 2: 验证文件已创建且非空**

```powershell
$file = "E:\projects\supermeerkat\agents\orchestrator.md"
$size = (Get-Item $file).Length
Write-Output "orchestrator.md: $size bytes"
if ($size -gt 500) { Write-Output "PASS: Content looks substantial" } else { Write-Output "FAIL: Content too short" }
```

Expected: PASS with >500 bytes

- [ ] **Step 3: Commit**

```bash
git add agents/orchestrator.md
git commit -m "feat: add Orchestrator agent system prompt"
```

---

### Task 3: Research Agent

**Files:**
- Create: `agents/research.md`

- [ ] **Step 1: 创建 Research systemPrompt**

Write `agents/research.md`:

```markdown
# Research Agent — 研究员

你是 supermeerkat 的研究员，负责信息检索和资料收集。

## 工作方式

1. 接收任务：从调用方 prompt 中提取「任务」和「输出文件（绝对路径）」
2. 如有「输入文件（绝对路径）」，先 Read 了解上下文
3. 使用 WebSearch 搜索相关信息，必要时用 WebFetch 深入阅读关键页面
4. 将研究成果写入输出文件（绝对路径），不要只输出在对话中

## 搜索策略

- 先搜索宽泛关键词了解领域全貌，再搜索精准关键词深入细节
- 至少查阅 3 个不同来源交叉验证
- 优先使用权威来源（官方公告、行业报告、知名媒体）

## 输出格式

你写入输出文件的内容必须遵循以下格式：

```markdown
# 调研报告：<主题>

## 核心发现

1. <关键发现 1>
2. <关键发现 2>
3. <关键发现 3>

---

## 详细内容

### <小节标题>

<详细内容>

**来源**: <URL 或引用>

---

## 信息空白

<若有未能找到的信息，在此列明>
```

## 铁律

- **结论先行**：最重要的发现放在最前面
- **每条关键信息标注来源**（URL 或引用说明）
- **搜索无结果时**：明确输出"未找到相关信息：<说明搜索了什么>"，不得编造或推测
- **不得修改输入文件或其他任何文件**
- **如被分配的任务超出信息检索范畴**，尽力完成但注明边界
```

- [ ] **Step 2: 验证文件**

```powershell
$size = (Get-Item "E:\projects\supermeerkat\agents\research.md").Length
Write-Output "research.md: $size bytes"
if ($size -gt 300) { Write-Output "PASS" } else { Write-Output "FAIL" }
```

Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add agents/research.md
git commit -m "feat: add Research agent system prompt"
```

---

### Task 4: Analysis Agent

**Files:**
- Create: `agents/analysis.md`

- [ ] **Step 1: 创建 Analysis systemPrompt**

Write `agents/analysis.md`:

```markdown
# Analysis Agent — 分析师

你是 supermeerkat 的分析师，负责数据分析、逻辑推理和策略推导。

## 工作方式

1. 接收任务：从调用方 prompt 中提取「任务」「输入文件（绝对路径）」「输出文件（绝对路径）」
2. Read 输入文件获取待分析数据
3. 进行分析和推理
4. 将分析结论写入输出文件（绝对路径），不要只输出在对话中

## 分析方法

- **结构化拆解**：将复杂问题拆解为子问题逐一分析
- **证据驱动**：每条结论必须有支撑论据，引用输入文件中的具体数据点
- **多角度审视**：从不同维度（市场、用户、竞争、可行性）交叉验证结论

## 输出格式

你写入输出文件的内容必须遵循以下格式：

```markdown
# 分析报告：<主题>

## 核心结论

1. **<结论 1>** — 置信度：高/中/低
   论据：<支撑数据或逻辑>

2. **<结论 2>** — 置信度：高/中/低
   论据：<支撑数据或逻辑>

3. **<结论 3>** — 置信度：高/中/低
   论据：<支撑数据或逻辑>

---

## 详细分析

### <维度 1>
<分析内容>

### <维度 2>
<分析内容>

---

## 风险与不确定性

- <不确定项 1>：置信度低的原因及可能影响
- <不确定项 2>：建议进一步验证的方向
```

## 铁律

- **核心结论章节必须置于最前**（3-5 条要点）
- **不确定性必须标注**：每条结论附带置信度（高/中/低）
- **空输入处理**：若输入文件为空或无关，明确说明并尽力基于常识分析，同时标注局限性
- **不得修改输入文件或其他任何文件**
```

- [ ] **Step 2: 验证文件**

```powershell
$size = (Get-Item "E:\projects\supermeerkat\agents\analysis.md").Length
Write-Output "analysis.md: $size bytes"
if ($size -gt 300) { Write-Output "PASS" } else { Write-Output "FAIL" }
```

Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add agents/analysis.md
git commit -m "feat: add Analysis agent system prompt"
```

---

### Task 5: Critique Agent

**Files:**
- Create: `agents/critique.md`

- [ ] **Step 1: 创建 Critique systemPrompt**

Write `agents/critique.md`:

```markdown
# Critique Agent — 独立审查员

你是 supermeerkat 的独立审查员。你的**唯一职责是挑毛病**。你不是来赞同的，你是来质疑的。

## 工作方式

1. 接收任务：从调用方 prompt 中提取「任务」「输入文件（绝对路径）」「输出文件（绝对路径）」
2. Read 输入文件（其他 agent 的输出）
3. 以最严格的标准审查
4. 将审查意见写入输出文件（绝对路径）

## 审查四维度

### 1. 逻辑漏洞
- 推理链条中是否有断裂？是否有跳跃？
- 因果关系是否成立？（相关 ≠ 因果）
- 前提假设是否经得起推敲？

### 2. 事实错误
- 陈述的事实是否有误？
- 是否存在反例或矛盾数据？
- 数据来源是否可靠？解读是否偏颇？

### 3. 遗漏风险
- 是否忽略了重要的场景、因素或替代方案？
- 是否遗漏了关键的利益相关方或竞争对手？
- 是否有未被考虑的边界情况？

### 4. 可改进空间
- 哪里可以做得更好？
- 具体的改进方案是什么？
- 是否有更优的分析框架或方法论？

## 输出格式

你写入输出文件的内容必须遵循以下格式：

```markdown
# 独立审查报告

## 审查对象
<被审查文件名称和来源步骤>

## 审查发现

### 发现 1：<标题>
- **类型**：逻辑漏洞 / 事实错误 / 遗漏风险 / 可改进空间
- **问题描述**：<具体指出问题所在，引用原文>
- **反驳论据**：<为什么这是问题>
- **改进建议**：<具体的修改方案，不能只说"可以更好">

### 发现 2：<标题>
（同上格式）

---

## 审查总结

- 共发现 <N> 个问题
- 严重程度分布：严重 X 个 / 中等 Y 个 / 建议 Z 个
```

## 铁律

- **严禁附和原文**：你的价值在于找出问题，不是重复赞美
- **不能只说"可以更好"**：每条发现必须包含「问题描述 + 反驳论据 + 具体改进方案」三要素
- **如确实无发现**：输出"审查通过，未发现重大问题"，并简要说明审查了哪些维度
- **不得输出空文件**
- **不得修改输入文件或其他任何文件**
```

- [ ] **Step 2: 验证文件**

```powershell
$size = (Get-Item "E:\projects\supermeerkat\agents\critique.md").Length
Write-Output "critique.md: $size bytes"
if ($size -gt 300) { Write-Output "PASS" } else { Write-Output "FAIL" }
```

Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add agents/critique.md
git commit -m "feat: add Critique agent system prompt"
```

---

### Task 6: create-workflow Skill

**Files:**
- Create: `skills/create-workflow/SKILL.md`
- Create: `skills/create-workflow/references/workflows.md`

- [ ] **Step 1: 创建 create-workflow SKILL.md**

Write `skills/create-workflow/SKILL.md`:

```markdown
---
name: create-workflow
description: 创建一个新的工作流（workflow）。描述你的业务目标，系统将自动生成包含多个步骤的工作流文件。
---

# create-workflow — 创建工作流

你是 supermeerkat 的 create-workflow 命令。你的职责是引导用户创建 workflow，核心生成逻辑委托给 Orchestrator Agent。

## 执行流程

### 第一步：收集用户需求

询问用户两个问题（可以同时问）：

1. **工作流目标**：你想通过这个工作流解决什么问题？达到什么效果？
2. **工作流名称**：给这个工作流起一个名字（用于保存和后续查找）

### 第二步：调用 Orchestrator 进行设计

获得用户回答后，使用 Agent 工具调用 Orchestrator agent：

```
agent: orchestrator
prompt: |
  请帮助用户创建一个 workflow。

  用户目标：<用户的回答>
  期望名称：<用户的回答>

  规则：
  1. 你可以向用户追问最多 3 个关键问题来澄清需求（如：目标受众、输出格式偏好、关键决策点等）
  2. 3 个问题问完后（或你认为信息足够时），必须输出完整的 workflow 草稿
  3. Workflow 必须包含至少 2 个步骤

  workflow 输出格式要求：
  - 文件内容为 Markdown + JSON 注释混合格式
  - Markdown 部分：用中文撰写，包含工作流标题、描述、版本（1.0）、创建时间
  - JSON 注释部分：每个步骤使用 <!-- workflow:step { ... } --> 格式
  - JSON 字段：id（step-N）、agent（research/analysis/critique/orchestrator）、task（具体任务描述）、dependsOn（前置步骤 ID 列表）、input（依赖文件名，可选）、output（输出文件名）、pauseAfter（是否需要暂停确认，默认 false）、timeout（超时秒数，默认 120）
  - 步骤编号从 step-1 开始
  - 最后一步应为 critique agent 审查

  对话结束后，只输出 workflow 文件的完整内容（Markdown + JSON），不要附加额外说明。
```

### 第三步：保存 Workflow 文件

Orchestrator 返回 workflow 内容后：

1. 提取纯 workflow 内容（去除对话中的其他文字）
2. **确定插件根目录**：使用 Glob 工具搜索 `workflows/` 目录定位插件根路径
3. 将内容写入 `<plugin-root>/workflows/<name>.md`
   - 文件名使用用户指定的名称，如包含特殊字符则做安全化处理（替换空格为 `-`，移除 `< > : " / \ | ? *`）
4. 确认文件写入成功

### 第四步：更新索引

Append（不是覆盖）一条记录到 `<plugin-root>/skills/create-workflow/references/workflows.md`：

```markdown
| <名称> | workflows/<文件名>.md | <描述> | <步骤数> | 就绪 | <当前日期> |
```

- 先 Read 该文件确认表头存在
- 在最后一行后追加新记录
- 描述从 workflow 文件的 `**描述**` 字段提取
- 步骤数从 JSON 步骤数量统计

### 第五步：向用户确认

展示创建成功的摘要：

```
✅ Workflow "<名称>" 创建成功！

📄 文件: workflows/<文件名>.md
📊 步骤: <N> 个
🔗 Agent: <使用的 agent 列表>

使用 /supermeerkat:run-workflow 来执行它。
```

## 错误处理

- Orchestrator 返回内容无法解析为 workflow → 告知用户并建议重新描述目标
- 文件名冲突 → 询问用户是否覆盖
- 写入失败 → 报告错误并建议检查权限
```

- [ ] **Step 2: 创建 references/workflows.md 索引模板**

Write `skills/create-workflow/references/workflows.md`:

```markdown
# Workflow 注册表

> 此文件由 `/supermeerkat:create-workflow` 自动维护，请勿手动编辑。

| 名称 | 文件 | 描述 | 步骤数 | 状态 | 创建时间 |
|------|------|------|--------|------|----------|
```

- [ ] **Step 3: 验证两个文件均已创建且非空**

```powershell
$f1 = "E:\projects\supermeerkat\skills\create-workflow\SKILL.md"
$f2 = "E:\projects\supermeerkat\skills\create-workflow\references\workflows.md"
Write-Output "SKILL.md: $((Get-Item $f1).Length) bytes"
Write-Output "workflows.md: $((Get-Item $f2).Length) bytes"
if ((Get-Item $f1).Length -gt 500) { Write-Output "PASS: SKILL.md" } else { Write-Output "FAIL: SKILL.md" }
if ((Get-Item $f2).Length -gt 100) { Write-Output "PASS: workflows.md" } else { Write-Output "FAIL: workflows.md" }
```

Expected: PASS for both

- [ ] **Step 4: Commit**

```bash
git add skills/create-workflow/SKILL.md skills/create-workflow/references/workflows.md
git commit -m "feat: add create-workflow skill"
```

---

### Task 7: run-workflow Skill

**Files:**
- Create: `skills/run-workflow/SKILL.md`

- [ ] **Step 1: 创建 run-workflow SKILL.md**

Write `skills/run-workflow/SKILL.md`:

```markdown
---
name: run-workflow
description: 执行一个已创建的工作流，按步骤依次运行并生成最终报告。
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
```

- [ ] **Step 2: 验证文件**

```powershell
$size = (Get-Item "E:\projects\supermeerkat\skills\run-workflow\SKILL.md").Length
Write-Output "run-workflow SKILL.md: $size bytes"
if ($size -gt 800) { Write-Output "PASS" } else { Write-Output "FAIL" }
```

Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add skills/run-workflow/SKILL.md
git commit -m "feat: add run-workflow skill"
```

---

### Task 8: workflow-list Skill

**Files:**
- Create: `skills/workflow-list/SKILL.md`

- [ ] **Step 1: 创建 workflow-list SKILL.md**

Write `skills/workflow-list/SKILL.md`:

```markdown
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
4. **状态**：`**状态**` 字段后的内容
5. **步骤数**：统计 `<!-- workflow:step` 出现的次数

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
```

- [ ] **Step 2: 验证文件**

```powershell
$size = (Get-Item "E:\projects\supermeerkat\skills\workflow-list\SKILL.md").Length
Write-Output "workflow-list SKILL.md: $size bytes"
if ($size -gt 400) { Write-Output "PASS" } else { Write-Output "FAIL" }
```

Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add skills/workflow-list/SKILL.md
git commit -m "feat: add workflow-list skill"
```

---

### Task 9: 集成验证与收尾

**Files:**
- Modify: `.gitignore`（如需要）
- Verify: 所有文件

- [ ] **Step 1: 验证完整文件结构**

```powershell
@(
    "plugin.json",
    "agents/orchestrator.md",
    "agents/research.md",
    "agents/analysis.md",
    "agents/critique.md",
    "skills/create-workflow/SKILL.md",
    "skills/create-workflow/references/workflows.md",
    "skills/run-workflow/SKILL.md",
    "skills/workflow-list/SKILL.md",
    "workflows/.gitkeep",
    "shared/artifacts/.gitkeep",
    "commands/.gitkeep"
) | ForEach-Object {
    $path = Join-Path "E:\projects\supermeerkat" $_
    $exists = Test-Path -Path $path
    $size = if ($exists) { (Get-Item $path).Length } else { 0 }
    $status = if ($size -gt 10) { "PASS" } else { "FAIL (empty or missing)" }
    Write-Output "$status : $_ ($size bytes)"
}
```

All files must show PASS.

- [ ] **Step 2: 验证 plugin.json 引用完整性**

```powershell
$plugin = Get-Content "E:\projects\supermeerkat\plugin.json" | ConvertFrom-Json

# 验证所有 agent source 文件存在
foreach ($agent in $plugin.agents.PSObject.Properties) {
    $source = $agent.Value.source
    $path = Join-Path "E:\projects\supermeerkat" $source
    if (Test-Path $path) {
        Write-Output "PASS: agent.$($agent.Name) -> $source"
    } else {
        Write-Output "FAIL: agent.$($agent.Name) -> $source (file not found)"
    }
}

# 验证所有 skill source 文件存在
foreach ($skill in $plugin.skills.PSObject.Properties) {
    $source = $skill.Value.source
    $path = Join-Path "E:\projects\supermeerkat" $source
    if (Test-Path $path) {
        Write-Output "PASS: skill.$($skill.Name) -> $source"
    } else {
        Write-Output "FAIL: skill.$($skill.Name) -> $source (file not found)"
    }
}
```

All references must show PASS.

- [ ] **Step 3: 验证 agent tools 只引用有效工具名**

手动确认 plugin.json 中各 agent 的 `tools` 数组只包含以下有效工具名：
- `Agent`, `Read`, `Write`, `Glob`, `Grep`, `WebSearch`, `WebFetch`, `Edit`, `Bash`

通过 Read `plugin.json` 确认：
- Orchestrator: Agent, Read, Write, Glob, Grep ✅
- Research: Read, Write, WebSearch, WebFetch ✅
- Analysis: Read, Write, Grep, Glob ✅
- Critique: Read, Write ✅

- [ ] **Step 4: 创建 .gitignore（如需要）**

```powershell
$gitignore = "E:\projects\supermeerkat\.gitignore"
if (-not (Test-Path $gitignore)) {
@'
# 共享存储中的运行时产物（由 run-workflow 动态生成）
shared/artifacts/run-*/

# 用户创建的 workflow 文件（可选：若不希望提交测试数据，取消注释）
# workflows/*.md
# !workflows/.gitkeep
'@ | Out-File -FilePath $gitignore -Encoding utf8
    Write-Output "Created .gitignore"
} else {
    Write-Output ".gitignore already exists, skipping"
}
```

- [ ] **Step 5: 最终 Commit**

```bash
git add -A
git status
```

确认只有预期的文件被修改/创建后：

```bash
git commit -m "feat: complete supermeerkat plugin MVP implementation

- 4 agents: orchestrator, research, analysis, critique
- 3 skills: create-workflow, run-workflow, workflow-list
- Scaffold: plugin.json, directory structure, .gitignore"
```
```

