# opencode-orchestrator-skill

> OpenCode Agent 编排技能 - 用于任务分解、Agent 路由与状态管理

## 概述

本技能为 [ClawHub](https://github.com/clawhub) 提供 OpenCode CLI 的编排能力，实现：

- **任务分解**：将复杂任务拆分为原子执行单元
- **Agent 路由**：根据任务特征自动选择合适的 Agent（Prometheus → Momus → Atlas）
- **状态管理**：读取和解析 OpenCode 运行状态
- **后台任务**：管理长时间运行的任务
- **错误恢复**：处理失败任务的自动重试

## 安装

```bash
# 通过 ClawHub 安装
clawhub install imwxc/opencode-orchestrator-skill
```

## 快速开始

### 基本调用

```bash
opencode -p "实现用户登录功能" -f json -c ~/myproject -q
```

### 使用 ultrawork 模式

```bash
opencode -p "ultrawork: 重构数据访问层" -f json -c ~/myproject -q
```

### 检查任务状态

```bash
cat ~/myproject/.sisyphus/boulder.json | jq .
```

## 文档

详细使用指南请参阅 [SKILL.md](./SKILL.md)

## Agent 路由策略

```
用户请求 → ultrawork → Prometheus(规划) → Momus(分发) → Atlas(执行)
```

| 任务类型 | 路由目标 |
|---------|---------|
| 简单代码修改 | Atlas |
| 多文件功能 | Prometheus → Atlas |
| 架构变更 | Prometheus → Momus → Atlas |

## 状态文件结构

```
.sisyphus/
├── plans/           # 执行计划
├── boulder.json     # 当前任务状态
├── sessions/        # 会话历史
└── notepads/        # 学习笔记
```

## 许可证

MIT License

## 相关项目

- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - OpenCode 核心
- [ClawHub](https://github.com/clawhub) - Skills 市场平台
