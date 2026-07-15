# Memory Search 参考

官方来源：

- https://docs.openclaw.ai/concepts/memory-search
- https://docs.openclaw.ai/reference/memory-config
- https://docs.openclaw.ai/concepts/memory

## 当前关键点

- 默认 builtin 引擎使用 per-agent SQLite、FTS/BM25 与可选向量检索。
- 通用 OpenAI-compatible embeddings 端点使用 `provider: "openai-compatible"`，避免继承 OpenAI chat 凭据。
- provider 也可指向兼容的 `models.providers.<id>`。
- 显式远程 provider 运行时不可用会 fail closed；`provider: "none"` 才是明确的 FTS-only。
- 改 provider、model、输入类型、sources、scope、chunking 或 tokenizer 后需要显式重建索引。
- API Key 使用 SecretRef；Codex OAuth 不能用于 embeddings。

## 官方默认值

- `chunking.tokens`: 400
- `chunking.overlap`: 80
- `query.maxResults`: 6
- `query.minScore`: 0.35
- `hybrid.enabled`: true
- `hybrid.vectorWeight`: 0.7
- `hybrid.textWeight`: 0.3
- `hybrid.candidateMultiplier`: 4
- `mmr.enabled`: false
- `mmr.lambda`: 0.7
- `temporalDecay.enabled`: false
- `temporalDecay.halfLifeDays`: 30

先用默认值。重复命中明显时启用 MMR；旧的 dated daily notes 经常压过新内容时启用 temporal decay。非日期文件和 `MEMORY.md` 不衰减。

## 可选能力

- 非对称 embeddings：设置 `queryInputType` 与 `documentInputType`。
- session transcripts：同时启用 `experimental.sessionMemory` 并在 `sources` 加入 `sessions`；注意存储、隐私和索引成本。
- QMD session search 还需 `memory.qmd.sessions.enabled: true`。
- multimodal：仅对 `extraPaths` 下的图像/音频生效，需 Gemini embedding-2；会上传二进制内容，应明确确认。
- local embeddings：需要官方 llama.cpp provider 插件；安装和原生构建必须单独安全审计与确认。

## 排障

- 无结果：`openclaw memory status`，必要时重建索引。
- 只有关键词：用 `openclaw memory status --deep` 检查 embedding。
- 显式 provider 不可用：修复认证/网络；不会静默退回 FTS。
- CJK 检索异常：重建 FTS 索引。
- `openclaw memory` 未注册：检查 active memory plugin，而不是直接重建。
