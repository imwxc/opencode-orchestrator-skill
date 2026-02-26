# 快速开始指南

5 分钟掌握 opencode-orchestrator-skill 的基本使用。

---

## 前置条件

在开始之前，请确保已安装：

1. **OpenCode CLI** >= 0.5.0
   ```bash
   npm install -g @oh-my-opencode/cli
   ```

2. **本技能**（通过 ClawHub 安装）
   ```bash
   clawhub install imwxc/opencode-orchestrator-skill
   ```

3. **jq**（可选但推荐，用于解析状态）
   ```bash
   # macOS
   brew install jq
   
   # Linux
   sudo apt-get install jq
   ```

---

## 5 分钟快速上手

### 第 1 分钟：验证安装

```bash
# 检查 OpenCode CLI
opencode --version

# 检查技能安装
clawhub list | grep opencode-orchestrator
```

### 第 2 分钟：执行第一个任务

创建一个简单的代码修改任务：

```bash
# 进入你的项目目录
cd ~/myproject

# 执行简单任务
opencode -p "为 README.md 添加项目标题" -f json -c . -q
```

**参数说明：**
- `-p`：Prompt 指令
- `-f json`：以 JSON 格式输出
- `-c .`：当前目录作为工作目录
- `-q`：静默模式（无需确认）

### 第 3 分钟：使用 ultrawork 模式

对于复杂任务，使用 ultrawork 模式自动规划：

```bash
opencode -p "ultrawork: 为项目添加 TypeScript 类型定义" -f json -c . -q
```

此时系统会：
1. 分析任务复杂度
2. 自动生成执行计划（存放到 `.sisyphus/plans/`）
3. 按 Wave 逐步执行

### 第 4 分钟：监控任务状态

打开另一个终端，实时监控进度：

```bash
# 查看当前任务状态
cat ~/myproject/.sisyphus/boulder.json | jq .
```

输出示例：
```json
{
  "current_wave": 1,
  "total_waves": 3,
  "status": "in_progress",
  "tasks": [
    {"id": "task-1", "status": "completed", "agent": "atlas"},
    {"id": "task-2", "status": "in_progress", "agent": "atlas"}
  ]
}
```

### 第 5 分钟：查看执行计划

```bash
# 查看所有计划
ls ~/myproject/.sisyphus/plans/

# 查看当前计划内容
cat ~/myproject/.sisyphus/plans/*.md
```

---

## 常见使用场景

### 场景 1：修复 Bug

```bash
# 直接执行，Atlas 会处理
opencode -p "修复 login.js 中的空指针异常" -f json -c ~/myproject -q
```

### 场景 2：添加新功能

```bash
# 使用 ultrawork 自动规划
opencode -p "ultrawork: 实现用户注册功能，包括表单验证和邮箱验证" -f json -c ~/myproject -q
```

### 场景 3：代码重构

```bash
# 复杂重构任务
opencode -p "ultrawork: 将 utils.js 拆分为独立的模块文件" -f json -c ~/myproject -q
```

### 场景 4：后台执行长任务

```bash
# 启动后台任务
nohup opencode -p "ultrawork: 重构整个认证模块" -f json -c ~/myproject -q > /tmp/oc-task.log 2>&1 &
echo $! > /tmp/oc-task.pid

# 查看日志
tail -f /tmp/oc-task.log

# 检查状态
cat ~/myproject/.sisyphus/boulder.json | jq '.status'

# 停止任务
kill $(cat /tmp/oc-task.pid)
```

### 场景 5：从失败中恢复

```bash
# 检查失败原因
cat ~/myproject/.sisyphus/boulder.json | jq '.error'

# 重置任务状态（谨慎使用）
rm ~/myproject/.sisyphus/boulder.json

# 重新执行
opencode -p "ultrawork: 重新执行之前的任务" -f json -c ~/myproject -q
```

---

## 常用命令速查

| 命令 | 用途 |
|------|------|
| `opencode -p "..." -f json -c . -q` | 基本调用 |
| `opencode -p "ultrawork: ..." -f json -c . -q` | 复杂任务 |
| `cat .sisyphus/boulder.json \| jq .` | 查看任务状态 |
| `ls .sisyphus/plans/` | 列出所有计划 |
| `cat .sisyphus/plans/*.md` | 查看计划内容 |

---

## 下一步

- 了解 [Agent 路由策略](./agent-routing.md)
- 阅读完整的 [SKILL.md](../SKILL.md) 参考手册
- 探索 [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) 更多功能
