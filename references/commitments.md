# Inferred Commitments 参考

官方来源：https://docs.openclaw.ai/concepts/commitments

Commitments 是可选的短期跟进记忆，默认关闭。它不是长期记忆，也不是精确定时提醒。

## 运行方式

- Agent 回复后可能触发隐藏 LLM 提取，工具禁用。
- 高置信度候选保存 agent id、session key、原 channel/target、due window 和建议 check-in。
- 到期后通过同一 agent/channel 的 heartbeat 处理。
- heartbeat `target: "none"` 时不外发。
- 到期 turn 不重放原对话，且不带 OpenClaw 工具。
- 最早投递时间至少是创建后的一个 heartbeat interval。

## 配置

```json
{
  "commitments": {
    "enabled": true,
    "maxPerDay": 3
  }
}
```

`maxPerDay` 是每个 agent session 的 rolling-day 投递上限，默认 3。

## 适用边界

Commitments：

- “我明天有面试”
- “我昨晚完全没睡”
- 对话形成了明确但未精确定时的开放跟进

Scheduled tasks：

- “下午 3 点提醒我”
- “20 分钟后通知我”
- “每个工作日运行报告”

## 管理

- `openclaw commitments`
- `openclaw commitments --all`
- `openclaw commitments --agent <id>`
- `openclaw commitments --status pending|sent|dismissed|snoozed|expired`
- `openclaw commitments dismiss <id>`

启用会增加后台模型使用，并读取判断所需的近期对话。Stored commitments 是本地 operational state。测试外发前需用户明确同意。
