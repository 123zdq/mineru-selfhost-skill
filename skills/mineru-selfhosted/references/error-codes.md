# 错误码处理表

> 基于 MinerU 4.0.11 源码核实 · AI 生成，供 Agent 使用

错误 JSON 的 `error` 对象含 `code`（分支依据）、`retryable`、`user_action`（官方建议动作）。按 `code` 分支，尊重 `retryable`；SKILL.md 的红线（不自动换解析器、不加 `--remote`、不降档、不改配置）始终适用。

## 本环境特有

| code | 含义 | 动作 |
|---|---|---|
| `parse_server_unavailable` | 远程自建服务器不可达 | 报告用户（服务器侧问题，本机无法修复），不要降级 flash、不要加 `--remote` |
| `invalid_api_key` | 服务器鉴权失败 | 询问用户是否更新 `parse_server.local.self_hosted_api_key`，经同意后再改 |
| `quality_tier_unavailable` / `no_engine` | 服务器未提供所请求档位 | 运行 `mineru server status --json` 查看 `parse_server` 状态并报告用户；通常是服务器未以 standard 能力档启动，等用户处理 |
| `engine_unavailable` | 引擎进程不可用 | `retryable` 为真时重试一次；否则 `mineru server status --json` 检查 |

## 通用

| code | 含义 | 动作 |
|---|---|---|
| `server_not_running` | 本机 doclib 后台服务未运行 | `mineru server start`，重试一次 |
| `parse_timeout` | 解析超时 | `--wait 180` 重试或 `show file` 查进度；advanced 档大文件耗时更长属正常 |
| `parse_oom` | 服务器显存/内存耗尽 | 报告用户；可建议缩小 `--pages`，经同意后执行 |
| `parse_failed` | 解析失败 | 报告失败；仅在用户同意后尝试其他档位 |
| `tier_mismatch` | 请求档位服务器不支持 | 报告用户可用的档位（来自 `server status --json`） |
| `tier_unsupported_for_file_type` | 对 Office/HTML/EPUB/CSV 等原生格式显式请求了 basic/standard/advanced | 去掉 `--tier`（自动走 flash）；原生格式只能 flash |
| `remote_unsupported_for_file_type` | 对原生格式使用了 `--remote` | 去掉 `--remote`（本环境本就禁用）；原生格式只能本地 flash |
| `tier_unsupported_for_remote` | `--remote` 与 flash 组合 | `--remote` 不支持 flash；本环境直接去掉 `--remote` |
| `page_range_invalid` | 页范围非法，或对非 PDF（含图片）使用了 `--pages` | 修正范围或去掉 `--pages` |
| `parse_not_required` | 输入是可直接读取的纯文本 | 直接读原文件，不要重试 parse |
| `file_not_found` | 路径或文件 id 不存在 | 向用户要正确路径，或 `mineru find` 查找 |
| `file_permission_denied` | 本地文件不可读 | 请用户修复权限 |
| `file_type_unsupported` | 格式不支持 | 报告不支持的类型 |
| `file_encrypted` | 文件有密码 | 请用户提供解锁副本 |
| `file_corrupted` | 文件无法读取 | 请用户提供有效副本 |
| `not_cached` / `cache_miss` | 请求的缓存内容不存在 | 先 `mineru parse` |
| `job_already_terminal` | 任务已处于终态 | 用 `mineru show parse <id>` 查看结果 |

## 重试规则

- 仅对瞬时启动错误或 `retryable: true` 的错误重试一次；不无限重试解析失败。
