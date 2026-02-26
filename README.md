# opencode-orchestrator-skill

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](./SKILL.md)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![OpenCode](https://img.shields.io/badge/OpenCode-Compatible-orange.svg)](https://github.com/code-yeongyu/oh-my-opencode)

> OpenCode Agent 编排技能 - 用于任务分解、Agent 路由与状态管理

## 概述

本技能为 [ClawHub](https://github.com/clawhub) 提供 OpenCode CLI 的编排能力，实现：

- **任务分解**：将复杂任务拆分为原子执行单元
- **Agent 路由**：根据任务特征自动选择合适的 Agent（Prometheus → Momus → Atlas）
- **状态管理**：读取和解析 OpenCode 运行状态
- **后台任务**：管理长时间运行的任务
- **错误恢复**：处理失败任务的自动重试

## 安装

### 方式一：通过 ClawHub 安装（推荐）

```bash
# 安装技能
clawhub install imwxc/opencode-orchestrator-skill

# 验证安装
clawhub list | grep opencode-orchestrator
```

### 方式二：手动安装

```bash
# 克隆仓库
git clone https://github.com/imwxc/opencode-orchestrator-skill.git

# 进入目录
cd opencode-orchestrator-skill

# 复制到 ClawHub 技能目录
cp -r . ~/.clawhub/skills/opencode-orchestrator-skill
```

### 前置要求

- [OpenCode CLI](https://github.com/code-yeongyu/oh-my-opencode) >= 0.5.0
- [jq](https://stedolan.github.io/jq/)（可选，用于解析 JSON 状态）

## 快速开始

### 1. 基本调用

执行简单代码修改任务：

```bash
opencode -p "实现用户登录功能" -f json -c ~/myproject -q
```

### 2. 使用 ultrawork 模式

对于复杂任务，自动触发规划流程：

```bash
opencode -p "ultrawork: 重构数据访问层" -f json -c ~/myproject -q
```

### 3. 检查任务状态

```bash
# 查看当前任务状态
cat ~/myproject/.sisyphus/boulder.json | jq .

# 查看执行计划
cat ~/myproject/.sisyphus/plans/*.md
```

## 配置说明

### 环境变量

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `OPENCODE_MODEL` | 使用的模型 | `claude-3-5-sonnet` |
| `OPENCODE_TIMEOUT` | 任务超时时间（秒） | `300` |
| `SISYPHUS_DIR` | 状态文件目录 | `.sisyphus` |

### 项目配置

在项目根目录创建 `.opencode/config.json`：

```json
{
  "orchestrator": {
    "auto_route": true,
    "max_retries": 3,
    "timeout": 300
  },
  "agents": {
    "prometheus": {
      "enabled": true,
      "plan_format": "wave"
    },
    "momus": {
      "enabled": true,
      "parallel": false
    },
    "atlas": {
      "enabled": true,
      "verify": true
    }
  }
}
```

## 文档

详细使用指南：

- [快速开始指南](./docs/quick-start.md) - 5 分钟上手教程
- [Agent 路由策略](./docs/agent-routing.md) - 深入了解路由机制
- [SKILL.md](./SKILL.md) - 完整技能参考手册

## Agent 路由策略

```
用户请求 → ultrawork → Prometheus(规划) → Momus(分发) → Atlas(执行)
```

| 任务类型 | 路由目标 | 触发关键词 |
|---------|---------|-----------|
| 简单代码修改 | Atlas | 修复、优化、添加方法 |
| 多文件功能 | Prometheus → Atlas | 实现、创建、开发 |
| 架构变更 | Prometheus → Momus → Atlas | 重构、迁移、重新设计 |
| 批量任务 | ultrawork → Prometheus | ultrawork, ulw, plan |

## 状态文件结构

```
.sisyphus/
├── plans/           # 执行计划
│   └── <plan-name>.md
├── boulder.json     # 当前任务状态
├── sessions/        # 会话历史
└── notepads/        # 学习笔记
```

## 许可证

MIT License

## 相关项目

- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - OpenCode 核心
- [ClawHub](https://github.com/clawhub) - Skills 市场平台
