# Agent 路由策略详解

深入了解 opencode-orchestrator-skill 的 Agent 路由机制，掌握任务分配的最佳实践。

---

## Agent 能力矩阵

| Agent | 主要职责 | 适用场景 | 能力等级 |
|-------|---------|---------|---------|
| **ultrawork** | 任务复杂度评估 | 所有任务入口 | 决策层 |
| **Prometheus** | 任务规划、分解为 Waves | 复杂任务、架构变更 | 规划层 |
| **Momus** | 任务分发、进度追踪 | 多 Wave 协调 | 协调层 |
| **Atlas** | 代码生成、文件修改 | 原子任务执行 | 执行层 |
| **git-master** | 版本控制操作 | 需要 git 操作的任务 | 辅助层 |

### Agent 详细说明

#### ultrawork
- **核心功能**：智能分析任务复杂度，决定是否需要规划
- **判断标准**：
  - 涉及文件数 < 3：简单任务
  - 涉及文件数 3-10：中等任务
  - 涉及文件数 > 10 或架构变更：复杂任务
- **输出**：路由决策 + 初始上下文

#### Prometheus
- **核心功能**：创建结构化执行计划
- **工作方式**：
  - 分析任务目标和约束
  - 分解为多个 Wave（执行波次）
  - 定义 Wave 间的依赖关系
- **输出**：`.sisyphus/plans/<plan-name>.md`

#### Momus
- **核心功能**：任务分发和状态管理
- **工作方式**：
  - 读取 Prometheus 创建的计划
  - 按 Wave 分发任务到 Atlas
  - 追踪每个任务的执行状态
- **触发条件**：计划包含 2 个以上 Wave

#### Atlas
- **核心功能**：执行具体的代码修改
- **能力范围**：
  - 文件创建、修改、删除
  - 代码重构和优化
  - 单元测试生成
- **输入**：原子任务描述
- **输出**：代码变更

---

## 路由决策树

```
用户请求
    │
    ▼
┌─────────────────────────────────────────┐
│  [1] ultrawork 评估                      │
│  - 分析任务描述                          │
│  - 识别关键词                            │
│  - 估算复杂度                            │
└────────────────┬────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
   简单任务            复杂任务
   (直接执行)              │
        │                 │
        ▼                 ▼
   ┌─────────┐     ┌─────────────────┐
   │  Atlas  │     │  [2] Prometheus  │
   │ 执行    │     │  - 创建执行计划   │
   └─────────┘     │  - 分解为 Waves  │
                   └────────┬────────┘
                            │
                    ┌───────┴───────┐
                    ▼               ▼
              单 Wave          多 Wave
                 │               │
                 ▼               ▼
            ┌─────────┐    ┌─────────────────┐
            │  Atlas  │    │  [3] Momus       │
            │ 执行    │    │  - 分发 Wave     │
            └─────────┘    │  - 追踪进度      │
                           └────────┬────────┘
                                    │
                                    ▼
                             ┌─────────────────┐
                             │  [4] /start-work │
                             │  - 启动工作流    │
                             └────────┬────────┘
                                      │
                                      ▼
                               ┌─────────────────┐
                               │  Atlas          │
                               │  - 执行原子任务  │
                               │  - 代码生成/修改 │
                               └─────────────────┘
```

---

## 触发关键词列表

### 直接路由到 Atlas（简单任务）

| 关键词 | 示例 Prompt | 任务特征 |
|--------|-------------|---------|
| 修复、fix | "修复 login.js 第 42 行的空指针异常" | 单点 Bug 修复 |
| 优化、optimize | "优化 UserService 的错误处理逻辑" | 代码改进 |
| 添加方法、add | "为 UserService 类添加 logout 方法" | 小功能增加 |
| 格式化、format | "格式化 utils.js 代码" | 代码风格 |
| 注释、comment | "为复杂函数添加注释" | 文档补充 |

**路由路径**：`用户请求 → ultrawork → Atlas`

### 路由到 Prometheus（规划任务）

| 关键词 | 示例 Prompt | 任务特征 |
|--------|-------------|---------|
| 实现、implement | "实现完整的用户注册流程" | 多步骤功能 |
| 创建、create | "创建订单模块" | 新模块开发 |
| 开发、develop | "开发支付接口" | 接口开发 |
| 集成、integrate | "集成第三方登录" | 系统集成 |
| 设计、design | "设计数据库模型" | 架构设计 |

**路由路径**：`用户请求 → ultrawork → Prometheus → Atlas`

### 路由到 Prometheus → Momus（复杂任务）

