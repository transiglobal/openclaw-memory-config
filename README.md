# openclaw-memory-config

用于配置和诊断 OpenClaw 记忆系统的 Workspace Skill，按当前 OpenClaw Schema 和官方文档执行最小、可回滚的变更。

## 覆盖能力

1. **Memory Core / active memory plugin**：识别当前 memory slot，避免误覆盖 LanceDB、Honcho 等第三方插件。
2. **Memory Search**：配置 builtin SQLite 混合检索、Embedding、索引和搜索质量。
3. **Automatic memory flush**：检查 compaction 前自动落盘，减少上下文压缩造成的记忆丢失。
4. **Dreaming**：启用 `memory-core` 的后台整理、Dream Diary 和长期记忆提升。
5. **Inferred Commitments**：配置短期、会话和渠道绑定的自然跟进。
6. **Memory Wiki**：按需配置 isolated、bridge 或 per-agent vault。

## 默认 Embedding

- Provider：`openai-compatible`
- Base URL：`https://lapi.transiglobal.com/v1`
- Model：`bge-large-zh-v1.5`
- Credential：`TRAPI_API_KEY` SecretRef

不会要求用户在聊天中粘贴明文 API Key。已有可解析 SecretRef 会优先保留。

## 使用方式

在 OpenClaw 对话中说：

> 配置记忆

技能会先读取当前版本、Schema、active memory plugin、Memory Search、Dreaming、Commitments、Heartbeat 和 Wiki 状态，再给出最小变更方案。

所有配置写入遵循：

1. 创建配置备份。
2. 使用 `openclaw config set/patch --dry-run` 验证。
3. 正式写入并运行 `openclaw config validate`。
4. 仅在 CLI 明确提示时重启 Gateway。
5. 使用真实但非敏感的记忆内容验证检索质量。

## Commitments 边界

Commitments 默认关闭。启用前会说明隐藏 LLM 提取的模型成本、近期对话读取范围和 Heartbeat 外发条件。

- “我明天有面试”这类自然跟进可使用 Commitments。
- “下午 3 点提醒我”或周期任务必须使用 scheduled tasks/cron。
- Heartbeat `target: "none"` 时，到期 Commitment 只保留内部状态，不会外发。

## 安全约束

- 不在未确认时替换第三方 memory slot。
- 不直接覆盖 `plugins.allow` 数组。
- 不默认改写官方搜索参数或强制重建大索引。
- Dreaming 使用 `memory-core` 自管的去重 cron，不额外创建重复任务。
- Wiki 的 URL ingest、unsafe-local、Obsidian CLI、compiled digest prompt 和通知任务均需单独确认。
- 不执行 `gateway stop`；需要时使用 `openclaw gateway restart`。

## 官方文档

- [Memory overview](https://docs.openclaw.ai/concepts/memory)
- [Memory search](https://docs.openclaw.ai/concepts/memory-search)
- [Inferred commitments](https://docs.openclaw.ai/concepts/commitments)
- [Dreaming](https://docs.openclaw.ai/concepts/dreaming)
- [Memory configuration reference](https://docs.openclaw.ai/reference/memory-config)
- [Memory Wiki](https://docs.openclaw.ai/plugins/memory-wiki)

## 许可证

MIT
