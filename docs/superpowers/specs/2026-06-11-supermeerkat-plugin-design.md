# supermeerkat 插件设计规格文档

> **版本**: 2.0
> **日期**: 2026-06-11
> **状态**: MVP 设计已确认
> **变更说明**: 修复 plugin.json 不完整、worktree 路径冲突、状态持久化缺失等问题；补充 Critique 审查机制、timeout 字段、非技术用户说明等

---

## 一、项目概述

### 1.1 背景与目标

用户在使用 Claude CLI 等 AI 编辑器时，通过安装 `supermeerkat` 插件，可以创建多个 Agent 并自动构建 Workflow 工作流。执行工作流后，获得对应的结果。

MVP 目标：跑通核心闭环——内置的 4 个 agent（Orchestrator、Research、Analysis、Critique）协同工作，实现 workflow 创建和执行。

> **用户须知**：目标用户为非技术背景的消费品行业从业者（老板、项目经理、产品经理、销售等）。**用户永远不需要手写或编辑 workflow 文件**，所有 workflow 均通过 `/supermeerkat:create-workflow` 命令交互生成。

### 1.2 目标用户

消费品行业的用户：老板、项目经理、产品经理、销售等。

### 1.3 MVP 范围

| 模块 | 是否纳入 MVP | 说明 |
|------|-------------|------|
| 内置 Agent（4 个） | ✅ | Orchestrator、Research、Analysis、Critique |
| 自定义 Agent 创建 | ❌ | 延后 |
| Workflow 创建 | ✅ | `/supermeerkat:create-workflow` |
| Workflow 执行 | ✅ | `/supermeerkat:run-workflow` |
| Workflow 列表查询 | ✅ | `/supermeerkat:workflow-list` |
| 自定义 Agent 列表 | ❌ | 延后 |

> **MVP 执行模式说明**：MVP 阶段仅实现**串行执行**。无依赖步骤的并行调度作为后续迭代方向（见第十章），不在本版本实现范围内。

---

## 二、整体架构

### 2.1 架构模式：Orchestrator 中心化

```
用户 → /supermeerkat:xxx 命令 → Skill → Orchestrator Agent
                                              ↓
                              ┌───────────────┼───────────────┐
                              ↓               ↓               ↓
                        Research Agent  Analysis Agent  Critique Agent
                              ↓               ↓               ↓
                              └───────────────┴───────→ 共享存储
                                                     shared/artifacts/<run-id>/
```

> **注意**：上图展示逻辑上的协作关系。MVP 阶段三个 Specialist Agent 按串行顺序依次调度，不并行。

### 2.2 核心设计决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| Agent 调度方式 | Claude Code 原生 `Agent` 工具 | 最自然复用现有调度模型；符合 Superpowers 的实践 |
| Agent 间通信 | 文件系统（`shared/artifacts/<run-id>/`，绝对路径） | 简单可靠、可调试、对用户透明、MVP 优先 |
| Workflow 格式 | Markdown + JSON 注释混合 | 人可读 + 机器可精确解析；非开发人员也能通过命令生成 |
| 执行交互模式 | 混合模式（自动 + 关键点暂停） | 平衡自动化与用户控制 |
| 架构模型 | Orchestrator 中心化（方案 A） | 职责清晰、集中调度、易于调试 |
| Specialist Agent 隔离 | **不使用** `isolation: worktree` | 避免文件路径跨 worktree 冲突；共享存储采用绝对路径即可保证隔离 |
| 步骤状态持久化 | `run-state.json` 写入共享存储 | Orchestrator 上下文窗口消失后状态不丢失，支持断点续跑 |

### 2.3 设计理念

1. **Orchestrator 负责"元认知"** — 收到用户输入后，不是直接回答，而是通过 brainstorming 进行需求分析，设计方案，写规格文档
2. **共享存储是"通信总线"** — 各 Agent 之间不直接通话，通过写入/读取共享存储（绝对路径）传递上下文
3. **Critique Agent 是质量保障核心** — 独立审查其他 agent 的输出，形成"对抗性检验"；审查意见由 Orchestrator 在最终报告中逐条回应（已采纳修订/暂不采纳并附理由）。MVP 阶段 Critique 输出审查意见后流程即结束，不做"修订→再审查"的迭代循环
4. **Harness 工程理念** — 遵循 Superpowers 的流程化设计：brainstorming → spec → plan → implement → review

---

