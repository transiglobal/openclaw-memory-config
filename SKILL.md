---
name: openclaw-memory-config
description: |
  配置 OpenClaw 记忆系统：Memory Search（向量搜索）、Dreaming（记忆整理）、Memory Wiki（知识库）。触发词："配置记忆"、"配置 memory"、"memory search"、"配置搜索"、"配置 dreaming"、"配置梦境"、"配置 wiki"、"memory wiki"、"记忆配置"。使用本机已有 trapi embedding 端点，为用户提供完整的记忆系统初始化流程。
---

# OpenClaw 记忆系统配置

一键配置 OpenClaw 记忆系统，包含 Memory Search（向量语义搜索）、Dreaming（夜间记忆整理）、Memory Wiki（知识库）三大模块。

## Embedding 服务信息（默认）

- **Provider**：`openai`（OpenAI 兼容格式）
- **Base URL**：`https://lapi.transiglobal.com/v1`
- **默认模型**：`bge-large-zh-v1.5`
- **API Key 来源**：优先复用本地已配置的 trapi provider；如未配置则要求用户提供

## 前置条件

1. OpenClaw 已安装并运行
2. 本机配置文件为 `openclaw.json`，使用 `gateway config.patch` 写入
3. Embedding API Key：
   - **优先**：检查本地 `models.providers.trapi.apiKey` 是否已配置，已配置则复用
   - **兜底**：如 trapi 未配置，向用户提供两种选择：
     - a) 提供 trapi API Key（使用默认 trapi embedding 端点）
     - b) 提供其他 embedding provider 信息（provider、baseUrl、apiKey、model）

---

## 模块 1：Memory Search（语义记忆搜索）

### 步骤 1：配置 Embedding

**检测 trapi**：先通过 `gateway config.get` 检查 `models.providers.trapi` 是否已存在：

- **已配置 trapi**：直接从配置中提取 `apiKey`，无需用户额外提供，跳到下一步
- **未配置 trapi**：向用户说明两种选择：
  - a) 提供 trapi API Key → 使用默认配置（baseUrl + bge-large-zh-v1.5）
  - b) 提供自定义 embedding provider 信息（需确认 provider、baseUrl、apiKey、model）

如用户拒绝提供任何 Key，终止流程。

使用 `gateway config.patch` 配置 `agents.defaults.memorySearch`：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "model": "bge-large-zh-v1.5",
        "remote": {
          "baseUrl": "https://lapi.transiglobal.com/v1",
          "apiKey": "<用户提供的API_KEY>"
        },
        "chunking": {
          "tokens": 200,
          "overlap": 30
        },
        "query": {
          "maxResults": 10,
          "minScore": 0.35,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.85,
            "textWeight": 0.15,
            "candidateMultiplier": 8,
            "mmr": {
              "enabled": true,
              "lambda": 0.5
            },
            "temporalDecay": {
              "enabled": true,
              "halfLifeDays": 30
            }
          }
        }
      }
    }
  }
}
```

### 步骤 2：搜索参数说明

配置中已包含以下优化参数（均已在上方 JSON 中预设，无需额外修改）：

| 参数组 | 参数 | 值 | 说明 |
|--------|------|-----|------|
| **分块** | `chunking.tokens` | 200 | 每个文本块 200 token |
| | `chunking.overlap` | 30 | 块间重叠 30 token，避免截断语义 |
| **基础查询** | `query.maxResults` | 10 | 最多返回 10 条结果 |
| | `query.minScore` | 0.35 | 最低相关度阈值，低于此分数不返回 |
| **混合搜索** | `hybrid.enabled` | true | 同时使用向量搜索 + BM25 关键词搜索 |
| | `hybrid.vectorWeight` | 0.85 | 向量搜索权重 85%（语义匹配优先） |
| | `hybrid.textWeight` | 0.15 | 关键词搜索权重 15%（精确匹配补充） |
| | `hybrid.candidateMultiplier` | 8 | 候选池倍数，越大召回越全但越慢 |
| **MMR 去重** | `mmr.enabled` | true | 减少重复结果，不同话题覆盖更广 |
| | `mmr.lambda` | 0.5 | 0=最多样，1=最相关，0.5 平衡 |
| **时间衰减** | `temporalDecay.enabled` | true | 旧记忆逐渐降权，新记忆优先展示 |
| | `temporalDecay.halfLifeDays` | 30 | 每 30 天权重减半，MEMORY.md 等常驻文件不衰减 |

> 以上参数适用于大多数场景，用户一般无需调整。如需微调可在配置完成后通过 `gateway config.patch` 修改。

配置写入后，立即执行：

```bash
openclaw memory index --force
```

### 步骤 3：测试搜索

索引构建完成后，用 `memory_search` 测试是否生效：

1. 从用户的 memory 目录中找到已有内容的关键词
2. 执行 `memory_search` 搜索
3. 确认能返回相关结果，报告搜索质量

**测试关键词示例**（根据实际记忆内容选择）：
- 搜索 "传米科技" → 应找到公司相关信息
- 搜索 "Tailscale" → 应找到网络配置信息
- 搜索最近的日期 → 应找到当天的日记

### 验证报告

| 测试项 | 关键词 | 结果 |
|--------|--------|------|
| 语义搜索 | <实际关键词> | ✅/❌ |
| 索引状态 | `openclaw memory status` | ✅/❌ |

---

## 模块 2：Dreaming（夜间记忆整理）

### 配置

使用 `gateway config.patch` 启用 dreaming：

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "30 4 * * *",
            "timezone": "Asia/Shanghai"
          }
        }
      }
    }
  }
}
```

