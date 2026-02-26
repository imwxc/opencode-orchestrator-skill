# OpenClaw × OpenCode × oh-my-opencode 集成计划

## TL;DR

> **目标**: 让 OpenClaw 作为外部指挥层，协调 OpenCode CLI + oh-my-opencode 完成极复杂任务
> 
> **核心策略**: 通过 Skills + CLI Backends 外部集成方式，**不改动 OpenClaw 核心代码**
> 
> **集成链**: OpenClaw (Skills) → OpenCode CLI → oh-my-opencode (Background Manager)

**Deliverables**:
- **独立 Skill 仓库** `imwxc/opencode-orchestrator-skill`（高级编排 + Agent 路由策略）
- `cli-backends.ts` 配置添加（opencode-cli backend）—— *可选，如需在 OpenClaw 中直接调用*
- 状态同步机制（读取 `.sisyphus/` 和 `.opencode/`）
- **Agent 路由策略**（根据任务类型自动选择合适的 Agent）
- **计划文档**（复制到 Skill 仓库 `docs/`）
- **ClawHub 发布**（从独立仓库上传 Skill）

**⚠️ 重要变更**: Skill 将存放在**独立的 GitHub 仓库**中，而不是 OpenClaw 仓库的 `skills/` 目录。这样可以：
1. 独立版本管理和发布
2. 用户可以通过 ClawHub 直接安装
3. 不需要修改 OpenClaw 核心代码

**Estimated Effort**: Medium
**Parallel Execution**: YES - 6 Waves
**Critical Path**: Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 (Git Commit) → Phase 6 (ClawHub)

## Context

### Original Request

> "让 openclaw 作为 指挥 opencode 完成 极复杂任务的 外部指挥层。由于 opencode 会与 oh my opencode 集成，所以 openclaw 在使用 opencode 和 ohmyopencode 时需要与其紧密连接，并且能够获取到 所有的反馈。我希望 可以 通过 openclaw 的 extension 等外部集成方式来使用，尽量不改动其核心。"

### User Workflow (CRITICAL)

用户处理复杂任务的标准工作流程：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        复杂任务处理工作流程                               │
└──────────────────────────────────────────────────────────────────────────┘

Phase 0: 探索与分析
┌─────────────────┐
│    Sisyphus     │  关键词: ultrawork, ulw
│  (ulw 模式)     │  → 激活 max variant
└────────┬────────┘  → 调用 explore/librarian 并行探索
         │
         ▼
Phase 1: 计划制定
┌─────────────────┐
│   Prometheus    │  接收探索结果
│   + Metis       │  → Prometheus 制定详细计划
└────────┬────────┘  → Metis 进行缺口分析
         │
         ▼
Phase 2: 计划审查
┌─────────────────┐
│     Momus       │  严格审查计划
└────────┬────────┘  → 验证文件引用、任务完整性
         │           → 循环直到 OKAY
         ▼
Phase 3: 用户 Review
┌─────────────────┐
│    用户自己     │  最终确认计划
└────────┬────────┘  → 可选高精度审查
         │
         ▼
Phase 4: 执行
┌─────────────────┐
│ /start-work     │  启动 Atlas Agent
│     Atlas       │  → 按 Wave 并行执行任务
└─────────────────┘  → 每个任务分配最优 category
```

**关键命令**:
- `ultrawork` 或 `ulw` → 激活 Sisyphus ultrawork 模式
- `/start-work <plan-name>` → 启动 Atlas 执行计划
- `/ulw-loop` → ultrawork 循环模式
- `/ralph-loop` → Ralph 循环模式

### Agent 路由策略 (CRITICAL - NEW)

#### oh-my-opencode Agent 能力矩阵

| Agent | 角色 | 成本 | 触发条件 | Category |
|-------|------|------|----------|----------|
| **Sisyphus** | 主编排者 | EXPENSIVE | 默认主 agent | - |
| **Sisyphus-Junior** | 轻量执行 | CHEAP | 简单任务、后台探索 | `quick` |
| **Prometheus** | 计划制定 | EXPENSIVE | "plan", "create work plan" | - |
| **Metis** | 缺口分析 | CHEAP | Prometheus 咨询 | - |
| **Momus** | 计划审查 | CHEAP | "review plan", 高精度模式 | - |
| **Atlas** | 计划执行 | EXPENSIVE | `/start-work` 触发 | - |
| **Oracle** | 架构咨询 | EXPENSIVE | 架构问题、trade-off 分析 | - |
| **Librarian** | 文档检索 | CHEAP | 外部库、API 文档 | - |
| **Explore** | 代码探索 | CHEAP | 代码库模式、实现查找 | - |
| **Hephaestus** | 重构专家 | EXPENSIVE | 重构、代码清理 | - |
| **Multimodal-Looker** | 多媒体分析 | CHEAP | 图片、PDF 分析 | - |

#### Task Category 映射

| Category | 适用场景 | 典型 Agent | 模型要求 |
|----------|----------|------------|----------|
| `quick` | 简单修改、配置、分析 | Sisyphus-Junior | 轻量模型 |
| `visual-engineering` | 前端 UI/UX、样式 | Sisyphus-Junior + playwright skill | 视觉理解能力 |
| `ultrabrain` | 极复杂逻辑、算法 | Sisyphus | 强推理能力 |
| `deep` | 自主问题解决、研究 | Sisyphus | 强推理 + 上下文 |
| `artistry` | 非常规方法 | Sisyphus | 创造性 |
| `unspecified-low` | 低复杂度通用任务 | Sisyphus-Junior | - |
| `unspecified-high` | 高复杂度通用任务 | Sisyphus | - |
| `writing` | 文档、技术写作 | Sisyphus-Junior | - |

#### Agent 选择决策树

```
用户任务输入
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 关键词检测 (keyword-detector hook)                        │
├─────────────────────────────────────────────────────────────┤
│ - "ultrawork" / "ulw" → Sisyphus ultrawork 模式             │
│ - "plan" / "create plan" → Prometheus                      │
│ - "review" / "audit" → Momus                               │
│ - 外部库提到 → Librarian                                    │
│ - 代码模式查找 → Explore                                    │
│ - 架构问题 → Oracle                                         │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 任务复杂度评估                                            │
├─────────────────────────────────────────────────────────────┤
│ - 单文件/小改动 → category="quick"                          │
│ - 多文件/中等改动 → category="unspecified-high"             │
│ - 架构级别 → Oracle 咨询                                    │
│ - 需要探索 → 并行 explore + librarian                       │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 执行阶段选择                                              │
├─────────────────────────────────────────────────────────────┤
│ - 计划阶段 → Prometheus + Metis + Momus                     │
│ - 执行阶段 → Atlas (via /start-work)                        │
│ - 后台任务 → Background Manager (category-based)            │
└─────────────────────────────────────────────────────────────┘
```

#### OpenClaw Skill 中的路由实现

在 `skills/opencode-orchestrator/SKILL.md` 中定义路由指令：

```markdown
## Agent 路由指令