## 三、插件目录结构

```
supermeerkat/
├── plugin.json                    # 插件清单（含 agents + skills 完整注册）
├── agents/                        # 内置 agent 的 systemPrompt 定义
│   ├── orchestrator.md
│   ├── research.md
│   ├── analysis.md
│   └── critique.md
├── skills/                        # 斜杠命令对应的 skill
│   ├── create-workflow/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── workflows.md       # workflow 索引（辅助缓存，非唯一数据源）
│   ├── run-workflow/
│   │   └── SKILL.md
│   └── workflow-list/
│       └── SKILL.md
├── workflows/                     # 用户创建的 workflow 文件（由命令生成，用户无需手写）
│   └── <workflow-name>.md
├── commands/                      # （预留）如需要独立命令定义
└── shared/                        # 共享存储
    └── artifacts/
        └── <run-id>/              # 每次执行一个隔离目录
            ├── run-state.json     # 步骤状态持久化（新增）
            ├── research-findings.md
            ├── analysis-sellpoints.md
            ├── strategy-draft.md
            ├── critique-review.md
            └── final-report.md
```

---

## 四、Workflow 文件格式

### 4.1 格式说明

Workflow 文件是 Markdown 文档，在 Markdown 正文中嵌入 JSON 注释块来定义结构化步骤。**此文件由 `/supermeerkat:create-workflow` 命令自动生成，用户无需手动编写。**

### 4.2 示例

```markdown
# 爆款营销工作流

**描述**: 为新品上市生成爆款营销策略报告
**版本**: 1.0
**创建时间**: 2026-06-11
**状态**: 就绪

---

## 执行计划

本工作流分为四个阶段：竞品调研 → 卖点分析 → 策略制定 → Critique 审查。

---

## 步骤

### 步骤 1：竞品调研

<!-- workflow:step
{
  "id": "step-1",
  "agent": "research",
  "task": "搜索并收集主要竞品的营销策略、定价、目标人群信息",
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
  "task": "基于竞品调研结果，提炼产品的核心卖点和差异化定位",
  "dependsOn": ["step-1"],
  "input": "research-findings.md",
  "output": "analysis-sellpoints.md",
  "pauseAfter": true,
  "timeout": 60
}
-->

### 步骤 3：策略制定

<!-- workflow:step
{
  "id": "step-3",
  "agent": "analysis",
  "task": "根据卖点分析制定完整的营销策略方案",
  "dependsOn": ["step-2"],
  "input": "analysis-sellpoints.md",
  "output": "strategy-draft.md",
  "pauseAfter": true,
  "timeout": 60
}
-->

### 步骤 4：Critique 审查

<!-- workflow:step
{
  "id": "step-4",
  "agent": "critique",
  "task": "审查营销策略草案，找出漏洞、风险和可改进之处",
  "dependsOn": ["step-3"],
  "input": "strategy-draft.md",
  "output": "critique-review.md",
  "timeout": 60
}
-->
```

### 4.3 JSON 步骤字段规范

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✅ | 步骤唯一标识，跨步骤 `dependsOn` 引用 |
| `agent` | string | ✅ | 调用的 agent 名称：`research` / `analysis` / `critique` / `orchestrator` |
| `task` | string | ✅ | 传给 agent 的具体任务描述 |
| `dependsOn` | string[] | ✅ | 前置依赖步骤的 ID 列表，空数组表示无依赖 |
| `input` | string | ❌ | 从共享存储读取的文件名（相对于 run-id 目录） |
| `output` | string | ✅ | 输出到共享存储的文件名 |
| `pauseAfter` | boolean | ❌ | 默认 `false`。该步骤完成后是否暂停等待用户确认 |
| `timeout` | number | ❌ | 步骤超时时间（秒），默认 `120`。此为**协作式软超时**：Orchestrator 将时限写入 specialist agent 的 prompt 中，agent 自行控制工作范围以接近此时限。若 agent 执行时间明显超出预期，Orchestrator 在 run-state.json 中标注并告知用户，由用户决定继续等待或中止 |

### 4.4 解析规则

1. `run-workflow` skill 读取 workflow 文件
2. 提取所有 `<!-- workflow:step ... -->` 中的 JSON 构建 DAG
3. 提取 Markdown 正文作为给 Orchestrator 的上下文说明
4. 校验：所有 `dependsOn` 引用的 ID 存在、无循环依赖、agent 名称有效
5. 校验失败时拒绝执行，提示用户通过 `create-workflow` 重新生成

