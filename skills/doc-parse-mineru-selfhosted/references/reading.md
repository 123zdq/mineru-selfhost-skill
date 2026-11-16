# 读取细节

## 等待与批量

自建 GPU 的 quality 解析建议并发不超过 **4**，依据用户压测；服务器配置上限为 32。flash 本机执行，不占远程并发。

批量按未结束的远程任务控制提交，任务结束后补交。`--no-wait` 仅结束 CLI 等待，不代表解析结束，也不限制解析并发。

| 返回状态 | 后续操作 |
|---|---|
| `pending` / `parsing` | 保留 `wait_parse_ids`，继续查询任务 |
| `parse_wait_timeout` | CLI 等待到期，任务可能仍在继续；保留为待处理 |
| `done` | 读取或导出结果 |
| `failed` | 按 `error.code` 处理，见 [错误码](error-codes.md) |

`parse` 默认等待 60 秒，`--wait` 只改变 CLI 等待时长。查询用 `mineru show parse <id> --json`，也可重复原 `parse` 等待；`read` 只读已完成缓存。

按本批文件、档位、页范围记录任务 ID、状态和导出路径，完成一项收集一项。`list parses` 默认 50 条且含历史任务；使用任务 ID 定点查询，或通过 `--limit` / `--offset` 完整分页后筛选本批。解析完成、导出完成、内容核对分别记录。

## 导出与截断

| 操作或现象 | 处理 |
|---|---|
| `parse --output <文件> --json` | 导出请求页范围；检查 `output.status=written`，此时 `content=null` 正常 |
| 保存当前 Markdown 片段 | 读取 `mineru read ... --json` 的 `content` 字段，或重定向标准输出。`read --output <文件>` 仅用于 `--format image` |
| MinerU 返回 `next_request` | 按返回请求续读 |
| 外层工具截断输出或长行 | 读落盘正文或缩小读取范围，无需重解析 |
| 管道捕获后乱码 | 检查缓存导出；仅副本转码损坏时重新导出，无需 `--force` |

导出指定页范围的完整正文用 `mineru parse <PDF> --pages <范围> -o result.md --json`，由 MinerU 写 UTF-8 文件。PowerShell 5.1 的 `>` 通常写 UTF-16。

## 定位与续读

| 参数 | 规则 |
|---|---|
| 页、块、字符 | 页/块从 1 起；`char` 是块内从 0 起的偏移 |
| 未指定 tier | `read` 选已缓存的 advanced → standard → basic；只有 flash 时显式加 `/tier:flash` |
| `--context N` | 单页前后各 N 页，或块/char 前后各 N 块；仅文本单点定位支持 |
| parse 续读 | 保留原 path，传回 `next_request.page_range`（如有）和 `after`；非 PDF 不加 `--pages` |
| read 续读 | 读整篇、单页或页范围的 Markdown 时使用 `context=0`。按 `next_request.locator` 续读，有 `next_request.after` 时原样传给 `--after`，先读完被截断的当前页 |

按返回的 locator 和游标续读，例如

```bash
mineru read "doc:ab12cd3/tier:flash/page:1-2" --after "doc:ab12cd3/tier:flash/page:1/block:12/char:520" --limit 12000 --json
```

## 图片读取

一次读单页或单块，不支持非零 `--context`。PDF/图片可读整页或按 bbox 裁块；原生格式仅支持有可访问图片素材的 visual block。

不带 `--output` 时返回临时素材路径；保存后缀为 `.png/.jpg/.jpeg/.webp`。
