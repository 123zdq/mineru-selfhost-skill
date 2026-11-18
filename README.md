# mineru-selfhost-skill

[MinerU 4.2.2](https://github.com/opendatalab/MinerU) 自部署定制 skill 与部署配方。

相对官方 skill 的改进：
* 自部署适配
* 由 flash 与 advanced 两档并发解析供 Agent 对照，以克服当前 VLM 档位的幻觉问题

## 快速开始

服务端和客户端均推荐用 [uv](https://docs.astral.sh/uv/) 管理 MinerU。

### 技能安装

适用于任意 [支持标准 Agent Skills 的客户端](https://agentskills.io/clients)。

```bash
npx skills add 123zdq/mineru-selfhost-skill --skill doc-parse-mineru-selfhosted --global --yes
```

或以任意方式将 `skills/doc-parse-mineru-selfhosted/` 复制到下表中的目录。

| 客户端 | 全局目录 | 项目级目录 |
|---|---|---|
| 通用 | `~/.agents/skills/` | `.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.opencode/skills/` |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |

> 技能本身暂不包含以下章节内容：客户端与服务端的安装与配置，需要手动/让agent自动安装、配置

### 服务器

#### 安装

```bash
uv tool install ninja
uv tool install -U "mineru[full]>=4.0,<5"
```

下载并验证模型。

```bash
mineru-kit models download --tier standard
mineru-kit models verify --tier standard
mineru-kit models show
```

配置文件为 `$MINERU_HOME/config.yaml`，默认路径为 `~/.mineru/config.yaml`。

#### 启动

```bash
mineru-kit api-server \
  --host 0.0.0.0 --port 8000 \
  --tier advanced \
  --concurrency 32 \
  --preload-models
```

如需鉴权，启动 API 服务和 WebUI 时均加 `--api-key "<密钥>"`。

另开终端检查，响应应列出四个档位。

```bash
curl http://0.0.0.0:8000/v1/tiers
```

启动 WebUI（可选）。

```bash
mineru-kit webui --server-name 0.0.0.0 --server-port 7860 --api-url http://0.0.0.0:8000
```

#### 设置开机自启

将 [mineru-api.service](mineru-api.service) 中所有 `<用户名>` 替换为服务端用户名。

```bash
sudo cp mineru-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now mineru-api.service
```

管理命令按需单独执行。

```bash
sudo systemctl status mineru-api
sudo systemctl restart mineru-api
sudo systemctl stop mineru-api
sudo systemctl disable --now mineru-api   # 停止并取消自启
sudo journalctl -u mineru-api -f          # 实时日志
```

### 客户端（agent）

将 `<服务器IP>` 替换为实际地址。服务端启用鉴权时，取消密钥行的注释并填入密钥。

客户端 doclib 遥测默认上报至 `mineru.net`，可如下关闭。服务端组件暂无遥测。

```bash
uv tool install "mineru>=4.0,<5"

mineru config set parse_server.local.self_hosted_url "http://<服务器IP>:8000"
# mineru config set parse_server.local.self_hosted_api_key "<服务器密钥>"
mineru config set parse_server.local.mode self_hosted
mineru telemetry disable       # 可选，关闭客户端 doclib 遥测

mineru server status --json   # local parse_server 应为 healthy，tiers 含 advanced
```

## License

MIT