---

## 五、Agent 设计

### 5.1 plugin.json 完整注册

```json
{
  "name": "supermeerkat",
  "version": "1.0.0",
  "description": "多 Agent 协作工作流引擎，面向消费品行业非技术用户",
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

> **说明**：Specialist Agent（Research、Analysis、Critique）不使用 `isolation: worktree`，避免文件路径跨 worktree 失效。所有共享存储操作均使用绝对路径传递，由 Orchestrator 在调度时注入。

### 5.2 Orchestrator Agent

**职责**: 分解任务、制定 workflow、解析和执行 DAG、调度 agent、持久化步骤状态、汇总结果、处理暂停/恢复/错误。

**systemPrompt 核心要点**:

```
你是 supermeerkat 的工作流编排器。你的职责：

1. 解析 workflow 文件，提取 Markdown + JSON 混合格式的步骤定义
2. 构建步骤 DAG，按依赖关系确定执行顺序（MVP 阶段：串行执行）
3. 生成唯一的 run-id（格式: run-YYYYMMDD-HHmmss），创建共享存储目录
4. 在共享存储中创建并持续更新 run-state.json，记录每个步骤状态
5. 若为断点续跑：读取已有 run-state.json，跳过 completed 步骤，从第一个 pending/failed 步骤继续
6. 使用 Agent 工具派发子任务给 specialist agent，传入绝对路径
7. 每个步骤完成后验证输出文件已生成且非空
8. 在 pauseAfter=true 的步骤完成后，向用户展示中间结果摘要并等待确认
9. 步骤失败或超时时提供三个选项：重试 / 跳过 / 中止
10. 所有步骤完成后，综合所有中间产物和 Critique 审查意见，生成最终报告

调度规则：
- 严格遵守 dependsOn 依赖关系
- MVP 阶段按拓扑排序串行执行，不并行
- 每次 agent 调用必须传入绝对路径（input/output）
- 依赖步骤的 input 文件必须已存在才能调度下一步
- 每步开始前将状态写入 run-state.json（in_progress），完成后更新（completed/failed/skipped）

最终报告生成规则：
- 综合 research、analysis、strategy 产物形成主报告
- 将 Critique 的审查意见逐条对照，明确标注"已采纳修订"或"暂不采纳（附理由）"
- 不得将各文件内容简单拼接，必须整合提炼
```

### 5.3 Research Agent

**职责**: 信息检索、资料收集。

```
你是研究员。工作方式：

1. 接收具体的检索任务（task）和绝对路径的 output 文件路径
2. 如有 input 文件（绝对路径），先读取了解上下文
3. 使用 WebSearch 和 WebFetch 工具搜索和收集信息
4. 将研究成果写出到 output 指定的绝对路径文件

输出格式要求：
- 结构化 Markdown
- 结论先行，再展开细节
- 每条关键信息标注来源（URL 或引用）
- 如搜索无结果，明确说明"未找到相关信息"，不得编造
```

### 5.4 Analysis Agent

**职责**: 数据分析、逻辑推理、策略推导。

```
你是分析师。工作方式：

1. 接收具体的分析任务（task）和绝对路径的文件路径
2. 读取 input 指定的绝对路径文件获取待分析数据
3. 进行数据分析、逻辑推理、策略推导
4. 将分析结论写出到 output 指定的绝对路径文件

输出格式要求：
- 结构化 Markdown
- "核心结论"章节置于最前（3-5 条要点）
- 每条结论附带支撑论据
- 如涉及不确定性，明确标注置信度（高/中/低）
```

### 5.5 Critique Agent

**职责**: 独立审查、漏洞发现、反驳验证。

```
你是独立审查员。你的唯一职责是挑毛病。工作方式：

1. 读取 input 指定的绝对路径文件（其他 agent 的输出）
2. 严格审查，严禁附和原文
3. 将审查意见写出到 output 指定的绝对路径文件

审查四维度：
- 逻辑漏洞：推理链条中是否有断裂？因果关系是否成立？
- 事实错误：陈述的事实是否有误？是否有反例？
- 遗漏风险：是否忽略了重要的场景、因素或替代方案？
- 可改进空间：哪里可以做得更好？具体建议是什么？