### 触发 Sisyphus Ultrawork 模式
bash pty:true command:"opencode -p 'ultrawork [任务描述]' -c /project"

### 触发 Prometheus 计划制定
bash pty:true command:"opencode -p 'prometheus: [任务描述]' -c /project"

### 触发 Atlas 执行计划
bash pty:true command:"opencode -p '/start-work [plan-name]' -c /project"

### 后台任务 (带 category)
bash pty:true command:"opencode -p '[任务]' -c /project" background:true

### 读取任务状态
Read: .sisyphus/boulder.json
Read: .sisyphus/plans/*.md
```

## TL;DR

> **目标**: 让 OpenClaw 作为外部指挥层，协调 OpenCode CLI + oh-my-opencode 完成极复杂任务
> 
> **核心策略**: 通过 Skills + CLI Backends 外部集成方式，**不改动 OpenClaw 核心代码**
> 
> **集成链**: OpenClaw (Skills) → OpenCode CLI → oh-my-opencode (Background Manager)

**Deliverables**:
- `skills/coding-agent/SKILL.md` 修改版（添加 OpenCode 调用）
- `skills/opencode-orchestrator/` 新 Skill（高级编排）
- `cli-backends.ts` 配置添加（opencode-cli backend）
- 状态同步机制（读取 `.sisyphus/` 和 `.opencode/`）

**Estimated Effort**: Medium
**Parallel Execution**: YES - 4 Waves
**Critical Path**: Phase 1 → Phase 2 → Phase 3 → Phase 4

---

## Context

### Original Request

> "让 openclaw 作为 指挥 opencode 完成 极复杂任务的 外部指挥层。由于 opencode 会与 oh my opencode 集成，所以 openclaw 在使用 opencode 和 ohmyopencode 时需要与其紧密连接，并且能够获取到 所有的反馈。我希望 可以 通过 openclaw 的 extension 等外部集成方式来使用，尽量不改动其核心。"

### Architecture Analysis Summary

**OpenCode CLI** (`/tmp/opencode/`):
- Go 实现的命令行工具
- 支持非交互模式: `opencode -p "prompt" -f json -q -c /path`
- Session 持久化到 SQLite
- 内置 PubSub 事件系统

**oh-my-opencode** (`/tmp/oh-my-opencode/`):
- TypeScript 实现的 OpenCode 插件
- 11 个专业代理（Sisyphus, Prometheus, Oracle, etc.）
- Background Manager 是主要集成点（1633 行）
- 提供 `launch()`, `getTask()`, `cancelTask()` API

**OpenClaw** (`/tmp/openclaw/`):
- Skills 系统: `skills/` 目录，`SKILL.md` + `skill.yaml`
- CLI Backends: `src/agents/cli-backends.ts`
- 已有 Claude Code CLI 集成作为参考

### Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenClaw (指挥层)                        │
│  ┌─────────────────┐  ┌─────────────────────────────────┐   │
│  │ Skills System   │  │ CLI Backends                    │   │
│  │ ┌─────────────┐ │  │ ┌─────────────────────────────┐ │   │
│  │ │coding-agent │ │  │ │ opencode-cli config         │ │   │
│  │ │(修改)       │ │  │ │ - command: opencode         │ │   │
│  │ └─────────────┘ │  │ │ - args: -f json -q          │ │   │
│  │ ┌─────────────┐ │  │ │ - status_parser: custom     │ │   │
│  │ │opencode-    │ │  │ └─────────────────────────────┘ │   │
│  │ │orchestrator │ │  └─────────────────────────────────┘   │
│  │ │(新建)       │ │                                        │
│  │ └─────────────┘ │                                        │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ bash pty:true
┌─────────────────────────────────────────────────────────────┐
│                   OpenCode CLI (执行层)                      │
│  $ opencode -p "complex task" -f json -q -c /project        │
│                                                              │
│  自动加载 oh-my-opencode 插件                                 │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ 插件集成
┌─────────────────────────────────────────────────────────────┐
│              oh-my-opencode (协调层)                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Background Manager                                   │    │
│  │ - launch({description, prompt, agent, category})   │    │
│  │ - getTask(taskId) → status, output, evidence       │    │
│  │ - cancelTask(taskId)                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  状态输出到: .sisyphus/, .opencode/                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Work Objectives

### Core Objective

实现 OpenClaw 作为外部指挥层，通过 Skills 和 CLI Backends 调用 OpenCode CLI，由 oh-my-opencode 提供高级编排能力，并能够获取完整的执行反馈。

### Concrete Deliverables

1. **Phase 1**: `skills/coding-agent/SKILL.md` 修改
   - 添加 OpenCode CLI 调用方式
   - 定义 `opencode` 指令格式

2. **Phase 2**: `skills/opencode-orchestrator/` 新建
   - `SKILL.md`: 高级编排指令
   - `skill.yaml`: Skill 配置

3. **Phase 3**: CLI Backend 配置
   - 在 `cli-backends.ts` 添加 `opencode-cli` 配置
   - 实现状态解析器

4. **Phase 4**: 状态同步机制
   - 读取 `.sisyphus/plans/*.md` 获取任务计划
   - 读取 `.sisyphus/evidence/` 获取执行证据
   - 读取 `.opencode/sessions/` 获取会话状态

### Definition of Done

- [ ] OpenClaw 可以通过 Skill 指令调用 OpenCode CLI
- [ ] OpenClaw 可以获取 OpenCode 执行的 JSON 输出
- [ ] OpenClaw 可以读取 oh-my-opencode 的任务状态
- [ ] 端到端测试通过: OpenClaw → OpenCode → oh-my-opencode → 反馈

### Must Have

- Skills + CLI Backends 集成方式（不改动核心）
- 完整的反馈获取机制
- 状态文件读取能力

### Must NOT Have (Guardrails)

- ❌ 不修改 OpenClaw 核心代码（`src/` 下的主要逻辑）
- ❌ 不修改 OpenCode CLI 核心代码
- ❌ 不修改 oh-my-opencode 核心代码
- ❌ 不添加新的进程间通信机制（只用文件 + CLI 输出）

---

## Verification Strategy

### Test Decision

- **Infrastructure exists**: NO（这是集成项目，不涉及单元测试框架）
- **Automated tests**: NO
- **Agent-Executed QA**: YES（手动验证 + CLI 调用验证）

### QA Policy

每个 Phase 完成后，通过以下方式验证：

1. **CLI 调用验证**: 直接运行 opencode 命令，检查 JSON 输出
2. **文件读取验证**: 检查 `.sisyphus/` 和 `.opencode/` 状态文件
3. **端到端验证**: OpenClaw Skill 指令 → OpenCode 执行 → 反馈读取

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation - 可并行):
├── Task 1: 分析 Claude Code CLI 集成模式 [quick]
├── Task 2: 设计 OpenCode CLI 调用参数 [quick]
├── Task 3: 设计状态文件读取策略 [quick]
└── Task 3.5: 分析 oh-my-opencode Agent 路由机制 [quick]

Wave 2 (Implementation - 顺序依赖):
├── Task 4: 修改 coding-agent/SKILL.md (添加 Agent 路由指令) [quick]
├── Task 5: 创建 opencode-orchestrator/Skill (包含完整路由策略) [unspecified-high]
└── Task 6: 添加 opencode-cli Backend 配置 [quick]

Wave 3 (Integration - 顺序依赖):
├── Task 7: 实现状态同步机制 [unspecified-high]
└── Task 8: 编写使用文档 (包含 Agent 路由指南) [writing]

Wave 4 (Verification - 可并行):
├── Task 9: CLI 调用验证 [quick]
├── Task 10: 状态读取验证 [quick]
├── Task 11: Agent 路由验证 (ulw → Sisyphus, /start-work → Atlas) [quick]
└── Task 12: 端到端集成测试 [unspecified-high]

Wave 5 (Git 提交与仓库管理 - 顺序):
├── Task 13: 复制计划文档到仓库并提交变更 [quick]
└── Task 14: 创建 Pull Request (可选) [quick]

Wave 6 (Skill 测试与 ClawHub 发布 - 顺序):
├── Task 15: Skill 功能测试 [quick]
├── Task 16: 人工确认（等待用户 Review） [quick]
└── Task 17: 上传 Skill 到 ClawHub [quick]
```

### Dependency Matrix

- **1-3.5**: — — 4, 5, 6
- **4**: 1, 2, 3.5 — 7
- **5**: 1, 2, 3.5 — 7
- **6**: 1, 2 — 7
- **7**: 4, 5, 6 — 9, 10, 11, 12
- **8**: 4, 5, 6, 7 — —
- **9**: 7 — 12
- **10**: 7 — 12
- **11**: 7 — 12
- **12**: 9, 10, 11 — 13
- **13**: 12 — 14
- **14**: 13 — 15
- **15**: 14 — 16
- **16**: 15 — 17
- **17**: 16 — —

### Critical Path

Task 1-3.5 → Task 4-6 → Task 7 → Task 12 → Task 13 → Task 14 → Task 15 → Task 16 → Task 17

## TODOs

### Wave 1: Foundation

- [ ] 1. **分析 Claude Code CLI 集成模式**

  **What to do**:
  - 阅读 `/tmp/openclaw/skills/coding-agent/SKILL.md`
  - 提取 Claude Code CLI 调用模式
  - 总结可复用的设计模式

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 纯阅读分析任务，无需复杂推理
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3)
  - **Blocks**: Task 4, 5, 6
  - **Blocked By**: None

  **References**:
  - `/tmp/openclaw/skills/coding-agent/SKILL.md` - Claude Code 调用方式
  - `/tmp/openclaw/src/agents/cli-backends.ts` - CLI Backend 配置

  **Acceptance Criteria**:
  - [ ] 提取 Claude Code 调用指令格式
  - [ ] 识别状态解析机制
  - [ ] 总结 3 个可复用模式

  **QA Scenarios**:
  ```
  Scenario: 验证分析结果完整性
    Tool: Read
    Steps:
      1. 读取分析输出
      2. 验证包含：调用指令格式、状态解析机制、可复用模式
    Expected Result: 输出包含所有必要信息
    Evidence: .sisyphus/evidence/task-1-analysis.md
  ```

---

- [ ] 2. **设计 OpenCode CLI 调用参数**

  **What to do**:
  - 基于 OpenCode CLI 分析结果
  - 设计最优调用参数组合
  - 确定 JSON 输出解析方式

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 参数设计任务，基于已有分析
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3)
  - **Blocks**: Task 4, 5, 6
  - **Blocked By**: None

  **References**:
  - `/tmp/opencode/cmd/root.go` - CLI 参数定义
  - `/tmp/opencode/internal/app/app.go:RunNonInteractive` - 非交互模式

  **Acceptance Criteria**:
  - [ ] 确定最小调用参数集
  - [ ] 设计 JSON 输出格式解析
  - [ ] 记录错误处理策略

  **QA Scenarios**:
  ```
  Scenario: 验证参数设计可用性
    Tool: Bash
    Steps:
      1. cd /tmp/opencode && go build -o opencode ./cmd
      2. ./opencode --help | grep -E "json|quiet|command"
    Expected Result: 确认参数存在且符合设计
    Evidence: .sisyphus/evidence/task-2-params.txt
  ```

---

- [ ] 3. **设计状态文件读取策略**

  **What to do**:
  - 分析 `.sisyphus/` 目录结构
  - 分析 `.opencode/` 目录结构
  - 设计状态读取接口

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 文件结构分析，无需复杂推理
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:
  - `/tmp/oh-my-opencode/src/features/background-agent/manager.ts` - 状态管理
  - `/tmp/opencode/internal/session/session.go` - Session 存储

  **Acceptance Criteria**:
  - [ ] 列出所有状态文件路径
  - [ ] 定义每个文件的解析方式
  - [ ] 设计状态数据结构

  **QA Scenarios**:
  ```
  Scenario: 验证状态文件路径正确性
    Tool: Bash
    Steps:
      1. 检查示例项目中的 .sisyphus/ 目录
      2. 列出所有 .md 和 .json 文件
    Expected Result: 路径列表与设计一致
    Evidence: .sisyphus/evidence/task-3-state-files.txt
  ```

---

- [ ] 3.5. **分析 oh-my-opencode Agent 路由机制**

  **What to do**:
  - 阅读 `/tmp/oh-my-opencode/src/agents/dynamic-agent-prompt-builder.ts`
  - 阅读 `/tmp/oh-my-opencode/src/hooks/keyword-detector/hook.ts`
  - 分析 AgentPromptMetadata 结构 (category, cost, triggers, useWhen, avoidWhen)
  - 理解 keyword-detector 如何触发 ultrawork 模式
  - 总结完整的 Agent 选择决策流程

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 纯阅读分析任务，提取可复用模式
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3)
  - **Blocks**: Task 4, 5 (需要路由策略)
  - **Blocked By**: None

  **References**:
  - `/tmp/oh-my-opencode/src/agents/types.ts` - AgentPromptMetadata 接口
  - `/tmp/oh-my-opencode/src/agents/dynamic-agent-prompt-builder.ts` - Agent 选择逻辑
  - `/tmp/oh-my-opencode/src/hooks/keyword-detector/hook.ts` - ultrawork 触发
  - `/tmp/oh-my-opencode/src/hooks/keyword-detector/ultrawork/planner.ts` - ultrawork 模式提示
  - `/tmp/oh-my-opencode/AGENTS.md` - Agent 能力概览

  **Acceptance Criteria**:
  - [ ] 提取完整的 Agent 能力矩阵
  - [ ] 理解 keyword-detector 的触发机制
  - [ ] 理解 category 与 Agent 的映射关系
  - [ ] 总结用户工作流程 (ulw → Prometheus → Momus → /start-work → Atlas)

  **QA Scenarios**:
  ```
  Scenario: 验证 Agent 路由理解正确性
    Tool: Read
    Steps:
      1. 读取分析输出
      2. 验证包含：Agent 能力矩阵、触发机制、category 映射、工作流程
    Expected Result: 输出包含所有必要信息
    Evidence: .sisyphus/evidence/task-3.5-agent-routing.md
  ```

---

### Wave 2: Implementation

- [ ] 4. **修改 coding-agent/SKILL.md**

  **What to do**:
  - 在现有 SKILL.md 中添加 OpenCode 部分
  - 定义 `opencode` 指令格式
  - 添加使用示例

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 文档修改，基于已完成的设计
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 7
  - **Blocked By**: Task 1, 2

  **References**:
  - `/tmp/openclaw/skills/coding-agent/SKILL.md` - 现有模板
  - Task 1, 2 的设计输出

  **Acceptance Criteria**:
  - [ ] SKILL.md 包含 OpenCode 调用指令
  - [ ] 指令格式清晰可执行
  - [ ] 包含 2+ 使用示例

  **QA Scenarios**:
  ```
  Scenario: 验证指令格式正确性
    Tool: Read
    Steps:
      1. 读取修改后的 SKILL.md
      2. 验证包含 opencode 指令定义
    Expected Result: 指令格式符合设计
    Evidence: .sisyphus/evidence/task-4-skill.md
  ```

  **Commit**: YES
  - Message: `feat(skills): add opencode-cli integration to coding-agent`
  - Files: `skills/coding-agent/SKILL.md`

---

- [ ] 5. **创建独立 Skill 仓库并编写 opencode-orchestrator Skill**

  **What to do**:
  - 使用 `gh repo create imwxc/opencode-orchestrator-skill --public` 创建独立仓库
  - Clone 仓库到 `/tmp/opencode-orchestrator-skill/`
  - 创建 Skill 目录结构:
    ```
    opencode-orchestrator-skill/
    ├── SKILL.md          # 主技能文档
    ├── skill.yaml        # 技能配置
    ├── README.md         # 仓库说明
    └── docs/
        └── agent-routing.md  # Agent 路由策略文档
    ```
  - 编写 `SKILL.md`: 高级编排指令 + Agent 路由策略
  - 编写 `skill.yaml`: Skill 配置（包含 ClawHub 发布信息）

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 需要设计新的编排模式，涉及复杂逻辑
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 7
  - **Blocked By**: Task 1, 2

  **References**:
  - `/tmp/openclaw/skills/coding-agent/` - 参考结构
  - `/tmp/oh-my-opencode/AGENTS.md` - 代理能力说明

  **Acceptance Criteria**:
  - [ ] GitHub 仓库已创建: `imwxc/opencode-orchestrator-skill`
  - [ ] 目录结构完整（SKILL.md + skill.yaml + README.md）
  - [ ] 包含任务分解指令
  - [ ] 包含 Agent 路由策略
  - [ ] 包含状态查询指令
  - [ ] 包含错误恢复指令

  **QA Scenarios**:
  ```
  Scenario: 验证 Skill 仓库结构
    Tool: Bash
    Steps:
      1. gh repo view imwxc/opencode-orchestrator-skill --json name,owner
      2. ls -la /tmp/opencode-orchestrator-skill/
      3. cat /tmp/opencode-orchestrator-skill/skill.yaml
    Expected Result: 仓库存在，结构正确
    Evidence: .sisyphus/evidence/task-5-skill-repo.txt
  ```

  **Commit**: YES
  - Message: `feat: initial opencode-orchestrator skill structure`
  - Files: `/tmp/opencode-orchestrator-skill/`

---

- [ ] 6. **添加 opencode-cli Backend 配置**

  **What to do**:
  - 在 `cli-backends.ts` 添加 `opencode-cli` 配置
  - 实现状态解析器
  - 添加错误处理

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 配置添加，基于已有模式
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 7
  - **Blocked By**: Task 1, 2

  **References**:
  - `/tmp/openclaw/src/agents/cli-backends.ts` - 现有配置
  - Task 2 的参数设计

  **Acceptance Criteria**:
  - [ ] 配置符合 `CliBackendConfig` 接口
  - [ ] 状态解析器正确处理 JSON 输出
  - [ ] 错误处理覆盖常见场景

  **QA Scenarios**:
  ```
  Scenario: 验证配置有效性
    Tool: Bash
    Steps:
      1. 检查 TypeScript 语法
      2. 验证接口实现完整性
    Expected Result: 配置通过类型检查
    Evidence: .sisyphus/evidence/task-6-backend-config.txt
  ```

  **Commit**: YES
  - Message: `feat(cli-backends): add opencode-cli backend configuration`
  - Files: `src/agents/cli-backends.ts`

---

### Wave 3: Integration

- [ ] 7. **实现状态同步机制**

  **What to do**:
  - 实现读取 `.sisyphus/plans/*.md` 的逻辑
  - 实现读取 `.sisyphus/evidence/` 的逻辑
  - 实现读取 `.opencode/sessions/` 的逻辑
  - 设计状态同步指令

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 需要理解多个状态文件格式，设计统一读取接口
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 9, 10, 11
  - **Blocked By**: Task 4, 5, 6

  **References**:
  - Task 3 的状态文件设计
  - `/tmp/oh-my-opencode/src/features/background-agent/manager.ts`

  **Acceptance Criteria**:
  - [ ] 可以读取任务计划
  - [ ] 可以读取执行证据
  - [ ] 可以读取会话状态
  - [ ] 状态同步指令可用

  **QA Scenarios**:
  ```
  Scenario: 验证状态读取正确性
    Tool: Bash
    Steps:
      1. 使用状态同步指令
      2. 验证输出包含任务状态
    Expected Result: 状态正确返回
    Evidence: .sisyphus/evidence/task-7-state-sync.txt
  ```

  **Commit**: YES
  - Message: `feat(skills): add state synchronization mechanism`
  - Files: `skills/opencode-orchestrator/SKILL.md`

---

- [ ] 8. **编写使用文档**

  **What to do**:
  - 编写集成使用指南
  - 包含完整示例
  - 包含故障排除

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: 文档编写任务
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: None
  - **Blocked By**: Task 4, 5, 6, 7

  **References**:
  - 所有已完成任务的输出

  **Acceptance Criteria**:
  - [ ] 文档包含快速开始指南
  - [ ] 文档包含 3+ 使用示例
  - [ ] 文档包含常见问题解答

  **Commit**: YES
  - Message: `docs: add opencode integration usage guide`
  - Files: `docs/opencode-integration.md`

---

### Wave 4: Verification

- [ ] 9. **CLI 调用验证**

  **What to do**:
  - 验证 OpenClaw 可以调用 OpenCode CLI
  - 验证 JSON 输出正确解析
  - 记录验证结果

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 简单验证任务
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Tasks 10)
  - **Blocks**: Task 11
  - **Blocked By**: Task 7

  **References**:
  - Task 4, 5, 6 的实现

  **Acceptance Criteria**:
  - [ ] OpenCode CLI 成功调用
  - [ ] JSON 输出正确解析
  - [ ] 错误场景正确处理

  **QA Scenarios**:
  ```
  Scenario: 端到端 CLI 调用测试
    Tool: Bash
    Steps:
      1. cd /tmp/openclaw
      2. 运行集成测试脚本
      3. 验证输出
    Expected Result: 所有测试通过
    Evidence: .sisyphus/evidence/task-9-cli-test.txt
  ```

---

- [ ] 10. **状态读取验证**

  **What to do**:
  - 验证可以读取 `.sisyphus/` 状态
  - 验证可以读取 `.opencode/` 状态
  - 记录验证结果

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 简单验证任务
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Tasks 9)
  - **Blocks**: Task 11
  - **Blocked By**: Task 7

  **References**:
  - Task 7 的状态同步实现

  **Acceptance Criteria**:
  - [ ] 计划文件正确读取
  - [ ] 证据文件正确读取
  - [ ] 会话状态正确读取

  **QA Scenarios**:
  ```
  Scenario: 状态读取测试
    Tool: Bash
    Steps:
      1. 运行状态读取脚本
      2. 验证输出包含预期字段
    Expected Result: 状态正确返回
    Evidence: .sisyphus/evidence/task-10-state-test.txt
  ```

---

- [ ] 12. **端到端集成测试**

  **What to do**:
  - 设计完整测试场景
  - 执行 OpenClaw → OpenCode → oh-my-opencode 调用链
  - 验证反馈完整性

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 复杂集成测试，需要处理多系统交互
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Wave 4 Final)
  - **Blocks**: Task 13
  - **Blocked By**: Task 9, 10, 11

  **References**:
  - 所有已完成的实现

  **Acceptance Criteria**:
  - [ ] 调用链完整执行
  - [ ] 反馈完整获取
  - [ ] 错误场景正确处理

  **QA Scenarios**:
  ```
  Scenario: 完整集成测试
    Tool: Bash
    Steps:
      1. 准备测试环境
      2. 执行完整调用链
      3. 验证反馈内容
    Expected Result: 所有步骤成功
    Evidence: .sisyphus/evidence/task-12-e2e-test.txt
  ```

---

- [ ] 11. **Agent 路由验证**

  **What to do**:
  - 验证 ultrawork 关键词触发 Sisyphus ultrawork 模式
  - 验证 /start-work 命令触发 Atlas Agent
  - 验证 Prometheus 计划制定流程
  - 验证 Momus 审查流程

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 简单验证任务，基于 CLI 调用
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Tasks 9, 10)
  - **Blocks**: Task 12
  - **Blocked By**: Task 7

  **References**:
  - `/tmp/oh-my-opencode/src/hooks/keyword-detector/hook.ts` - ultrawork 触发
  - `/tmp/oh-my-opencode/src/features/builtin-commands/templates/start-work.ts` - /start-work 实现

  **Acceptance Criteria**:
  - [ ] ultrawork 关键词正确触发 max variant
  - [ ] /start-work 正确启动 Atlas Agent
  - [ ] Agent 路由决策树符合预期

  **QA Scenarios**:
  ```
  Scenario: Agent 路由验证
    Tool: Bash
    Steps:
      1. cd /tmp/opencode && ./opencode -p "ultrawork test" -f json -q
      2. 检查输出中的 variant 是否为 "max"
      3. cd /tmp/opencode && ./opencode -p "/start-work test-plan" -f json -q
      4. 检查是否切换到 Atlas Agent
    Expected Result: Agent 路由正确触发
    Evidence: .sisyphus/evidence/task-11-agent-routing.txt
  ```

  ```

---

### Wave 5: Git 提交与仓库管理

- [ ] 13. **复制计划文档到仓库并提交变更**

  **What to do**:
  - 创建 `/tmp/openclaw/docs/plans/` 目录（如不存在）
  - 复制计划文件到仓库:
    ```bash
    mkdir -p /tmp/openclaw/docs/plans/
    cp /Users/wangxianchen/.sisyphus/plans/openclaw-opencode-integration.md /tmp/openclaw/docs/plans/
    ```
  - 在 `/tmp/openclaw` 创建 feature 分支
  - 使用 `git add` 添加所有修改的文件（包括复制的计划文件）
  - 使用 `git commit` 提交变更
  - 使用 `gh` 验证远程仓库状态
  - 推送变更到 fork 仓库

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Git 操作任务，使用 git-master skill
  - **Skills**: [`git-master`]
    - `git-master`: Git 操作最佳实践，原子提交

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Wave 5)
  - **Blocks**: Task 14
  - **Blocked By**: Task 12 (所有实现完成)

  **References**:
  - `/tmp/openclaw/` - OpenClaw fork 仓库
  - Commit Strategy 表格 - 提交消息格式

  **Acceptance Criteria**:
  - [ ] 计划文件已复制到 `/tmp/openclaw/docs/plans/openclaw-opencode-integration.md`
  - [ ] 创建 feature 分支: `feature/opencode-ohmyopencode-integration`
  - [ ] 所有变更文件已添加到暂存区（包括复制的计划文件）
  - [ ] 提交消息符合 Conventional Commits 规范
  - [ ] 变更已推送到 fork 仓库
  **QA Scenarios**:
  ```
  Scenario: Git 提交验证
    Tool: Bash
    Steps:
      1. cd /tmp/openclaw && git status
      2. git log --oneline -5
      3. git branch -a | grep feature/opencode
      4. gh repo view imwxc/openclaw --json defaultBranchRef
    Expected Result: 分支存在，提交记录正确
    Evidence: .sisyphus/evidence/task-13-git-commit.txt
  ```

---

- [ ] 14. **创建 Pull Request (可选)**

  **What to do**:
  - 使用 `gh pr create` 创建 Pull Request
  - 添加 PR 描述，包含变更摘要
  - 关联相关 Issue (如有)
  - 等待 CI 检查通过

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: GitHub CLI 操作
  - **Skills**: [`git-master`]
    - `git-master`: PR 创建最佳实践

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Wave 5 Final)
  - **Blocks**: None
  - **Blocked By**: Task 13

  **References**:
  - `gh pr create --help` - PR 创建命令

  **Acceptance Criteria**:
  - [ ] PR 已创建
  - [ ] PR 描述清晰
  - [ ] CI 检查通过 (如有)

  **QA Scenarios**:
  ```
  Scenario: PR 创建验证
    Tool: Bash
    Steps:
      1. cd /tmp/openclaw && gh pr list --head feature/opencode-ohmyopencode-integration
      2. gh pr view <PR-NUMBER> --json title,state,url
    Expected Result: PR 存在且状态为 open
    Evidence: .sisyphus/evidence/task-14-pr-created.txt
  ```

---

### Wave 6: Skill 测试与 ClawHub 发布

- [ ] 15. **Skill 功能测试**

  **What to do**:
  - 验证独立 Skill 仓库中的 `SKILL.md` 完整性
  - 验证 `skill.yaml` 配置正确
  - 测试 Agent 路由指令的正确性
  - 验证 ClawHub 发布准备就绪

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Skill 测试验证任务
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Wave 6)
  - **Blocks**: Task 16
  - **Blocked By**: Task 14

  **References**:
  - `/tmp/opencode-orchestrator-skill/SKILL.md` - 独立仓库中的 Skill
  - `/tmp/opencode-orchestrator-skill/skill.yaml` - Skill 配置

  **Acceptance Criteria**:
  - [ ] SKILL.md 格式正确，包含完整的 Agent 路由策略
  - [ ] skill.yaml 配置符合 ClawHub 发布要求
  - [ ] README.md 包含使用说明
  - [ ] Agent 路由指令可正常执行

  **QA Scenarios**:
  ```
  Scenario: Skill 功能测试
    Tool: Bash
    Steps:
      1. cd /tmp/opencode-orchestrator-skill && cat SKILL.md | head -100
      2. cat skill.yaml
      3. 验证 README.md 存在且包含安装说明
    Expected Result: Skill 格式正确，可发布到 ClawHub
    Evidence: .sisyphus/evidence/task-15-skill-test.txt
  ```
---

- [ ] 16. **人工确认（等待用户 Review）**

  **What to do**:
  - 暂停执行，等待用户确认
  - 用户需要检查：
    - Skill 内容是否符合预期
    - Agent 路由策略是否正确
    - 测试结果是否通过
  - 用户确认后继续执行 Task 17

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 等待用户确认的任务
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Wave 6 - 阻塞点)
  - **Blocks**: Task 17
  - **Blocked By**: Task 15

  **References**:
  - Task 15 的测试结果
  - `.sisyphus/evidence/task-15-skill-test.txt`

  **Acceptance Criteria**:
  - [ ] 用户已 Review Skill 内容
  - [ ] 用户已确认 Agent 路由策略
  - [ ] 用户已确认测试结果
  - [ ] 用户明确同意继续上传到 ClawHub

  **QA Scenarios**:
  ```
  Scenario: 人工确认
    Tool: Question
    Steps:
      1. 展示 Skill 测试结果摘要
      2. 询问用户是否确认上传到 ClawHub
    Expected Result: 用户选择 "确认上传" 或 "需要修改"
    Evidence: .sisyphus/evidence/task-16-user-confirmation.txt
  ```

  **⚠️ 注意**: 此任务会暂停执行流程，等待用户明确确认后才会继续。

---

- [ ] 17. **上传 Skill 到 ClawHub**

  **What to do**:
  - 使用 OpenClaw 的 ClawHub 上传命令
  - 从独立仓库 `/tmp/opencode-orchestrator-skill/` 上传到 ClawHub
  - 验证上传成功
  - 记录 Skill 在 ClawHub 的 URL

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: ClawHub 上传操作
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Wave 6 Final)
  - **Blocks**: None
  - **Blocked By**: Task 16 (必须等待人工确认)

  **References**:
  - OpenClaw ClawHub 文档
  - `/tmp/opencode-orchestrator-skill/` - 独立 Skill 仓库

  **Acceptance Criteria**:
  - [ ] Skill 成功上传到 ClawHub
  - [ ] 获得 ClawHub Skill URL
  - [ ] Skill 在 ClawHub 上可被其他用户安装

  **QA Scenarios**:
  ```
  Scenario: ClawHub 上传验证
    Tool: Bash
    Steps:
      1. cd /tmp/opencode-orchestrator-skill && openclaw skill publish .
      2. 验证返回的 ClawHub URL
      3. 访问 URL 确认 Skill 页面存在
    Expected Result: Skill 成功发布到 ClawHub
    Evidence: .sisyphus/evidence/task-17-clawhub-publish.txt
  ```

  **Commit**: YES
  - Message: `chore: publish opencode-orchestrator skill to ClawHub`
  - Files: `/tmp/opencode-orchestrator-skill/skill.yaml` (更新 ClawHub 元数据)
---

---
## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  验证所有 "Must Have" 已实现，所有 "Must NOT Have" 未出现。

- [ ] F2. **Code Quality Review** — `unspecified-high`
  检查代码风格、错误处理、文档完整性。

- [ ] F3. **Real Manual QA** — `unspecified-high`
  执行所有 QA 场景，捕获证据。

- [ ] F4. **Scope Fidelity Check** — `deep`
  验证实现与计划一致，无范围蔓延。

---

## Commit Strategy

| Task | Repository | Commit | Message |
|------|------------|--------|---------|
| 5 | `imwxc/opencode-orchestrator-skill` | YES | `feat: initial opencode-orchestrator skill structure` |
| 6 | `imwxc/openclaw` | YES | `feat(cli-backends): add opencode-cli backend configuration` |
| 7 | `imwxc/opencode-orchestrator-skill` | YES | `feat(skills): add state synchronization mechanism` |
| 8 | `imwxc/opencode-orchestrator-skill` | YES | `docs: add usage guide` |
| 13 | `imwxc/opencode-orchestrator-skill` | YES | `docs: add integration plan` |
| 17 | `imwxc/opencode-orchestrator-skill` | YES | `chore: publish to ClawHub` |

**⚠️ 注意**: Skill 存放在独立仓库 `imwxc/opencode-orchestrator-skill`，而非 OpenClaw 仓库。
## Success Criteria

### Verification Commands

```bash
# 1. 验证 OpenCode CLI 可调用
cd /tmp/opencode && go build -o opencode ./cmd
./opencode --help

# 2. 验证 oh-my-opencode 可加载
cd /tmp/oh-my-opencode && npm run build

# 3. 验证独立 Skill 仓库
gh repo view imwxc/opencode-orchestrator-skill --json name,owner

# 4. 验证 Skill 结构
cd /tmp/opencode-orchestrator-skill
ls -la && cat SKILL.md | head -50

# 5. 验证 ClawHub 发布
openclaw skill list --installed | grep opencode-orchestrator
```

### Final Checklist

- [ ] 所有 "Must Have" 已实现
- [ ] 所有 "Must NOT Have" 未出现
- [ ] 端到端测试通过
- [ ] 文档完整
- [ ] Skill 已提交到独立仓库 `imwxc/opencode-orchestrator-skill`
- [ ] Skill 已上传到 ClawHub
