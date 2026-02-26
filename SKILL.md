# opencode-orchestrator

> OpenCode Agent 编排技能 - 用于任务分解、Agent 路由与状态管理

## 概述

本技能提供 OpenCode CLI 的编排能力，包括：
- **任务分解**：将复杂任务拆分为原子单元
- **Agent 路由**：根据任务类型选择合适的 Agent
- **状态管理**：读取和解析 OpenCode 运行状态
- **后台任务**：管理长时间运行的任务
- **错误恢复**：处理失败任务的重试策略

---

## 1. OpenCode CLI 调用指令

### 基本调用格式

```bash
opencode -p "<prompt>" -f json -c <project-dir> -q
```

**参数说明：**
| 参数 | 说明 | 示例 |
|------|------|------|
| `-p` | Prompt 指令 | `-p "实现用户登录功能"` |
| `-f` | 输出格式 | `-f json` (推荐) 或 `-f text` |
| `-c` | 工作目录 | `-c /path/to/project` |
| `-q` | 静默模式（无交互确认） | `-q` |

### 三种执行模式

#### 模式 A：单次任务（同步）
适用于短任务、需要立即获取结果：

```bash
opencode -p "为 UserService 添加单元测试" -f json -c ~/myproject -q
```

#### 模式 B：后台任务（异步）
适用于耗时较长的任务，需持续监控：

```bash
# 启动后台任务
opencode -p "重构整个认证模块" -f json -c ~/myproject -q &

# 或通过 nohup
nohup opencode -p "重构整个认证模块" -f json -c ~/myproject -q > /tmp/opencode.log 2>&1 &
```

#### 模式 C：会话恢复
继续之前的会话：

```bash
opencode --resume <session-id> -c ~/myproject
```

---

## 2. Agent 路由策略

### 路由层级

```
用户请求
    │
    ▼
┌─────────────────────────────────────────┐
│  ultrawork                               │
│  - 判断任务复杂度                         │
│  - 决定是否需要规划                       │
└────────────────┬────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
   简单任务           复杂任务
   (直接执行)              │
                          ▼
                 ┌─────────────────┐
                 │  Prometheus      │
                 │  - 创建执行计划   │
                 │  - 分解为 Waves   │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  Momus          │
                 │  - 任务分发      │
                 │  - 进度追踪      │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  /start-work     │
                 │  - 启动工作流    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  Atlas          │
                 │  - 执行原子任务   │
                 │  - 代码生成/修改 │
                 └─────────────────┘
```

### Agent 选择规则

| 任务类型 | 路由目标 | 示例 Prompt |
|---------|---------|-------------|
| 简单代码修改 | Atlas | "修复 login.py 第 42 行的空指针异常" |
| 单文件重构 | Atlas | "优化 UserService 的错误处理逻辑" |
| 多文件功能 | Prometheus → Atlas | "实现完整的用户注册流程" |
| 架构变更 | Prometheus → Momus → Atlas | "将单体应用拆分为微服务" |
| 调试排错 | Atlas + git-master | "找出为什么支付接口超时" |

### 调用示例

```bash
# 触发 Prometheus 规划
opencode -p "请使用 Prometheus 规划：实现订单模块的支付功能集成" -f json -c ~/myproject -q

# 触发 ultrawork 模式（自动决策）
opencode -p "ultrawork: 重构数据访问层，提高性能" -f json -c ~/myproject -q
```

---

## 3. 状态读取指令

### 状态文件位置

OpenCode 在项目目录下维护以下状态文件：

```
.sisyphus/
├── plans/
│   └── <plan-name>.md      # 执行计划文件
├── boulder.json            # 当前任务状态
├── sessions/               # 会话历史
└── notepads/               # 学习笔记
```

### 读取执行计划

```bash
# 查看当前计划
cat ~/myproject/.sisyphus/plans/*.md

# 查看特定计划
cat ~/myproject/.sisyphus/plans/my-feature.md
```

### 读取任务状态 (boulder.json)

```bash
# 查看当前任务状态
cat ~/myproject/.sisyphus/boulder.json | jq .
```

**boulder.json 结构示例：**
```json
{
  "current_wave": 2,
  "total_waves": 5,
  "status": "in_progress",
  "tasks": [
    {"id": "task-1", "status": "completed", "agent": "atlas"},
    {"id": "task-2", "status": "in_progress", "agent": "atlas"}
  ]
}
```