输出格式要求：
- 每个发现独立成段
- 每条必须包含：发现 + 反驳论据 + 改进建议
- 不能只说"可以更好"，必须给出具体方案
- 如无发现，输出"审查通过，未发现重大问题"，不得输出空文件
```

---

## 六、Skill 设计

### 6.1 create-workflow Skill

**命令**: `/supermeerkat:create-workflow`

**职责**: 作为入口，引导用户创建 workflow，核心生成逻辑委托给 Orchestrator Agent。

**对话约束**: Orchestrator brainstorming 阶段**最多询问用户 3 个关键问题**，然后输出 workflow 草稿让用户确认，避免无限追问。

**流程**:

```
用户运行 /supermeerkat:create-workflow
        ↓
Skill 询问用户：workflow 的目标和期望名称
        ↓
Skill 调用 Orchestrator Agent
  prompt: "请帮助用户创建一个 workflow。
           目标: {用户输入的目标}
           名称: {用户输入的名称}
           
           规则：最多向用户追问 3 个关键问题，
           然后输出完整 workflow 草稿让用户确认。"
        ↓
Orchestrator 通过 brainstorming 与用户对话（最多 3 问）：
  - 理解目标 → 拆解步骤 → 分配 agent → 确认关键决策点
        ↓
Orchestrator 产出完整的 Markdown + JSON 混合内容
        ↓
Skill 将内容写入 workflows/<name>.md
        ↓
Skill 追加到 references/workflows.md 索引表（辅助缓存）
        ↓
向用户确认创建成功，展示 workflow 摘要
```

### 6.2 run-workflow Skill

**命令**: `/supermeerkat:run-workflow`

**职责**: 作为入口，让用户选择并执行 workflow，执行逻辑委托给 Orchestrator Agent。

**流程**:

```
用户运行 /supermeerkat:run-workflow
        ↓
Skill 扫描 workflows/ 目录展示可用 workflow 列表
（以目录扫描为唯一数据源，保证与实际文件一致）
        ↓
用户选择（或直接指定 workflow 名称）
        ↓
Skill 扫描 shared/artifacts/ 下是否存在该 workflow 的历史 run-id 目录
  → 若存在且 run-state.json 中有未完成步骤 → 询问用户：
    "检测到未完成的执行 [run-20260611-143022]，是否从断点继续？"
    选项：[从断点继续 / 重新执行 / 查看详情后决定]
  → 若选择"从断点继续"：复用已有 run-id，读取 run-state.json 获取已完成步骤
  → 若选择"重新执行"或不存在断点：生成新 run-id
        ↓
Skill 生成或复用 run-id = "run-YYYYMMDD-HHmmss"
Skill 确定共享存储绝对路径 = "<plugin-root>/shared/artifacts/<run-id>/"
        ↓
Skill 调用 Orchestrator Agent
  prompt: "请执行 workflow 文件 <plugin-root>/workflows/<name>.md
           run-id: <run-id>
           共享存储绝对路径: <plugin-root>/shared/artifacts/<run-id>/
           断点续跑: <true/false>（若为 true，Orchestrator 读取已有 run-state.json，
           跳过已完成的步骤，从第一个 pending/failed 步骤继续）"
        ↓
Orchestrator 解析 DAG → 读取或创建 run-state.json → 按串行顺序执行步骤
  (详见第八章数据流)
        ↓
执行完成（成功/失败/中止），Orchestrator 返回最终报告
```

### 6.3 workflow-list Skill

**命令**: `/supermeerkat:workflow-list`

**职责**: 查询并展示所有已保存的 workflow。

**流程**:

```
用户运行 /supermeerkat:workflow-list
        ↓
Skill 扫描 workflows/ 目录（动态读取，不依赖索引文件）
        ↓
Skill 格式化输出表格：
  | 名称 | 描述 | 步骤数 | 状态 | 创建时间 |
```

> **说明**：`workflow-list` 和 `run-workflow` 均以扫描 `workflows/` 目录为唯一数据源，避免索引文件与实际文件不一致的问题。`references/workflows.md` 仅由 `create-workflow` 写入维护，作为轻量索引缓存（不参与 workflow 读取链路）。

### 6.4 references/workflows.md 索引文件格式

```markdown
# Workflow 注册表

