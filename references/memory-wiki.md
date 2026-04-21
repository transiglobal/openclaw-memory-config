# Memory Wiki 参考文档

> 来源：OpenClaw 官方文档
> - https://docs.openclaw.ai/plugins/memory-wiki

## 概述

Memory Wiki 将持久记忆编译为结构化知识库，不替代 active memory plugin。

## Vault 模式

| 模式 | 说明 |
|------|------|
| `isolated` | 独立 vault，不依赖 memory-core |
| `bridge` | 从 memory-core 读取公开 artifact |
| `unsafe-local` | 本地文件系统直接访问（实验性） |

## Vault 目录结构

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/      # 实体（人、系统、项目）
  concepts/      # 概念、模式、策略
  syntheses/     # 编译摘要
  sources/       # 导入的原始材料
  reports/       # 仪表板报告
  _attachments/
  _views/
  .openclaw-wiki/
```

## 配置（isolated 模式）

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
            "enabled": false
          },
          "search": {
            "backend": "shared",
            "corpus": "all"
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

## 关键配置项

| 配置项 | 说明 |
|--------|------|
| `vaultMode` | `isolated` / `bridge` / `unsafe-local` |
| `vault.renderMode` | `native` 或 `obsidian` |
| `search.backend` | `shared`（共享记忆搜索）或 `local` |
| `search.corpus` | `wiki` / `memory` / `all` |
| `render.createBacklinks` | 生成相关链接 |
| `render.createDashboards` | 生成仪表板页面 |
| `ingest.autoCompile` | 自动编译 |

## Agent 工具

| 工具 | 说明 |
|------|------|
| `wiki_status` | vault 模式、健康状态 |
| `wiki_search` | 搜索 wiki 页面 |
| `wiki_get` | 读取 wiki 页面 |
| `wiki_apply` | 窄范围修改（synthesis/metadata） |
| `wiki_lint` | 结构检查、provenance、矛盾、问题 |

## CLI 命令

```bash
openclaw wiki status      # 状态
openclaw wiki doctor      # 诊断
openclaw wiki init        # 初始化
openclaw wiki ingest <path>  # 导入内容
openclaw wiki compile     # 编译
openclaw wiki lint        # 检查
openclaw wiki search "query"  # 搜索
```

## Bridge 模式可索引的内容

- exported memory artifacts
- dream reports
- daily notes
- memory root files
- memory event logs
