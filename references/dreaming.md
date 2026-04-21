# Dreaming 参考文档

> 来源：OpenClaw 官方文档
> - https://docs.openclaw.ai/concepts/dreaming

## 概述

Dreaming 是后台记忆整理系统，将短期信号整理为持久记忆。**默认禁用，需手动开启。**

## 三个阶段

| 阶段 | 目的 | 持久写入 |
|------|------|---------|
| **Light** | 排序和暂存短期材料 | 否 |
| **Deep** | 评分并提升到长期记忆 | ✅ 写入 MEMORY.md |
| **REM** | 反思主题和重复模式 | 否 |

## 配置

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 3 * * *",
            "timezone": "Asia/Shanghai"
          }
        }
      }
    }
  }
}
```

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `enabled` | false | 是否启用 |
| `frequency` | `0 3 * * *` | 执行频率（cron 表达式） |
| `timezone` | 系统时区 | 时区 |

## 输出文件

- `DREAMS.md`（或 `dreams.md`）：人类可读的梦境日记
- `memory/dreaming/<phase>/YYYY-MM-DD.md`：阶段报告
- `memory/.dreams/`：机器状态（recall store、信号、检查点）

## Deep 排序信号（6个权重）

| 信号 | 权重 | 说明 |
|------|------|------|
| Frequency | 0.24 | 短期信号累计次数 |
| Relevance | 0.30 | 平均检索质量 |
| Query diversity | 0.15 | 不同查询/天上下文 |
| Recency | 0.15 | 时间衰减新鲜度 |
| Consolidation | 0.10 | 跨天重复强度 |
| Conceptual richness | 0.06 | 概念标签密度 |

## CLI 命令

```bash
/dreaming status    # 查看状态
/dreaming on        # 启用
/dreaming off       # 禁用

openclaw memory promote          # 预览待提升项
openclaw memory promote --apply  # 执行提升
openclaw memory promote-explain "关键词"  # 解释某候选
openclaw memory rem-harness      # 预览 REM 反思
```

## 关键点

- Deep 阶段需要 `minScore`、`minRecallCount`、`minUniqueQueries` 都通过才提升
- 提升前会从原始文件重新验证片段，已删除的跳过
- Dreaming 生成的日记条目不参与短期提升