| 名称 | 文件 | 描述 | 步骤数 | 状态 | 创建时间 |
|------|------|------|--------|------|----------|
| 爆款营销 | workflows/爆款营销.md | 新品上市营销策略 | 4 | 就绪 | 2026-06-11 |
```

---

## 七、错误处理

### 7.1 错误分类与处理策略

| 错误场景 | 检测方式 | 处理策略 |
|----------|----------|----------|
| Agent 调用失败 | Agent 工具返回错误 | Orchestrator 捕获 → 报告原因 → 提供选项：**重试** / **跳过** / **中止** |
| 步骤超时 | timeout 字段定义的秒数（协作式软超时） | Orchestrator 检测到执行时间显著超限 → 告知用户 → 提供选项：继续等待 / 中止当前步骤 |
| input 文件不存在 | Orchestrator 读取前验证 | 报告依赖断裂 → 建议重试前一步或手动提供输入 |
| output 文件未生成 | Orchestrator 步骤后验证 | 报告 agent 可能内部失败 → 提供重试/跳过/中止 |
| Workflow JSON 解析失败 | run-workflow 入口校验 | 报错并指出具体行号 → 建议用户重新 create-workflow |
| 循环依赖 | create-workflow 保存时校验 | 拓扑排序检测 → 拒绝保存 → 提示用户修正 |
| 依赖的步骤 ID 不存在 | create-workflow 保存时校验 | 报告哪个 dependsOn 引用无效 → 拒绝保存 |
| 文件写入冲突 | 每次执行独立 run-id 目录 | 不会冲突；如目录意外已存在则追加序号后缀 |
| 用户中断（Ctrl+C） | 对话终止 | 已完成步骤产物保留在共享存储 → 下次执行时询问是否从断点继续 |

### 7.2 中间产物管理

- 每次 workflow 执行结束（成功/失败/中止），Orchestrator 确认是否保留中间产物
- 默认保留，便于调试和断点续跑
- 后续版本可添加 `/supermeerkat:clean` 命令批量清理

---

## 八、完整数据流（端到端示例）

```
用户: /supermeerkat:run-workflow

1. Skill 入口
   扫描 workflows/ 目录 → 展示列表
   用户选择 "爆款营销"

2. Skill 准备执行上下文
   生成 run-id = "run-20260611-143022"
   确定绝对路径 ARTIFACTS = "/path/to/supermeerkat/shared/artifacts/run-20260611-143022/"
   Skill 调用 Agent tool:
     agent: orchestrator
     prompt: "请执行 workflow 文件 /path/to/supermeerkat/workflows/爆款营销.md
               run-id: run-20260611-143022
               共享存储绝对路径: /path/to/supermeerkat/shared/artifacts/run-20260611-143022/"

3. Orchestrator 解析 DAG
   Read /path/to/supermeerkat/workflows/爆款营销.md
   提取 <!-- workflow:step --> JSON
   构建依赖图（串行）:
     step-1 (research) → step-2 (analysis, pauseAfter) → step-3 (analysis, pauseAfter) → step-4 (critique)
   创建 run-state.json:
     { "step-1": "pending", "step-2": "pending", "step-3": "pending", "step-4": "pending" }

4. Orchestrator 执行 step-1（无依赖，直接启动）
   更新 run-state.json: { "step-1": "in_progress", ... }
   调用 Agent tool:
     agent: research
     prompt: "任务：搜索并收集主要竞品的营销策略、定价、目标人群信息
              输出文件（绝对路径）：/path/to/.../run-20260611-143022/research-findings.md
              超时：120 秒"

5. Research Agent 执行
   WebSearch + WebFetch → 收集信息 → 写入 research-findings.md

6. Orchestrator 校验产物
   Read /path/to/.../run-20260611-143022/research-findings.md
   → 文件存在且非空 ✓
   更新 run-state.json: { "step-1": "completed", ... }

7. Orchestrator 执行 step-2（依赖 step-1 已完成）
   更新 run-state.json: { ..., "step-2": "in_progress", ... }
   调用 Agent tool:
     agent: analysis
     prompt: "任务：基于竞品调研结果提炼产品核心卖点和差异化定位
              输入文件（绝对路径）：/path/to/.../research-findings.md
              输出文件（绝对路径）：/path/to/.../analysis-sellpoints.md
              超时：60 秒"

8. Analysis Agent 执行
   Read research-findings.md → 分析 → 写入 analysis-sellpoints.md

9. 暂停点 (pauseAfter=true)
   更新 run-state.json: { ..., "step-2": "completed", ... }
   Orchestrator 展示中间结果摘要:
   "📊 卖点分析已完成。核心结论：
    1. X 差异化优势明显
    2. Y 价格带存在空白
    → 是否继续下一步？ [继续 / 修改分析 / 中止]"