### 检查计划完成度

```bash
# 提取计划中的任务完成状态
grep -E "^\s*-\s*\[[ x]\]" ~/myproject/.sisyphus/plans/*.md | head -20
```

---

## 4. 后台任务管理

### 启动后台任务

```bash
# 方式 1: 直接后台执行
nohup opencode -p "长时间任务描述" -f json -c ~/myproject -q > /tmp/oc-task.log 2>&1 &
echo $! > /tmp/oc-task.pid

# 方式 2: 使用 tmux（推荐用于需要交互的任务）
tmux new-session -d -s oc-task "opencode -p '任务描述' -c ~/myproject"
```

### 监控后台任务

```bash
# 查看日志输出
tail -f /tmp/oc-task.log

# 检查进程状态
ps -p $(cat /tmp/oc-task.pid)

# 检查 tmux 会话
tmux capture-pane -t oc-task -p
```

### 终止后台任务

```bash
# 终止普通后台任务
kill $(cat /tmp/oc-task.pid)

# 终止 tmux 会话
tmux kill-session -t oc-task
```

---

## 5. 错误恢复指令

### 常见错误场景

| 错误类型 | 症状 | 恢复策略 |
|---------|------|---------|
| Agent 超时 | 长时间无输出 | 重启任务，增加超时 |
| 依赖缺失 | 模块导入失败 | 先安装依赖，再重试 |
| 冲突锁定 | 文件被锁定 | 清理锁文件 |
| 配置错误 | 初始化失败 | 检查配置文件 |

### 恢复工作流

```bash
# 1. 检查当前状态
cat ~/myproject/.sisyphus/boulder.json | jq '.status'

# 2. 如果状态为 "failed" 或 "stuck"
#    查看最近的错误日志
tail -100 ~/myproject/.sisyphus/sessions/latest.log

# 3. 重置任务状态（谨慎使用）
rm ~/myproject/.sisyphus/boulder.json

# 4. 从断点恢复
opencode --resume-from-wave 3 -c ~/myproject
```

### 强制重新规划

```bash
# 清除当前计划，重新开始
rm -rf ~/myproject/.sisyphus/plans/*.md
opencode -p "请重新规划：原任务描述" -f json -c ~/myproject -q
```

---

## 6. 集成示例

### ClawCla 集成调用

当从 ClawCla 调用 OpenCode 时，使用以下模式：

```
用户: 帮我实现用户登录功能

ClawCla 响应:
我将使用 OpenCode 来执行此任务。让我启动编排流程...

[调用 OpenCode]
opencode -p "ultrawork: 实现用户登录功能，包括表单验证、密码加密和 session 管理" -f json -c ~/target-project -q

[监控状态]
正在通过 Prometheus 规划任务...
计划已创建，共 3 个 Wave。

[执行 Wave 1]
Atlas 正在创建基础文件结构...

[完成]
任务已完成。请查看以下文件：
- src/auth/login.ts
- src/auth/session.ts
- tests/auth.test.ts
```

---

## 7. 最佳实践

### 任务分解原则

1. **原子性**：每个任务应该是独立可完成的
2. **可验证**：任务有明确的完成标准
3. **有依赖**：明确任务间的依赖关系
4. **可回滚**：每个变更都应该可以独立回滚

### Prompt 编写建议

```bash
# 好的 Prompt
opencode -p "为 UserService 类添加 logout 方法，需要：
1. 清除当前 session
2. 记录登出日志
3. 返回登出成功状态
请确保添加对应的单元测试。" -f json -c ~/myproject -q

# 不好的 Prompt（太模糊）
opencode -p "加个登出功能" -f json -c ~/myproject -q
```

### 状态检查频率

- 短任务（< 5 分钟）：完成后检查一次
- 中等任务（5-30 分钟）：每 2 分钟检查一次
- 长任务（> 30 分钟）：每 5 分钟检查一次

---

## 参考资料

- [OpenCode CLI 文档](https://github.com/code-yeongyu/oh-my-opencode)
- [oh-my-opencode Agent 指南](https://github.com/code-yeongyu/oh-my-opencode/blob/main/docs/AGENTS.md)
- [ClawHub Skills 开发指南](https://github.com/clawhub/skills)
