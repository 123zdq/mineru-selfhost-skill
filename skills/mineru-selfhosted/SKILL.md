---
name: mineru-selfhosted
description: 用 mineru 管线阅读 PDF、扫描件、图片、Word/PPT/Excel、EPUB、OFD、HTML、CSV 等文档，可靠还原 Read 类工具与多模态 API 处理不了的复杂版式。用户给出文件想了解、提取、OCR、总结、引用或续读其内容时使用——即使没明说"解析"，也不要用 Read 类工具直接读这些格式。
metadata:
  version: "1.5"
  mineru-version: "4.0.11"
  dual-tier-note: 无条件双档是补偿 VLM 幻觉的临时策略；官方流水线与 VLM 能力足够强后可撤销，恢复单 advanced 档
---

# MinerU 自托管文档阅读

## 环境约定（已固化，勿改动）

- 本机 doclib 已配置 `parse_server.local.mode=self_hosted`：一切 quality 档（basic/standard/advanced）解析自动发往用户**自有**远程 GPU 服务器，无需上传许可询问。
- **默认双档：flash + advanced**（PDF/图片）。advanced 必须显式加 `--tier advanced`——省略时 doclib 会默认选 standard；flash 本机执行、零服务器成本。
- **禁止使用 `--remote`**：那会把文档上传到官方云服务，而非用户的服务器。

## 决策路径

1. 文件路径要内容 → 纯文本（`.txt .md .markdown .rst .tex` 等）直接读原文件；否则 `mineru parse`（PDF/图片双档）
2. `doc:...` locator → `mineru read`，不要重新 parse
3. 输出含 `<!-- Next: ... -->` → 原样执行，不要自己猜页码
4. 按文件名找文档 → `mineru find`；在已解析内容里搜 → `mineru search`

## 核心命令

```bash
# 首次读（双档）：advanced 先提交远程（慢，轮询见下）；flash 本机立即返回
mineru parse "document.pdf" --tier advanced --no-wait --json
mineru parse "document.pdf" --tier flash --limit 12000 --json

# 指定页范围（仅 PDF；1 起始含端点，r1=最后一页，all=全本），两档页范围应一致
mineru parse "book.pdf" --pages 1-20 --tier advanced --no-wait --json
mineru parse "book.pdf" --pages 1-20 --tier flash --json

# 按 locator 续读（页码/块号从 1 起；--context N 取相邻页/块；char:{offset} 为块内 0 起字符定位）
mineru read "doc:ab12cd3/tier:advanced/page:11" --limit 12000 --json
mineru read "doc:ab12cd3/tier:advanced/page:12/block:5" --context 2 --json
mineru read "doc:ab12cd3" --json   # 只给 doc id 时读最高档缓存

# 页/块图片（视觉检查、表格公式核对）
mineru read "doc:ab12cd3/tier:advanced/page:4" --format image

# 查找与搜索；--output 把转换结果存文件（用户要求保存时）
mineru find "annual report" --ext pdf --json
mineru search "revenue recognition" --json
mineru parse "document.pdf" --tier advanced --output ./document.md
```

advanced 档大文件解析慢属正常：单文件可用 `--wait 180` 阻塞等待；`--no-wait` 提交后用 `mineru show file "<path>" --json` / `mineru list parses --json` 轮询，完成前可先用已返回的 flash 结果工作，完成后用 `mineru read` 取内容。

**批量文件**：先进远程队列，再补本机 flash，统一轮询，不要串行等待：

```bash
mineru parse "a.pdf" --tier advanced --no-wait --json
mineru parse "b.pdf" --tier advanced --no-wait --json
mineru parse "a.pdf" --tier flash --no-wait --json
mineru parse "b.pdf" --tier flash --no-wait --json
mineru list parses --json
```

## Gotchas

- **Office/HTML/EPUB/OFD/CSV 等原生格式只能走本地 flash 档**（结构化格式原生解析，无模型推理）：省略 `--tier` 自动路由 flash，显式传 quality 档报 `tier_unsupported_for_file_type`；`--pages` 仅 PDF 支持（图片也不支持）。质量上限由格式决定——内嵌扫描图片中的文字不在结果中（无档位可解），发现缺失向用户说明而非重试。
- **解析结果自动持久化在本地 doclib，跨会话、跨重启有效**：parse 一次后 locator 长期可用，新会话中直接 `read`/`search`/`find` 复用，不要重新 parse。缓存按 (文档, 档位, **页批次**) 增量——扩大页范围（如已读 1-10 再要 1-20）只解析未覆盖页，历史页零成本。仅当用户说结果过期/错误时用 `--force`（本次重解）或 `mineru invalidate "<file>"`（作废该文件全部档位缓存）。
- `--pages all` 只在用户明确要求或文件已知很小时使用。
- **双档互补**：advanced（VLM）版式还原完整，但属生成式模型，可能改写甚至丢失信息；flash 无生成环节，文字忠实但丢失复杂版式。同时阅读两份输出，由你对照还原原文。flash 在本机执行、不占远程 GPU，可与 advanced 并发。
- `search` 搜全库已解析内容（含历史会话）；若命中只有 flash 结果，先以 advanced 解析再依赖其内容。
- 不打印 config 中的密钥。

## 错误处理

按 JSON `error.code` 分支（不按 message 措辞）。唯一自愈动作：`server_not_running` → `mineru server start` 后重试一次。

红线：不因故障自动换其他解析器、不加 `--remote`、不降档、不改配置/重启服务——除非用户要求或同意。

其余错误码（服务器不可达、鉴权失败、超时、OOM、文件问题等）的处理见 [references/error-codes.md](references/error-codes.md)，遇到错误或处理方式不明确时读取。

## 深入参考

- [references/escape-hatches.md](references/escape-hatches.md)：低频逃生通道——乱码文本强制 OCR 重解析、批量建库/监控、库维护；遇到对应场景时读取
- [references/official-docs.md](references/official-docs.md)：CLI 参数细节、输出格式协议、服务器侧行为与官方文档/仓库链接；本文档未给出答案时读取

## 引用

回答文档问题时附 locator 供溯源：

```text
质保期为 24 个月（doc:ab12cd3/tier:advanced/page:7/block:3）
```

表格、公式、版面敏感的问题：读相关页/块，必要时 `--format image`。基于截断内容回答时注明已读范围。
