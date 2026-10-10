# mineru-selfhost-skill

[MinerU](https://github.com/opendatalab/MinerU) 自部署定制 skill 与 部署配方，基于 MinerU **4.0.11**。

> 相对官方 skill，除自部署适配外还有一处默认行为改变：flash+advanced 双档并发解析供 Agent 对照，以解决当前版本高档位的幻觉问题。



## 依赖

推荐使用 [uv](https://docs.astral.sh/uv/) 来管理 MinerU

## 快速开始

### 技能安装

标准 [Agent Skills](https://agentskills.io/) 风格，兼容任意常见 agent（[包括但不限于](https://agentskills.io/clients) OpenCode、Claude Code、Cursor、Gemini CLI 等）

```bash
npx skills add 123zdq/mineru-selfhost-skill --skill mineru-selfhosted --global --yes
```

或以任意方式复制 `skills/mineru-selfhosted/` 目录到你用的 agent 的 skills 目录即可：

| 客户端 | 全局目录 | 项目级目录 |
|---|---|---|
| 通用 | `~/.agents/skills/` | `.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.opencode/skills/` |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |

### 服务器

#### 安装

```bash
uv tool install -U "mineru[full]>=4.0,<5"
```

下载模型并验证：

```bash
mineru-kit models download --tier standard
mineru-kit models verify --tier standard
mineru-kit models show
```

配置文件默认位于 `$MINERU_HOME/config.yaml`，`MINERU_HOME` 默认 `~/.mineru`。

#### 启动

VLM 服务 （[显存预算 docs/EFFICIENCY.md](docs/EFFICIENCY.md)）：

```bash
mineru-kit vlm-server --engine vllm \
  --host 127.0.0.1 --port 30000 \
  --gpu-memory-utilization 0.70
```

API 服务：

```bash
mineru-kit api-server \
  --host 0.0.0.0 --port 8000 \
  --tier standard \
  --concurrency 16 \
  --preload-models \
  --vlm-server-url http://127.0.0.1:30000
  # --api-key <你的密钥>   ← 绑定 0.0.0.0 时务必启用，或放至带鉴权的反向代理之后
```

WebUI（可选）：

```bash
mineru-kit webui --server-name 0.0.0.0 --server-port 7860 \
  --api-url http://127.0.0.1:8000
```

### 客户端（agent）

同依赖 [uv](https://docs.astral.sh/uv/)

客户端可关闭遥测

```bash
uv tool install "mineru>=4.0,<5"

mineru config set parse_server.local.self_hosted_url http://<服务器IP>:8000
# mineru config set parse_server.local.self_hosted_api_key <服务器密钥>
mineru config set parse_server.local.mode self_hosted

mineru server status --json   # local parse_server 应为 healthy，tiers 含 advanced
mineru telemetry disable      # 关闭 doclib 遥测（默认开启并上报 mineru.net；服务器侧组件暂无遥测）
```

## License

MIT
