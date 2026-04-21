# Memory Search 参考文档

> 来源：OpenClaw 官方文档
> - https://docs.openclaw.ai/concepts/memory-search
> - https://docs.openclaw.ai/reference/memory-config
> - https://docs.openclaw.ai/concepts/memory-builtin
> - https://docs.openclaw.ai/concepts/memory

## Memory Search 工作原理

两路并行检索，加权合并：

1. **Vector Search（向量搜索）**：通过 embedding 找语义相似的笔记
2. **BM25 Keyword Search（关键词搜索）**：找精确匹配（ID、错误字符串、配置键）

如只有一路可用，另一路单独运行。

## Builtin Memory Engine（默认）

- 索引存储在 SQLite（per-agent）：`~/.openclaw/memory/<agentId>.sqlite`
- FTS5 全文索引 + 可选 sqlite-vec 向量扩展
- CJK 分词支持（中文/日文/韩文）
- 文件变更自动触发 reindex（1.5s debounce）
- 无额外依赖

## Embedding Provider 配置

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "model": "bge-large-zh-v1.5",
        "remote": {
          "baseUrl": "https://lapi.transiglobal.com/v1",
          "apiKey": "YOUR_KEY"
        }
      }
    }
  }
}
```

### 支持的 Provider

| Provider | ID | 自动检测 | 说明 |
|----------|----|---------|------|
| OpenAI | `openai` | ✅ | 默认 text-embedding-3-small |
| Gemini | `gemini` | ✅ | 支持多模态（图片+音频） |
| Voyage | `voyage` | ✅ | |
| Mistral | `mistral` | ✅ | |
| Ollama | `ollama` | ❌ 需显式设置 | 本地 |
| Local | `local` | ✅ 优先级最高 | GGUF 模型，~0.6GB |

自定义 OpenAI 兼容端点通过 `remote.baseUrl` + `remote.apiKey` 配置。

## 搜索参数详解

### 分块（Chunking）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `chunking.tokens` | 400 | 每块 token 数 |
| `chunking.overlap` | 80 | 块间重叠 |

### 基础查询

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `query.maxResults` | — | 最大返回结果数 |
| `query.minScore` | — | 最低相关度阈值 |

### 混合搜索（Hybrid）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `hybrid.enabled` | true | 启用混合搜索 |
| `hybrid.vectorWeight` | 0.7 | 向量权重 |
| `hybrid.textWeight` | 0.3 | 关键词权重 |
| `hybrid.candidateMultiplier` | 4 | 候选池倍数 |

### MMR 去重

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mmr.enabled` | false | 启用 MMR 重排序 |
| `mmr.lambda` | 0.7 | 0=最多样，1=最相关 |

### 时间衰减

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `temporalDecay.enabled` | false | 启用时间衰减 |
| `temporalDecay.halfLifeDays` | 30 | 半衰期天数 |

> MEMORY.md 等常驻文件不参与衰减。

## CLI 命令

```bash
openclaw memory status          # 查看索引状态
openclaw memory status --deep   # 深度检查（含 embedding）
openclaw memory index --force   # 强制重建索引
openclaw memory search "query"  # 命令行搜索
```

## 故障排查

- **无结果**：`openclaw memory status` 检查索引，空则 `openclaw memory index --force`
- **只有关键词匹配**：embedding provider 未配置，检查 `openclaw memory status --deep`
- **CJK 搜索失败**：`openclaw memory index --force` 重建 FTS
- **sqlite-vec 加载失败**：自动降级为进程内余弦相似度计算
