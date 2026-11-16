# mineru-selfhost-skill

[MinerU](https://github.com/opendatalab/MinerU) 自部署定制 skill 与 部署配方，基于 MinerU **4.2.2**。

> 相对官方 skill，除自部署适配外还有一处默认行为改变：flash+advanced 双档并发解析供 Agent 对照，以解决当前版本高档位的幻觉问题。



## 依赖

推荐使用 [uv](https://docs.astral.sh/uv/) 来管理 MinerU

## 快速开始

### 技能安装

标准 [Agent Skills](https://agentskills.io/) 风格，兼容任意常见 agent（[包括但不限于](https://agentskills.io/clients) OpenCode、Claude Code 等）

```bash
npx skills add 123zdq/mineru-selfhost-skill --skill doc-parse-mineru-selfhosted --global --yes
```

或以任意方式复制 `skills/doc-parse-mineru-selfhosted/` 目录到你用的 agent 的 skills 目录即可：

| 客户端 | 全局目录 | 项目级目录 |
|---|---|---|
| 通用 | `~/.agents/skills/` | `.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.opencode/skills/` |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |

### 服务器

#### 安装

```bash
uv tool install ninja
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

```bash
mineru-kit api-server \
  --host 0.0.0.0 --port 8000 \
  --tier  advanced \
  --concurrency 32 \
  --preload-models \
  # --api-key <你的密钥> \
```

验证，应正常返回4档
```bash
curl http://0.0.0.0:8000/v1/tiers
```

webui
```bash
mineru-kit webui --server-name 0.0.0.0 --server-port 7860 --api-url 8000
```

#### 设置开机自启

通过 `mineru-api.service`

> 安装前先把 unit 中的 `<用户名>`（`User`、`HOME`、`PATH`、`ExecStart`、`--upload-dir` 共 5 处）替换为服务器上的实际用户名；密钥等敏感信息不要写进仓库。

```bash
# 安装
sudo cp mineru-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now mineru-api.service

# 管理
sudo systemctl status|restart|stop mineru-api
sudo systemctl disable --now mineru-api   # 取消自启动
sudo journalctl -u mineru-api -f          # 实时日志
```


### 客户端（agent）

同依赖 [uv](https://docs.astral.sh/uv/)

客户端侧可关闭 doclib 遥测（默认开启并上报 mineru.net），服务器侧组件暂无遥测

```bash
uv tool install "mineru>=4.0,<5"

mineru config set parse_server.local.self_hosted_url http://<服务器IP>:8000
# mineru config set parse_server.local.self_hosted_api_key <服务器密钥>
mineru config set parse_server.local.mode self_hosted
mineru telemetry disable

mineru server status --json   # local parse_server 应为 healthy，tiers 含 advanced
```

## License

MIT
