# MinerU `--concurrency` 与内部并发机制深度研究

> 全部结论基于源码核实：`mineru/parser/api_server.py`、`mineru/model/runtime/execution.py`、`mineru/backend/analysis/pdf/{pipeline,window,images,constants}.py`、`mineru/model/vlm/async_runtime.py`、`mineru/doclib/{services/parse_svc.py,background/parse_worker.py}`。
> 文末附：flash + advanced 双档融合（抗幻觉）方案评估。

> [!NOTE]
> 本文档由 AI 生成，面向 Agent 阅读（亦可供维护者参考）。
> 基于 MinerU **4.0.11** 源码研究（opendatalab/MinerU，2026-10 快照）。

---

## 1. `--concurrency` 如何生效

传递链：

```
mineru-kit api-server --concurrency N
  → parser/api_server.py: create_app(concurrency=N)
  → JobStore(concurrency=N)
  → self._semaphore = asyncio.Semaphore(N)     # 进程级、跨所有 tier 共享
```

任务生命周期（`JobStore.start_task`）：

1. `POST /v1/parse/jobs` 只创建任务记录（status=`queued`），立刻返回 `job_id`；
2. 每个任务是一个 `asyncio.Task`，任务体内 `async with self._semaphore:` ——**拿到槽位才开始真正解析**；
3. 槽位在任务清理完成后才释放（`finally` 保证异常也释放）。

要点：

- **N 是"同时解析的文档数"，对所有请求档位（flash/basic/standard/advanced）共享**，不按档位分池。
- 超出 N 的任务不会失败，只是 `queued` 变长（官方文档明确）。
- 它是单进程参数。多进程（router 多 worker）时每个 worker 各有一份。

## 2. 内部并发分层模型

单文档解析不是"一个线程跑到底"，而是分层并发。自外向内共六层：

```
层1  文档级     asyncio.Semaphore(--concurrency)        N 个文档任务在事件循环上交错
层2  阶段边界   run_sync / asyncio.to_thread            同步重活卸载到线程池，事件循环不被阻塞
层3  设备级     local_model_stage: per-device RLock     同一设备的小模型同步阶段互斥（不跨 VLM 等待持锁）
层4  渲染级     PDFium 多线程                           MINERU_PDF_RENDER_THREADS，默认 3 线程/文档
层5  VLM 页级   Semaphore(--vlm-max-concurrency=100)    页请求并发发出
层6  vLLM 内部  continuous batching                     真正的 GPU 吞吐引擎，自动组批
```

### 层3 是理解吞吐的关键（最容易被误解的一层）

`execution.py` 维护 per-device 的 `threading.RLock`。prepare 和 finish 两个同步阶段都持有它：

```python
# window.py 同步路径；异步路径等价（_prepare_locked_window / _finish_locked_window）
with local_model_stage(hybrid_model.device):      # ← 拿设备锁
    state = _prepare_pdf_window(...)               # render + layout + 表格朝向（小模型）
result = vlm_predictor.batch_extract_with_layout(...)  # ← VLM 等待期间【不持锁】
with local_model_stage(hybrid_model.device):      # ← 再拿设备锁
    return _finish_pdf_window(...)                 # OCR/公式/表格/回填/裁图（小模型）
```

锁的语义（注释原话："只保护同步本地模型阶段，不能跨越 VLM 等待持有该锁"）：

- **同设备的 layout/OCR/MFR/表格推理在单进程内全局串行**——无论小模型放 CPU(onnx) 还是 GPU(torch)，都是同一把 per-device 锁。`--concurrency` 调多大都改变不了这一段的吞吐。
- 但 VLM 推理（占 standard/advanced 总时长的大头，advanced 尤甚）**不受这把锁约束**：文档 A 拿锁跑 layout 时，文档 B/C 可以处于 VLM 等待，GPU 上的 vLLM 持续有活干。
- `MINERU_ENABLE_LOCAL_MODEL_INFERENCE_LOCKS`（默认 False）控制的是另一组更细的模型级锁，与这层设备锁无关；保持默认即可。

### 单文档内：窗口串行、页内并发

