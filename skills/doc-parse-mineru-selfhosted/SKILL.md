---
name: doc-parse-mineru-selfhosted
description: 凡需理解文档内容，统一用本技能。使用用户自部署的 MinerU 提取文字、还原复杂版式，支持 PDF、扫描件、图片及 Office、HTML、EPUB、OFD、CSV、TSV。
metadata:
  version: "2.6"
  mineru-version: "4.2.2"
---

# MinerU 自托管文档阅读

## 环境约定（固化勿改）

- `parse_server.local.mode=self_hosted`：quality 档（basic/standard/advanced）自动发往用户**自有** GPU 服务器，无需上传许可询问；flash 本机执行，不占远程 GPU。
- PDF/图片默认 **flash + advanced**。advanced 必须显式加 `--tier advanced`；省略时 doclib 按 standard → basic 选可用档。
- **禁用 `--remote`**：它是官方云，非用户的服务器。

## 决策路径

1. 文件路径 → 纯文本（`.txt .md .markdown .rst .tex` 等）直接读；其余 `mineru parse`
2. `doc:...` locator → `mineru read`，不重新 parse
3. `<!-- Next: ... -->` → 原样执行；JSON 按 `next_request` 续读，保留 `after`，不要猜页码
4. 按文件名找 → `mineru find`；搜已解析内容 → `mineru search`

## 核心命令

```bash
# PDF/图片的双档调用示例
mineru parse "document.pdf" --tier advanced --no-wait --json
mineru parse "document.pdf" --tier flash --limit 12000 --json

# 定位读取与图片核对
mineru read "doc:ab12cd3/tier:advanced/page:4" --limit 12000 --json
mineru read "doc:ab12cd3/tier:advanced/page:4/block:5" --context 2 --json
mineru read "doc:ab12cd3/tier:advanced/page:4" --format image

# 查找、搜索与导出
mineru find "annual report" --ext pdf --json
mineru search "revenue recognition" --json
mineru parse "document.pdf" --tier advanced --output ./document.md --json
```

`--pages` 仅 PDF，默认前 10 页；如 `--pages "1-5,8"`、`--pages r3-r1`，1 起含端点，`r1` 为末页。双档范围保持一致；`all` 只在用户明确要求或文件已知很小时用。

**等待**：各档 `parse` 默认等 60 秒。`pending/parsing`、`parse_wait_timeout` 尚未完成；用 `wait_parse_ids` 查询 `mineru show parse <id> --json`，或重复原 `parse` 继续等待。完成后 `read` 取结果。

**批量**：自建 GPU 解析建议并发不超过 **4**（压测建议；服务上限 32）。双档均可 `--no-wait` 提交，按本批文档、档位、页范围收齐结果；详见 [读取细节](references/reading.md#等待与批量)。

## 注意事项

- **原生格式**（Office/HTML/MHTML/EPUB/OFD/CSV/TSV 等）：省略 `--tier` 自动走本地 flash，不支持 quality 档或 `--pages`。内嵌扫描图不 OCR，缺字向用户说明，不重试。
- **缓存**：doclib 按 (文档, 档位, 页批次) 持久化，跨会话/重启复用；扩大 PDF 页范围只补未覆盖页。仅当用户说结果过期/错误时，用 `--force` 本次重解，或 `mineru invalidate "<file>"` 作废该文件全部档位缓存。
- **双档对照**：advanced（VLM）保留版式，但可能改写或遗漏；flash 无生成环节，文字更忠实，但复杂版式还原较差。综合对照两份结果还原原文。
- **导出**：大段正文可用 `parse --output <临时文件> --json`；检查 `output.status=written`。导出不扩大页范围。
- `search` 包含历史会话的已解析内容；若 PDF/图片只有 flash 命中，先补 advanced 再依赖其内容。
- 禁止打印 config 中的密钥。

## 错误处理

按 JSON `error.code` 分支。`server_not_running` → `mineru server start` 后重试一次；其余错误查 [错误码处理表](references/error-codes.md)。

## 按需参考

- [读取细节](references/reading.md)：批量、等待、导出、截断、locator 参数
- [字段核验](references/field-verification.md)：字段歧义、双档分歧、跨文档比对
- [低频操作](references/escape-hatches.md)：强制 OCR、建库/监控、库维护
- [官方参考](references/official-docs.md)：本文未覆盖的参数、协议、部署与版本问题
