---
name: "openclaw-memory-config"
description: "按最新官方 Memory 与 Commitments 文档安全配置 OpenClaw 记忆系统"
---

# OpenClaw 记忆系统配置

依据当前 OpenClaw Schema 和官方文档做最小、可回滚的配置。默认使用 builtin memory；不得在未确认时替换第三方 memory slot、重建大索引、开启后台模型调用或创建对外通知。

官方基线：

- Memory overview: https://docs.openclaw.ai/concepts/memory
- Memory search: https://docs.openclaw.ai/concepts/memory-search
- Commitments: https://docs.openclaw.ai/concepts/commitments
- Dreaming: https://docs.openclaw.ai/concepts/dreaming
- Memory config: https://docs.openclaw.ai/reference/memory-config
- Memory Wiki: https://docs.openclaw.ai/plugins/memory-wiki

## 原则

- 先读当前 Schema 和配置，再写入；不同 OpenClaw 版本可能有字段差异。
- 使用 `openclaw config get/set/patch/validate`。所有写入先 `--dry-run`，成功后再执行正式写入。
- `config get` 只用于读取脱敏快照。不得提取、打印或复制明文 API Key。
- 凭据优先使用 SecretRef；不得要求用户在聊天中粘贴密钥。
- 变更 `plugins.entries` 后按 CLI 提示重启 Gateway；不要自行停止 Gateway。
- 改变 embedding provider、模型、输入类型、来源、chunking 或 tokenizer 会改变索引身份。先说明重建成本，再运行 `openclaw memory index --force`。
- 显式提醒和精确定时任务使用 scheduled tasks/cron；Commitments 只用于推断出的短期跟进。
- Memory、Commitments、Dreaming、Wiki 可以分别启用，不把它们误写成严格串行依赖。

## 0. 预检与备份

先执行只读检查：

```bash
openclaw --version
openclaw config validate --json
openclaw config get memory --json
openclaw config get agents.defaults.memorySearch --json
openclaw config get agents.defaults.compaction.memoryFlush --json
openclaw config get plugins.slots.memory --json
openclaw config get plugins.entries.memory-core --json
openclaw config get commitments --json
openclaw config get agents.defaults.heartbeat --json
openclaw plugins list
openclaw plugins doctor
```

缺失路径可视为未配置，不是失败。需要写配置前创建配置备份：

```bash
openclaw backup create --only-config --verify
```

再用 `openclaw config schema` 或当前版本官方文档确认目标字段存在。若当前版本不支持 `commitments`、目标 provider 或 Wiki 字段，停止并建议先升级 OpenClaw，不写未知字段。

## 1. 明确记忆层级

向用户说明：

- `MEMORY.md`：精炼的长期事实、偏好和决策；会进入会话启动上下文，需控制体积。
- `memory/YYYY-MM-DD.md`：每日工作层；用于详细日志和近期上下文。
- `DREAMS.md`：可选的人类审阅面；不是普通长期记忆来源。
- Commitments：短期、会话与渠道绑定的待跟进状态，不写入 `MEMORY.md`。
- Wiki：独立的结构化知识层，不替代 active memory plugin。

涉及权限、时效、交接、解锁条件或来源权威的记忆，应同时写明“何时可以行动、何时失效、禁止什么、谁有授权”。记忆只保存上下文，不代替审批、沙箱或调度器。

## 2. 选择 active memory plugin

`memory_search` / `memory_get` 由当前 memory slot 插件提供。默认插件是内置 `memory-core`，但现场可能使用 LanceDB、Honcho 等第三方插件。

判断：

- slot 已是 `memory-core`：保留。
- slot 为空且 builtin 正常：不必为了显式化而写配置。
- slot 是第三方插件：报告其能力和迁移影响；只有用户明确同意才切回 `memory-core`。
- 选择 QMD：读取官方 QMD 文档并单独设计；不要沿用 builtin embedding 配置。

切换到 builtin + memory-core 时：

1. 先备份。
2. 用 `openclaw plugins enable memory-core` 启用内置插件，避免手工覆盖 `plugins.allow` 数组。
3. 对以下写入先加 `--dry-run`，通过后去掉该参数正式执行：