- 文档按 `MINERU_PROCESSING_WINDOW_SIZE`（默认 **64 页**）切窗口，**窗口间严格串行**（控制内存上界：任一时刻只有 64 页的 PIL 图 + numpy 数组在内存里）。
- 窗口内的阶段链与资源需求：

| 阶段 | stage_timer 标识 | 模型/引擎 | 算力位置 | 批大小 |
|---|---|---|---|---|
| 页图渲染 | `pdf.render` | PDFium | CPU（3 线程） | 按窗口 |
| 版面检测 | `pdf.layout` | 小模型 layout | GPU(torch) 或 CPU(onnx) | `min(8, 2×1)` = **≤8 页/批** |
| 表格朝向 | `pdf.table_orientation` | 小模型 | 同上 | —（flash/medium/high 有；**xhigh 跳过**，交给 VLM） |
| 原生表格 | `pdf.native_tables` | 无模型，PDF 向量几何 | CPU | 仅 txt+medium/high |
| **VLM 抽取** | （无锁窗口） | vLLM / llama.cpp | **GPU** | 页级并发 → vLLM 组批 |
| OCR 检测/识别 | `pdf.ocr_detection/recognition` | 小模型 | 同 layout | det 基准批 8 |
| 公式识别 | `pdf.formulas` | MFR 小模型 | 同 layout | 2×16 = **32** |
| 表格识别 | `pdf.table_recognition` | 表格小模型 | 同 layout | 仅 medium |
| 文本回填 | `pdf.text_fill` | PDF 文本层/OCR 结果 | CPU | — |
| 裁图资产 | `pdf.image_assets` | PIL | CPU | — |

### VLM 档位差异（standard vs advanced 在这一层分化）

| | standard (effort=high) | advanced (effort=xhigh) |
|---|---|---|
| VLM 调用 | `batch_extract_with_layout`：单步抽取，带小模型 layout 提示 | `batch_two_step_extract`：**两步抽取**，更贵 |
| 图片分析 | 关 | `image_analysis=True`（图表细读） |
| 表格朝向小模型 | 有 | 无（VLM 负责） |
| VLM 时间占比 | 高 | **更高**（两步 + 图片 token） |

## 3. 资源需求与瓶颈推论

单 api-server 进程的资源画像：

| 资源 | 消耗方 | 并发放大性 |
|---|---|---|
| GPU 显存 | vLLM 权重+KV cache（分离部署时不在本进程）；torch 小模型权重（~1-2GB） | 模型单例常驻（`HybridLocalModelContextSingleton`），**不随 --concurrency 增长** |
| GPU 算力 | vLLM continuous batching；torch 小模型 | vLLM 随页级并发放大；小模型受设备锁串行 |
| CPU | PDFium 渲染（3 线程/文档）、onnx 小模型、文本回填、裁图 | **随 --concurrency 近线性放大**（渲染不在设备锁内，onnx 在锁内） |
| 内存 | 每文档一个窗口（64 页图 + np 数组，峰值约数百 MB~GB 级） | 随 `--concurrency` 线性增长（每任务一个窗口） |
| 线程 | 事件循环×1 + 默认线程池 + 每文档渲染 3 线程 + VLM 专属线程×1 | 线程池与渲染线程随并发增长 |

调优结论：

1. **`--concurrency` 的收益主要来自"VLM 等待期间填满 GPU"**：文档 A 在 VLM 推理时，文档 B/C 的 CPU 阶段（渲染/回填/裁图）并行推进，GPU 不空转。advanced(xhigh) VLM 占比最大，收益最明显。
2. **小模型同步阶段是串行段**：当任务以 basic/txt-PDF 为主（小模型占比高）时，`--concurrency` 调大收益递减——瓶颈在设备锁。此时正确方向是提升单批效率（GPU torch 后端）或接受现状，而非堆并发。
3. **内存上限 ≈ 并发数 × 单窗口峰值**：`--concurrency 4` + 64 页窗口 ≈ 同时 4×64 页位图在内存。大文档场景可调小 `MINERU_PROCESSING_WINDOW_SIZE`（如 32）换取更平滑的内存曲线，代价是窗口切换开销略增。
4. CPU 算力充足时，"小模型 CPU(onnx) + VLM 独占 GPU"（EFFICIENCY.md §4）不仅省显存，还让 CPU 阶段与 GPU 阶段流水线错峰，与层3 的串行约束配合得最好。