10. 用户确认 → 执行 step-3 → 暂停 → 用户确认 → 执行 step-4

11. Critique Agent 审查
    Read strategy-draft.md → 四维度审查 → 写入 critique-review.md
    更新 run-state.json: { ..., "step-4": "completed" }

12. 汇总最终报告
    Orchestrator 综合所有中间产物：
    - 整合 research + analysis + strategy 内容
    - 逐条对照 critique-review.md 的审查意见
    - 对每条意见标注"已采纳修订"或"暂不采纳（附理由）"
    - 生成 final-report.md → 展示给用户

13. 用户确认是否保留中间产物 → 结束
```

---

## 九、关键接口约定

### 9.1 Agent 工具调用规范

Orchestrator 调度 agent 时的调用格式：

```
调用 Agent 工具:
  agent: <agent-name>
  prompt: |
    任务：<task 内容>
    输入文件（绝对路径）：<ARTIFACTS>/<input-file>  （如有）
    输出文件（绝对路径）：<ARTIFACTS>/<output-file>
    超时：<timeout> 秒
    
    注意：
    - 必须将结果写入输出文件（绝对路径）
    - 不要修改其他文件
    - 如无结果可写，输出说明文字，不得生成空文件
```

### 9.2 共享存储路径约定

```
<plugin-root>/shared/artifacts/<run-id>/
├── run-state.json             # 步骤状态持久化（Orchestrator 维护）
├── <步骤 output 文件>
└── final-report.md
```

- `<plugin-root>` 为插件安装绝对路径，由 Skill 在调用 Orchestrator 时注入
- `<run-id>` 格式：`run-YYYYMMDD-HHmmss`
- 文件命名由 workflow 步骤的 `output` 字段决定
- Agent 只读写自己分配的文件，不触碰其他文件

### 9.3 步骤状态（run-state.json）

Orchestrator 在执行期间持续写入并更新 `run-state.json`：

```json
{
  "runId": "run-20260611-143022",
  "workflow": "爆款营销",
  "startedAt": "2026-06-11T14:30:22Z",
  "steps": {
    "step-1": "completed",
    "step-2": "completed",
    "step-3": "in_progress",
    "step-4": "pending"
  }
}
```

状态值：`pending` → `in_progress` → `completed` / `failed` / `skipped`

**写入时机**：
- 步骤开始前：写入 `in_progress`
- 步骤完成后：写入 `completed` / `failed` / `skipped`
- 文件随 run-id 目录一起保留，支持断点续跑时读取恢复

---

## 十、后续迭代方向

以下功能不在 MVP 范围内，但设计时已预留扩展空间：

1. **断点续跑完善** — 读取 `run-state.json`，自动跳过已完成步骤，从失败步骤恢复
2. **并行执行优化** — 自动识别无依赖步骤并行调度（DAG 结构已支持，需 Orchestrator 实现并行逻辑）
3. **自定义 Agent 创建** (`/supermeerkat:create-agent`) — `agents/` 目录支持用户自定义 agent 文件
4. **Agent 列表查询** (`/supermeerkat:agent-list`)
5. **Workflow 模板库** — 预置常用行业 workflow 模板（针对消费品行业）
6. **共享存储升级** — 从文件系统升级到向量检索，提升信息利用率
7. **Workflow Engine 独立** — 将调度引擎从 Orchestrator 中抽取为独立组件
8. **可视化 DAG** — 生成 workflow 的依赖图可视展示
9. **`/supermeerkat:clean`** — 批量清理历史中间产物

---

## 十一、参考资料

- [Claude Code 创建插件文档](https://code.claude.com/docs/zh-CN/plugins)
- [Claude Code Plugins 参考](https://code.claude.com/docs/zh-CN/plugins-reference)
- [创建和分发 Plugin Marketplace](https://code.claude.com/docs/zh-CN/plugin-marketplaces)
- [插件开发进阶指南](https://inferloop.dev/claude-plugins/appendix/advanced-dev-guide/)
- [Superpowers 插件调研](https://www.yuque.com/qiwoniudaxiang/bfol2c/tpastdnx6ld5wwir)
- [Superpowers 工程级开发](https://datawhalechina.github.io/easy-vibe/zh-cn/stage-3/core-skills/superpowers/)
