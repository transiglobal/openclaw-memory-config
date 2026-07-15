# Dreaming 参考

官方来源：https://docs.openclaw.ai/concepts/dreaming

Dreaming 是 `memory-core` 的可选后台 consolidation，默认关闭。启用后由插件自动管理一个去重后的 full-sweep cron。

## 阶段与输出

顺序固定为 Light → REM → Deep：

- Light：排序和暂存短期材料，不写 `MEMORY.md`。
- REM：生成主题反思，不写 `MEMORY.md`。
- Deep：通过 score、recall count、query diversity 门槛后提升到 `MEMORY.md`。

机器状态位于 `memory/.dreams/`；人类审阅输出位于 `DREAMS.md`，可选阶段报告位于 `memory/dreaming/<phase>/YYYY-MM-DD.md`。Diary/report 不作为 promotion 来源。

## 配置

路径：`plugins.entries.memory-core.config.dreaming`

- `enabled`: 默认 false
- `frequency`: 默认 `0 3 * * *`
- `model`: 可选 Dream Diary 模型
- `phases.deep.maxPromotedSnippetTokens`: 默认 160

设置 `dreaming.model` 时必须启用 `plugins.entries.memory-core.subagent.allowModelOverride`，并建议用 `allowedModels` 限制规范的 `provider/model`。

## CLI

- `/dreaming status|on|off|help`
- `openclaw memory promote`
- `openclaw memory promote --apply`
- `openclaw memory promote-explain "关键词" --json`
- `openclaw memory rem-harness --json`
- `openclaw memory rem-backfill --path ./memory`
- `openclaw memory rem-backfill --rollback`
- `openclaw memory rem-backfill --rollback-short-term`

历史 backfill 先预览；`--stage-short-term` 只暂存候选，仍由 Deep 决定是否进入长期记忆。