```bash
openclaw config set plugins.slots.memory memory-core --dry-run
openclaw config set memory.backend builtin --dry-run
```

4. 按 CLI 提示使用 `openclaw gateway restart`。
5. 运行 `openclaw plugins doctor` 和该插件提供的 memory CLI/工具验证。

不得在未确认时禁用或卸载原 memory 插件。

## 3. Memory Search

### 3.1 默认 trapi 路径

传米 trapi 是通用 OpenAI-compatible embeddings 端点，应使用：

- provider: `openai-compatible`
- base URL: `https://lapi.transiglobal.com/v1`
- model: `bge-large-zh-v1.5`
- SecretRef 环境变量: `TRAPI_API_KEY`

若现有 `agents.defaults.memorySearch.remote.apiKey` 已是可解析 SecretRef，保留它。若只有 `models.providers.trapi.apiKey`：

- 它是 SecretRef 时，可在用户确认后复用同一引用。
- 它是明文时，不读取或回显；引导迁移到 `TRAPI_API_KEY` SecretRef。

确保环境变量对 Gateway 服务可见，而不只是当前交互 shell。使用最小 JSON5 patch；先 dry-run，再正式写入：

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "openai-compatible",
        model: "bge-large-zh-v1.5",
        remote: {
          baseUrl: "https://lapi.transiglobal.com/v1",
          apiKey: {
            source: "env",
            provider: "default",
            id: "TRAPI_API_KEY"
          }
        }
      }
    }
  }
}
```

```bash
openclaw config patch --file ./memory-search.patch.json5 --dry-run
openclaw config patch --file ./memory-search.patch.json5
openclaw config validate --json
```

不要默认覆盖官方检索参数。当前默认值为：chunk 400/80、maxResults 6、minScore 0.35、hybrid 0.7/0.3、candidateMultiplier 4，MMR 和 temporal decay 默认关闭。只有出现重复结果、旧记录压过新记录或精确/语义权重失衡时才按 `references/memory-search.md` 调整。

对需要 `input_type` 的非对称 embedding 模型，按供应商要求设置 `queryInputType` 与 `documentInputType`，并重建索引。

### 3.2 验证

若 active plugin 注册了官方 memory CLI：

```bash
openclaw memory status --deep
openclaw memory index --force
openclaw memory search "一个确实存在于记忆中的关键词"
```

只在索引身份改变或索引损坏时强制重建。若 `openclaw memory` 未注册，先检查 active memory plugin 与 Gateway 状态；第三方插件使用其自带诊断工具，不把“未知命令”误判为索引为空。

测试应选择当前 workspace 中已存在、非敏感的关键词，并检查命中来源和相关性。

## 4. Automatic memory flush

compaction 前的 memory flush 默认开启。读取 `agents.defaults.compaction.memoryFlush`：

- 未配置：保留默认开启。
- 被明确关闭：说明上下文压缩前可能丢失尚未落盘的重要信息，再询问是否恢复。
- 仅在用户要求降低主模型成本时设置专用 `model`；该模型覆盖只作用于 memory-flush turn，不继承会话 fallback 链。

不要为了“显式化默认值”写冗余配置。

## 5. Dreaming

Dreaming 属于 `memory-core`，默认关闭。它自管一个去重后的 full-sweep cron；不要再创建第二个外部 Dreaming cron。

启用前确认：

- active memory plugin 为 `memory-core`
- 接受后台模型调用和长期记忆写入
- frequency、timezone 与现有维护窗口不冲突
- 是否需要可选 Dream Diary 模型覆盖

基础配置路径：

```bash
openclaw config set plugins.entries.memory-core.config.dreaming.enabled true --strict-json --dry-run
openclaw config set plugins.entries.memory-core.config.dreaming.enabled true --strict-json
```

默认频率是 `0 3 * * *`。仅在用户指定维护窗口时设置 `frequency` 和 `timezone`。若设置 `dreaming.model`，还必须：

- `plugins.entries.memory-core.subagent.allowModelOverride: true`
- 建议同时限制 `plugins.entries.memory-core.subagent.allowedModels`
- 使用规范的 `provider/model` 名称

插件配置变化后按 CLI 提示重启。验证：

```text
/dreaming status
```

可选只读/预览检查：

```bash
openclaw memory status --deep
openclaw memory promote
openclaw memory promote-explain "关键词" --json
openclaw memory rem-harness --json
```

不要把 Light、REM、Deep 当作三个独立 cron；执行顺序是 Light → REM → Deep。历史回填先预览，写入前确认；回填 staged candidates 仍需 Deep 才能进入 `MEMORY.md`。

## 6. Inferred Commitments

Commitments 默认关闭，是“记忆与自动化之间”的短期跟进层。开启前必须说明并确认：

- 每个合格对话后可能增加一次隐藏 LLM 提取，产生模型成本。
- 提取会读取判断跟进所需的近期对话。
- 记录保存在本地 OpenClaw 状态，绑定原 agent、session、channel 与 delivery target。
- 到期项通过 heartbeat 投递；若 heartbeat `target: "none"`，只保留内部状态，不外发。
- 到期 heartbeat turn 不带 OpenClaw 工具，且不会重放原始对话。
- Commitments 不是精确提醒；“3 点提醒我”等需求必须走 scheduled tasks。

用户确认后，默认每天每个 agent session 最多投递 3 个：

```bash
openclaw config set commitments.enabled true --strict-json --dry-run
openclaw config set commitments.maxPerDay 3 --strict-json --dry-run
openclaw config set commitments.enabled true --strict-json
openclaw config set commitments.maxPerDay 3 --strict-json
openclaw config validate --json
```

检查 heartbeat 是否运行、投递目标和安静时段是否符合用户预期。不要为了测试而主动向外发送消息。

管理与验证：

```bash
openclaw config get commitments --json
openclaw commitments
openclaw commitments --all
openclaw commitments --agent main
openclaw commitments --status snoozed
openclaw commitments dismiss cm_abc123
```

Commitment 不会在创建后立即投递；最早时间至少是创建后的一个 heartbeat interval。功能测试需用户同意，使用一个非紧急、非精确定时的自然跟进场景，等待 heartbeat 后核对同一 agent/channel 是否收到自然 check-in；无需外发测试时只做配置和列表验证。

关闭：

```bash
openclaw config set commitments.enabled false --strict-json
```

关闭功能不会把 commitments 转成长期记忆。

## 7. Memory Wiki（可选）

仅在用户需要结构化知识库时配置。先选择：

- `vaultMode`: `isolated`（默认）、`bridge`、`unsafe-local`
- `vault.scope`: `global`（默认）或 `agent`
- `vault.renderMode`: `native`（默认）或 `obsidian`
- `search.corpus`: `wiki`（默认）、`memory`、`all`

单 agent、独立知识库优先使用最小 isolated 配置。多 agent 需要隔离时使用 `vault.scope: "agent"`；这只是同进程知识边界，不是操作系统安全边界。切换 scope 不会自动迁移旧 vault，移动前必须备份并确认。

Bridge 模式才启用 `bridge.enabled` 并选择 memory artifacts、Dream reports、daily notes 与 event logs。不要在 isolated 模式下误以为 bridge 索引开关会产生导入。

Obsidian CLI、URL ingest、unsafe-local、compiled digest prompt 和每日维护 cron 均为可选能力，逐项确认；不要默认安装依赖、开放 URL 导入或向飞书发送维护报告。

验证：

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki lint
openclaw wiki search "已导入内容中的关键词"
```

只有用户提供并确认内容路径后才执行 `openclaw wiki ingest <path>` 和 `openclaw wiki compile`。

## 8. 最终验证与报告

执行：

```bash
openclaw config validate --json
openclaw plugins doctor
openclaw gateway status --json --require-rpc
```

按启用模块追加：

- Memory Search：provider/模型/索引身份、深度状态、真实关键词搜索。
- Dreaming：`/dreaming status`、managed sweep 是否存在、下次运行时间。
- Commitments：配置、heartbeat 投递条件、pending/all 列表；外发测试仅在用户同意后进行。
- Wiki：status、doctor、lint 和真实内容搜索。

报告必须列出：变更前后、备份位置、实际写入路径、是否重启、索引是否重建、后台模型成本、外发条件、跳过项和未解决告警。