## 4. doclib 客户端侧（agent 轻薄本）的并发

- 后台 `ParseWorkerPool` 默认 **2 个** asyncio worker（`doclib.parse_workers=2`，配置可调）。
- SQLite 队列 `parses` 表，优先级排序（主动请求 priority=1 > watch 发现 0），带锁超时回收。
- **缓存与去重按 `(sha256, tier, page_range)` 批次**：同一文件不同档位是完全独立的任务与缓存条目；去重（reused_parse_ids）只发生在**同档位**内。
- flash 档由 doclib **本地执行**（`local(flash)`），不进任何服务器；basic/standard/advanced 走 self_hosted 服务器。

---

## 5. flash + advanced 双档融合（抗幻觉）方案评估

### 机制可行性：成立，且条件意外地好

| 事实 | 依据 | 对融合的意义 |
|---|---|---|
| flash 在 agent 本地 CPU 执行，advanced 在远程 GPU 服务器执行 | `parse_svc.py`: `local(flash)` 分支；质量档走 `_resolve_api_target` | **双机天然并行，服务器零额外负担，flash 秒级先回** |
| 同文件两档是独立任务、独立缓存 | parses 表按 `(sha256, tier)` 键控；去重仅同档位 | 两条命令可同时提交，互不阻塞 |
| doclib 有 2 个 parse worker | `parse_workers=2` | 一个跑 flash(本地)、一个等 advanced(服务器) 恰好满载 |
| 两档共享同一 `doc:{short_id}` | short_id 由 sha256 决定 | **locator 跨档成立**：`tier:flash/page:N` 与 `tier:advanced/page:N` 同页码空间（1-based 原始页码），逐页对齐零成本 |
| 输出同构 | 都产出 MiddleJson → Markdown/Content List | 后处理可直接逐页 diff |

### 抗幻觉价值分析

- **flash 的本质是抽取而非生成**：txt 类 PDF 走 PDF 内嵌文本层（DocVortex 原生模型），零幻觉；扫描件退化为本地 Flash OCR（质量低于 VLM，但仍非生成式）。
- **advanced 是生成式 VLM**：幻觉风险真实存在（改写数字、编造表格单元格、漏低对比度文本）。
- 因此融合的正确姿势不是"两份结果投票"，而是**分工**：
  - **flash = 文本保真锚点**（尤其数字、编号、金额、日期、代码）
  - **advanced = 结构权威**（阅读顺序、表格结构、公式、标题层级、跨栏）
  - 后处理规则：逐页对齐后，advanced 文本与 flash 文本不一致的片段 → 以 flash 为准替换或标注"存疑"；表格/公式/图片块 → 采信 advanced；flash 没有的块（VLM 发现的）→ 采信 advanced 但可标注来源。
- 客观提醒：MinerU 的 hybrid 管线**本身已经在单档内做"抽取+生成"融合**（`_process_text_and_formulas` 会把原生文本层/OCR 结果回填进 VLM 的 block 结构）。双档融合是在其上再叠一层文档级 ensemble，对**高价值文档**（合同、财务、法律）收益明显；日常阅读可能过重。

### 推荐工作流（agent 侧）

```bash
# 同时提交（flash 本地立即跑，advanced 上服务器）
mineru parse "doc.pdf" --tier flash --no-wait --json
mineru parse "doc.pdf" --tier advanced --no-wait --json

# flash 通常几秒内完成，可先用它回答紧急问题
mineru read "doc:xxxx/tier:flash/page:1-10" --json

# advanced 完成后逐页对齐
mineru read "doc:xxxx/tier:advanced/page:5" --json
mineru read "doc:xxxx/tier:flash/page:5" --json
```

注意事项：

- Office/EPUB/HTML 等原生格式**没有质量档**（固定 flash 本地解析），双档融合仅适用于 **PDF 和图片**。
- 扫描件（无文本层）的 flash 走本地 Flash OCR，保真锚点价值下降，此时主要靠 advanced + 抽查页图（`--format image`）。
- 双档 = agent 上下文消耗约 ×2，建议融合阅读只用于关键页（表格/数字密集页），纯文字页读单档即可。
- `--force` 重解时注意两档缓存独立，需要分别刷新。
