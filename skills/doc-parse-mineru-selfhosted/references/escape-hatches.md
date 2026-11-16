# OCR、索引与维护

## 强制 OCR

扫描 PDF 带污染文本层、auto 解析后乱码时，可强制 OCR。`mineru parse` 固定 auto，需通过 kit 直连现有自建服务：

```bash
mineru-kit parse "bad.pdf" -o ./out --tier advanced --remote-url "http://<服务器>:8000" --api-key "<密钥>" --ocr-mode ocr
```

沿用现有地址和密钥。结果写入独立目录，不进入 doclib，无 doclib locator。

SDK 对应 `MinerUApiParser(..., ocr_mode="ocr")`，参数在构造器。

## 发现与索引

单文件 `parse` 已自动入库；一次性发现目录内文件用 `scan`，持续监控目录用 `watch`：

```bash
mineru scan "~/Documents/project" --no-wait --json
mineru watch add "~/Documents/project"
```

`scan` 不创建监控；`watch` 默认 flash。主动阅读仍用 `parse` / `read`。

## 库维护

| 命令 | 作用 |
|---|---|
| `mineru forget "<路径>"` | 移出库，不删源文件；仍在 watch 内的路径可能被重新发现 |
| `mineru cleanup deleted-files` | 清理已删源文件的记录与缓存 |
| `mineru cleanup orphan-docs` | 清理无路径关联的缓存 |

以上默认预览。用户要求实际维护时，在指定范围内加 `--no-dry-run` 执行；日常读取无需维护库。
