# openclaw-memory-config

OpenClaw 技能（Skill）：一键配置 OpenClaw 记忆系统。

## 简介

本技能帮助用户在 OpenClaw 实例上配置完整的记忆系统，包含三大模块：

1. **Memory Search** — 基于向量的语义记忆搜索
2. **Dreaming** — 夜间自动记忆整理（短期 → 长期）
3. **Memory Wiki** — 结构化知识库（isolated 模式）

## Embedding 服务

默认使用传米科技 trapi 的 embedding 端点：

| 项目 | 值 |
|------|-----|
| Provider | `openai`（OpenAI 兼容） |
| Base URL | `https://lapi.transiglobal.com/v1` |
| 模型 | `bge-large-zh-v1.5` |

复用 trapi API Key，无需单独配置 embedding provider。

## 使用方式

### 安装技能

```bash
openclaw skill install openclaw-memory-config.skill
```

### 配置

在 OpenClaw 对话中说：

> "配置记忆"

技能会引导完成：
1. **Memory Search**：配置 embedding → 构建索引 → 测试搜索
2. **Dreaming**：启用夜间记忆整理（每天 4:30 自动运行）
3. **Memory Wiki**：确认内容目录 → 初始化 vault → 导入内容 → 创建每日维护定时任务

### 安全说明

- API Key 必须由用户提供，技能中不存储默认密钥
- Memory Wiki 默认使用 isolated 模式，数据完全独立
- 所有配置通过 `gateway config.patch` 标准方式写入

## 许可证

MIT