| 关键词 | 示例 Prompt | 任务特征 |
|--------|-------------|---------|
| ultrawork, ulw | "ultrawork: 重构数据访问层" | 自动模式 |
| 重构、refactor | "将单体应用拆分为微服务" | 架构重构 |
| 迁移、migrate | "从 REST 迁移到 GraphQL" | 技术迁移 |
| 重新设计、redesign | "重新设计用户认证系统" | 系统重构 |
| 大规模、large-scale | "大规模清理技术债务" | 批量任务 |
| 计划、plan | "plan: 实现微服务架构" | 显式规划 |

**路由路径**：`用户请求 → ultrawork → Prometheus → Momus → Atlas`

### 特殊路由（调试排错）

| 关键词 | 示例 Prompt | 路由 |
|--------|-------------|------|
| 调试、debug | "调试为什么支付接口超时" | Atlas + git-master |
| 排查、troubleshoot | "排查内存泄漏问题" | Atlas + git-master |
| 测试、test | "测试用户注册流程" | Atlas + 测试工具 |

---

## 使用示例

### 示例 1：简单 Bug 修复

**Prompt**：
```bash
opencode -p "修复 login.js 中的空指针异常" -f json -c ~/myproject -q
```

**路由决策**：
1. ultrawork 识别关键词 "修复"
2. 判断为简单任务（单文件修改）
3. 直接路由到 Atlas

**执行流程**：
```
用户请求 → ultrawork → Atlas → 完成
```

### 示例 2：功能实现

**Prompt**：
```bash
opencode -p "实现用户注册功能，包括表单验证和邮箱验证" -f json -c ~/myproject -q
```

**路由决策**：
1. ultrawork 识别关键词 "实现"
2. 判断为中等复杂度任务
3. 触发 Prometheus 进行规划

**执行流程**：
```
用户请求 → ultrawork → Prometheus → Atlas
                        ↓
              创建 plans/user-registration.md
                        ↓
              Atlas 按 Wave 执行
```

### 示例 3：架构重构（ultrawork 模式）

**Prompt**：
```bash
opencode -p "ultrawork: 将单体应用拆分为微服务架构" -f json -c ~/myproject -q
```

**路由决策**：
1. ultrawork 识别关键词 "ultrawork"
2. 自动评估为复杂任务
3. 完整路由链启动

**执行流程**：
```
用户请求 → ultrawork → Prometheus → Momus → /start-work → Atlas
                ↓            ↓           ↓            ↓
           复杂度评估   创建多 Wave   分发任务    执行原子任务
                        计划                     并报告进度
```

### 示例 4：显式指定 Agent

**Prompt**：
```bash
# 强制使用 Prometheus 规划
opencode -p "请使用 Prometheus 规划：实现订单模块的支付功能" -f json -c ~/myproject -q

# 强制使用 Atlas 执行
opencode -p "使用 Atlas 执行：修复第 42 行的 Bug" -f json -c ~/myproject -q
```

---

## 最佳实践

### 1. 选择合适的触发方式

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 简单修改 | 直接描述 | 快速执行，无需规划开销 |
| 新功能 | 使用 "实现" 关键词 | 自动触发适当规划 |
| 复杂重构 | 使用 "ultrawork:" 前缀 | 确保完整规划流程 |
| 不确定 | 使用 "ultrawork:" 前缀 | 让系统自动决策 |

### 2. 编写有效的 Prompt

**好的 Prompt**：
```bash
opencode -p "为 UserService 类添加 logout 方法，需要：
1. 清除当前 session
2. 记录登出日志
3. 返回登出成功状态
请确保添加对应的单元测试。" -f json -c ~/myproject -q
```

**不好的 Prompt**：
```bash
opencode -p "加个登出功能" -f json -c ~/myproject -q
```

### 3. 监控和干预

```bash
# 检查当前路由决策
cat ~/myproject/.sisyphus/boulder.json | jq '.routing_decision'

# 查看 Prometheus 创建的计划
cat ~/myproject/.sisyphus/plans/*.md

# 如果需要人工干预，可以编辑计划
vim ~/myproject/.sisyphus/plans/current-plan.md
```

### 4. 故障排除

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 简单任务被过度规划 | 关键词识别错误 | 简化 Prompt，避免复杂词汇 |
| 复杂任务未被规划 | 关键词不明显 | 添加 "ultrawork:" 前缀 |
| 任务卡在 Momus | Wave 依赖错误 | 检查计划文件中的依赖关系 |
| Atlas 执行失败 | 上下文不足 | 在 Prompt 中提供更多背景信息 |

---

## 参考资料

- [快速开始指南](./quick-start.md)
- [SKILL.md](../SKILL.md) - 完整参考手册
- [oh-my-opencode Agent 指南](https://github.com/code-yeongyu/oh-my-opencode/blob/main/docs/AGENTS.md)
