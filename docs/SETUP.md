# 安装与配置指南

> [!NOTE]
> 本文档由 AI 生成，面向 Agent 阅读（亦可供维护者参考）。
> 基于 MinerU **4.0.11** 源码与文档研究（opendatalab/MinerU，2026-10 快照）。

## 内容导航

| 文件 | 用途 |
|---|---|
| `skills/mineru-selfhosted/SKILL.md` | OpenCode 技能本体：agent 通过自建服务器读文档。默认 flash+advanced 双档、self_hosted 模式、禁用 `--remote` |
| `docs/EFFICIENCY.md` | 服务器效率压榨研究报告：三层并发模型、部署拓扑、参数调优清单、监控压测、瓶颈速查 |
| `docs/CONCURRENCY.md` | `--concurrency` 与内部并发机制深度研究，附双档融合方案评估 |
| `docs/CACHE.md` | 本地缓存与持久化研究：doclib 双层存储（SQLite+文件）、缓存键与命中、生命周期（无 TTL）、崩溃恢复、备份迁移 |

## 安装技能（agent 所在的机器）

本技能为标准 [Agent Skills](https://agentskills.io/) 格式，任何支持该格式的 agent（OpenCode、Claude Code、Cursor、Gemini CLI 等）均可使用。

skills CLI 一行安装：

```bash
npx skills add 123zdq/mineru-selfhost-skill --skill mineru-selfhosted --global --yes
```

或手动复制 `skills/mineru-selfhosted/` 目录到 agent 的 skills 目录。以 OpenCode 全局安装为例：

```powershell
# Windows PowerShell
mkdir -Force "$env:USERPROFILE\.config\opencode\skills" | Out-Null
Copy-Item -Recurse .\skills\mineru-selfhosted "$env:USERPROFILE\.config\opencode\skills\"
```

```bash
# Linux / macOS
mkdir -p ~/.config/opencode/skills
cp -r skills/mineru-selfhosted ~/.config/opencode/skills/
```

其他客户端的对应目录：Claude Code `~/.claude/skills/`；多客户端通用 `~/.agents/skills/`；项目级安装复制到项目的 `.opencode/skills/`（或对应客户端的项目级 skills 目录）。

> 注意：如果本机还装过官方 `mineru` skill（面向本地解析场景，行为规则不同），两个技能会同时被广告。建议只保留 `mineru-selfhosted`，避免 agent 混用规则。

## 技能生效前提（一次性配置）

agent 侧机器安装 MinerU CLI（base 包即可，无需任何 GPU 依赖），并把自建服务器注册为解析后端：

```bash
uv tool install "mineru>=4.0,<5"

mineru config set parse_server.local.self_hosted_url http://<服务器IP>:8000
mineru config set parse_server.local.self_hosted_api_key <服务器密钥>
mineru config set parse_server.local.mode self_hosted

mineru server status --json   # local parse_server 应为 healthy，tiers 含 advanced
mineru telemetry disable      # 关闭 doclib 遥测（默认开启并上报 mineru.net；服务器侧组件无遥测）
```

服务器侧部署命令见 [EFFICIENCY.md](EFFICIENCY.md) §2（推荐方案 B：VLM 分离部署）。

## 完成配置后的验收

```bash
mineru parse 任意测试文档.pdf --tier advanced --json
mineru parse 任意测试文档.pdf --tier flash --json
```

返回 `content.content` 与 `next_request` 即打通；之后 agent 读文件会自动走此链路。
