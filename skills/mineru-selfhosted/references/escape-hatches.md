# 低频逃生通道

> 基于 MinerU 4.0.11 源码核实 · AI 生成，供 Agent 使用

主路径之外的低频操作，按需加载。共同点：绕过 doclib（无缓存、无 locator）或改变库状态——执行前向用户确认。

## 强制 OCR（文本层污染的 PDF）

症状：`mineru parse` 结果文字错乱/乱码，但文档实际是扫描件（auto 分类漏检了假文本层）。

`mineru parse` 无 ocr_mode 参数（doclib 链路固定 auto），用 Python SDK 直连自建服务器强制重解析：

```python
from mineru.parser import MinerUApiParser

result = MinerUApiParser(
    api_url="http://<服务器>:8000", api_key="<密钥>", tier="advanced"
).parse("bad.pdf", ocr_mode="ocr")
print(result.markdown())
```

注意：`mineru-kit parse --remote-url ... --ocr-mode ocr` **不生效**——kit 的远程分支不透传 ocr_mode（`kit/commands/parse.py` 的 remote 路径仅在本地解析时传该参数），不要使用。

## 批量建库与持续监控

`mineru parse` 已自动 ingest 单文件并持久化（结果落 `~/.mineru/doclib`，无需建库）。大量文件的一次性/持续索引才用：

```bash
mineru scan "~/Documents/project" --no-wait --json   # 一次性扫描发现
mineru watch add "~/Documents"                       # 持续监控（flash 档索引）
```

watch/scan 结果是发现索引，不是高质量解析；主动阅读仍走 parse/read。

## 库维护

```bash
mineru forget "<路径>" [--no-dry-run]        # 移出库（不删源文件）
mineru cleanup deleted-files [--no-dry-run]  # 清理已删源文件的记录与缓存
mineru cleanup orphan-docs [--no-dry-run]    # 清理无路径关联的缓存
```

均默认 dry-run；`--no-dry-run` 属破坏性操作，必须用户明确同意后执行。