- **频率**：每天凌晨 4:30（在安全巡检和系统升级之后）
- **三个阶段**：Light（排序）→ REM（反思）→ Deep（提升到长期记忆）
- **输出**：`memory/dreaming/` 下的阶段报告 + `DREAMS.md` 日记
- **长期记忆**：Deep 阶段会将高价值内容写入 `MEMORY.md`

### 验证

运行 `/dreaming status` 确认 dreaming 已启用并显示下次执行时间。

---

## 模块 3：Memory Wiki（知识库）

### 前置确认

**必须先向用户确认**：是否有需要放入 Wiki 的内容目录。

- **有**：继续配置
- **没有**：跳过此模块，告知用户后续可随时配置

### 步骤 1：配置 Wiki

使用 `gateway config.patch` 启用 memory-wiki（isolated 模式）：

```json
{
  "plugins": {
    "entries": {
      "memory-wiki": {
        "enabled": true,
        "config": {
          "vaultMode": "isolated",
          "vault": {
            "path": "~/.openclaw/wiki/main",
            "renderMode": "obsidian"
          },
          "bridge": {
            "enabled": false,
            "readMemoryArtifacts": true,
            "indexDreamReports": true,
            "indexDailyNotes": true,
            "indexMemoryRoot": true,
            "followMemoryEvents": true
          },
          "search": {
            "backend": "shared",
            "corpus": "all"
          },
          "context": {
            "includeCompiledDigestPrompt": false
          },
          "render": {
            "preserveHumanBlocks": true,
            "createBacklinks": true,
            "createDashboards": true
          },
          "ingest": {
            "autoCompile": true,
            "maxConcurrentJobs": 1
          },
          "obsidian": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

### 步骤 2：导入内容

用户提供内容目录后，使用 CLI 导入：

```bash
openclaw wiki init
openclaw wiki ingest <用户提供的内容目录>
openclaw wiki compile
```

### 步骤 3：创建每日维护定时任务

创建定时任务（参考已有配置），每天凌晨 5:00 执行 Wiki 维护：

```
cron add:
  name: "Wiki 每日维护（Sources 同步 + Lint）"
  schedule: "0 5 * * *" (Asia/Shanghai)
  sessionTarget: "isolated"
  payload:
    kind: "agentTurn"
    message: |
      执行每日 Wiki 维护任务：
      1. 执行 wiki_lint 检查
      2. 自动修复可修复的问题（缺失 frontmatter、匹配 sourceIds）
      3. 无法自动修复的问题（broken wikilinks）跳过
      4. 再跑一次 wiki_lint 确认
      5. 汇报结果
    toolsAllow: ["exec", "read", "edit", "write", "wiki_lint", "wiki_apply", "wiki_search", "wiki_get"]
    timeoutSeconds: 600
  delivery:
    mode: "announce"
    channel: "feishu"
```

### 验证

- `wiki_status` 确认 vault 模式和健康状态
- `wiki_search` 测试能否搜到导入的内容
- `wiki_lint` 检查结构完整性

---

## 完整验证报告

全部模块配置完成后，输出汇总表：

| 模块 | 配置项 | 状态 |
|------|--------|------|
| Memory Search | Embedding + 索引 | ✅/❌ |
| Memory Search | 语义搜索测试 | ✅/❌ |
| Dreaming | 启用 + cron | ✅/❌ |
| Memory Wiki | Vault 初始化 | ✅/❌（可能跳过） |
| Memory Wiki | 内容导入 | ✅/❌（可能跳过） |
| Memory Wiki | 定时维护任务 | ✅/❌（可能跳过） |

## 故障排查

- **索引为空**：`openclaw memory index --force` 强制重建
- **只有关键词匹配**：embedding provider 未配好，检查 API Key 和网络
- **CJK 搜索失败**：`openclaw memory index --force` 重建 FTS
- **Dreaming 未运行**：检查 `/dreaming status` 和 cron 任务
- **Wiki bridge 无数据**：isolated 模式下 bridge 关闭是正常的
- **Wiki ingest 失败**：确认目录路径和文件权限
