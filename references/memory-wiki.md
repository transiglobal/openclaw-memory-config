# Memory Wiki 参考

官方来源：https://docs.openclaw.ai/plugins/memory-wiki

Memory Wiki 是结构化知识层，不替代 active memory plugin。

## 当前配置选择

- `vaultMode`: `isolated`（默认）、`bridge`、`unsafe-local`
- `vault.scope`: `global`（默认）或 `agent`
- `vault.path`: global 默认 `~/.openclaw/wiki/main`；agent scope 默认父目录 `~/.openclaw/wiki`
- `vault.renderMode`: `native`（默认）或 `obsidian`
- `search.backend`: `shared`（默认）或 `local`
- `search.corpus`: `wiki`（默认）、`memory`、`all`

Bridge 模式可导入 active memory plugin 的 public artifacts、Dream reports、daily notes、memory root 和 event logs。Agent-scoped bridge 会按 artifact 所属 agent 过滤。

`vault.scope` 改变不会复制或拆分现有 vault。多 agent 的 per-agent vault 是同进程知识边界，不是操作系统安全边界。

## 可选项

- Obsidian 官方 CLI：`obsidian.useOfficialCli`
- URL 导入：`ingest.allowUrlIngest`
- compiled digest prompt：`context.includeCompiledDigestPrompt`
- unsafe-local：必须显式允许 private memory-core access 并限定 paths

这些能力涉及依赖、网络、私有文件或 prompt 体积，不能默认开启。

## CLI

- `openclaw wiki status`
- `openclaw wiki doctor`
- `openclaw wiki init`
- `openclaw wiki ingest <path>`
- `openclaw wiki compile`
- `openclaw wiki lint`
- `openclaw wiki search "query"`
- `openclaw wiki bridge import`
- `openclaw wiki obsidian status`
