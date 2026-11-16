# 错误码

按 `error.code` 处理，`message` / `param` 辅助定位。参数错误修正后再试；瞬时故障重试一次，持续失败则保留错误及未完成范围。

## 任务与资源

| code | 处置 |
|---|---|
| `parse_wait_timeout` | 继续查询任务或重复原 `parse` 等待，见 [等待与批量](reading.md#等待与批量) |
| `server_busy` | 稍后重试一次 |
| `parse_timeout` | 查任务状态；延长 CLI 的 `--wait` 不会延长引擎超时 |
| `parse_oom` | 减少并发；PDF 可拆小页批次，仍覆盖原目标范围 |
| `resource_limit` | 按错误所指缩小单次输入，分批覆盖原任务范围 |
| `parse_failed` / `parse_empty` | 查任务错误、文件状态及输入有效性 |
| `http_error` / `parse_page_remap_failed` | 核对客户端/服务器版本、失败页范围与服务端日志 |

## 服务与鉴权

| code | 处置 |
|---|---|
| `server_not_running` | `mineru server start` 后重试一次 |
| `parse_server_unavailable` | 查 `mineru server status --json` 及自建服务器连通性 |
| `engine_unavailable` | 查服务状态；瞬时启动失败可重试一次 |
| `quality_tier_unavailable` / `no_engine` / `tier_mismatch` | 核对状态中的 `parse_server.local.supported_tiers`；advanced 需服务端开放该档位 |
| `invalid_api_key` | 核对 `parse_server.local.self_hosted_api_key` 与服务端鉴权配置；缺少有效密钥时向用户获取 |
| `invalid_doclib_token` | 核对 CLI 与服务是否使用同一 `MINERU_HOME`；token 由 CLI/SDK 自动读取 |
| `doclib_upgrade_required` | 核对 CLI/doclib 版本；迁移时用旧版本停止旧 doclib，再用当前版本启动，不手删 endpoint/lock |
| `service_unavailable` / `server_instance_mismatch` | 核对服务状态、`MINERU_HOME` 与实例，不手动删除锁文件 |

`api-server --tier` 指开放档位的上限；以 standard 启动不包含 advanced。

## 输入与参数

| code | 处置 |
|---|---|
| `tier_unsupported_for_file_type` | 去掉 `--tier`，原生格式自动走 flash |
| `remote_unsupported_for_file_type` / `tier_unsupported_for_remote` | 去掉 `--remote`；本环境使用自建服务 |
| `page_range_invalid` | 修正 PDF 页范围；非 PDF 去掉 `--pages` |
| `parse_not_required` | 直接读原文件 |
| `file_not_found` | `mineru find` 查找，仍未找到时确认路径 |
| `file_permission_denied` | 检查源文件读取权限 |
| `file_type_unsupported` | 当前入口不支持该格式 |
| `file_encrypted` | 使用解锁副本 |
| `file_corrupted` | 使用有效副本 |
| `file_too_large` | 按输入限制拆分或使用较小副本 |
| `invalid_locator` / `invalid_request` | 按 `param` 修正；游标约束见 [定位与续读](reading.md#定位与续读) |
| `context_not_applicable` | 去掉非零 context，或改为单页/块/char 的文本读取 |
| `multi_page_image_not_supported` | 改为单页 locator，逐页读取 |
| `image_output_extension_unsupported` | 改用 `.png/.jpg/.jpeg/.webp` 或省略 `--output` |
| `format_not_supported` | 使用支持图片的页/块定位，见 [图片读取](reading.md#图片读取) |

## 缓存与素材

| code | 处置 |
|---|---|
| `not_cached` | 先查对应任务；待完成则继续等，确实未解析才 `parse` |
| `tier_not_cached` | 只有 flash 时显式加 `/tier:flash`；PDF/图片按双档策略补齐 |
| `page_not_cached` | PDF 用 `parse --pages` 补所需页，保留原档位 |
| `job_not_found` | 用原文件、档位和页范围确认缓存；必要时重复原 `parse` 获取状态，无需 `--force` |
| `block_not_found` / `doc_not_found` | 用 `find` / `show file` 核对文档与 locator |
| `no_accessible_file` / `asset_not_available` | 恢复源文件或图片素材后再读 |
