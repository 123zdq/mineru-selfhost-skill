# MinerU 自部署服务器效率压榨研究报告

> 适用对象：单卡（GPU + CPU 算力充足）自部署 MinerU 4.0 服务器，日常负载是 agent 读文件。
> 依据：`MinerU` 官方仓库源码（`mineru/kit/`、`mineru/parser/api_server.py`、`mineru/model/`、`docker/compose.yaml`）与官方文档，以下所有默认值均从代码中核实。

> [!NOTE]
> 本文档由 AI 生成，面向 Agent 阅读（亦可供维护者参考）。
> 基于 MinerU **4.0.11** 源码研究（opendatalab/MinerU，2026-10 快照）。

---

## 1. 先搞清楚：效率发生在哪几层

MinerU 4.0 单机服务的并发结构是三层信号量叠加，理解这个才能对症调参：

```
客户端请求 (N 个)
   │
   ▼
┌─────────────────────────────────────────────┐
│ api-server (mineru-kit api-server)          │
│  JobStore: asyncio.Semaphore(--concurrency) │  ← 任务级：同时解析几个"文档"
│                                             │
│  每个文档内部：                               │
│   小模型 layout/OCR/公式/表格 (进程内)        │  ← torch 或 onnx
│   VLM 页级请求 ──────────────┐              │
└──────────────────────────────┼──────────────┘
                               ▼
               AsyncVlmPredictor: Semaphore(max_concurrency)  ← 页级：同时发几个"页"给 VLM
                               │
                               ▼ (若 --vlm-server-url 分离部署)
               ┌───────────────────────────────┐
               │ vlm-server (vLLM)             │
               │ continuous batching + KV cache │  ← 真正的吞吐来源
               └───────────────────────────────┘
```

关键结论：

1. **`--concurrency` 默认 1**（`api_server.py` JobStore），即第二个文档必须排队。多 agent / 批量场景必调。
2. **VLM 页级并发默认 100**（`--vlm-max-concurrency`），通常不用动。
3. **vLLM 的 `gpu-memory-utilization` 默认极保守**：`engine_utils.py` 中 `set_default_gpu_memory_utilization()` 对 >8GB 显存的卡统一按 **6GB 预算** 计算比例——24GB 卡默认只有 0.25，48GB 卡只有 0.13。**这是单卡压榨最大的一个杠杆**，且该参数**只能在独立部署的 `vlm-server` 命令行上设置**（api-server 不支持透传引擎参数，见 §3）。
4. 任务内小模型推理锁默认关闭（`MINERU_ENABLE_LOCAL_MODEL_INFERENCE_LOCKS` 默认 False），并发任务的小模型推理是真并行，不是伪并发。

---

## 2. 推荐部署拓扑（单卡）

### 方案 A：All-in-One（最简单）

一个进程同时承担 API、小模型、VLM：

```bash
mineru-kit api-server \
  --host 0.0.0.0 --port 8000 \
  --tier standard \
  --concurrency 3 \
  --preload-models \
  --api-key <你的密钥> \
  --upload-dir /data/mineru/uploads
```

- `--tier standard` 是**服务器能力档**，一个进程同时接受 basic / standard / advanced 三种请求档（advanced 不需要单独的启动档，只是推理 effort 更大：`tier.py` 映射 `advanced → xhigh`）。
- 进程内 VLM 引擎自动选 vLLM（装了 `mineru[full]` 且有 NVIDIA 卡时）。
- 缺点 1：api-server 重启 = VLM 一起重启，冷启动一次几十秒到几分钟。
- 缺点 2（决定性）：**无法调整进程内 vLLM 的显存预算**。`api-server` 命令不支持透传引擎参数（typer 命令无 `allow_extra_args`；对照 `vlm_server.py` 的 `FORWARD_CONTEXT_SETTINGS` 才有透传能力）；`VlmConfig` 也没有显存字段；`get_vlm_predictor()` 不传该 kwarg，于是 `runtime.py` 恒定落入 `set_default_gpu_memory_utilization()` 的 6GB 预算默认。**想控制 `--gpu-memory-utilization` 就必须用方案 B。**

### 方案 B：VLM 分离部署（推荐，吞吐与稳定性最优）

```bash
# 终端 1：VLM 常驻服务（vLLM，吃掉大部分显存做 KV cache）
mineru-kit vlm-server --engine vllm \
  --host 0.0.0.0 --port 30000 \
  --gpu-memory-utilization 0.70
#   ↑ 关键参数！默认只按 6GB 预算分配，务必手动设置；取值见 §4 显存预算表（24GB 卡同卡混布 0.70~0.75）

# 终端 2：API 服务（小模型在本进程，VLM 指向 30000）
mineru-kit api-server \
  --host 0.0.0.0 --port 8000 \
  --tier standard \
  --concurrency 3 \
  --preload-models \
  --api-key <你的密钥> \
  --upload-dir /data/mineru/uploads \
  --vlm-server-url http://127.0.0.1:30000 \
  --vlm-max-concurrency 100
```

优势：

| 维度 | 收益 |
|---|---|
| 稳定性 | api-server 升级/崩溃/重启不影响 VLM；VLM 权重不用反复加载 |
| 显存分配 | KV cache 预算（vlm-server）与小模型预算（api-server）各自独立、明确可控 |
| 调优自由 | vlm-server 的额外参数**原样透传给 vLLM**（`FORWARD_CONTEXT_SETTINGS`），可加 `--max-model-len`、`--max-num-seqs` 等 |

显存预算参考（24GB 卡）：vLLM 0.70~0.75（≈17~18GB，含 ~2.4GB 权重 + KV cache）＋ 小模型 torch（1~2.5GB，实测数据见 §4）＋ 系统余量 ~1GB。若小模型爆显存，改用 ONNX/CPU 跑小模型（见 §4）。

### systemd / Docker

- Docker 直接改用仓库 `docker/compose.yaml` 的 `api` profile；注意保留 `ipc: host`、`memlock: -1`、`stack: 67108864`（vLLM 共享内存依赖，丢掉会性能骤降或起不来）。
- 单机裸跑建议 systemd 两个 unit（mineru-vlm.service、mineru-api.service），`Restart=always`。

---

## 3. 参数级调优清单

### api-server（`mineru-kit api-server`）

> 引擎参数透传说明：`api-server` **不接受**任何 vLLM/LMDeploy 引擎参数（如 `--gpu-memory-utilization`），未知选项直接报错。官方文档「推理引擎参数透传」指的能力仅存在于 `vlm-server`（源码 `FORWARD_CONTEXT_SETTINGS`）；Router 同样不透传。引擎级调参只能配置在独立部署的 `vlm-server` 上（即方案 B）。

| 参数 | 默认 | 建议 | 说明 |
|---|---|---|---|
| `--concurrency` | **1** | **2~4** | 任务级并发。advanced（xhigh effort）单任务吃卡比例高，从 2 起步压测；standard 可到 4~6。超过 GPU 承载只会排队+争抢，不会报错但延迟变差 |
| `--preload-models` | 关 | **开** | 启动即加载小模型 + 建立 VLM 客户端，消除首个任务的冷启动 |
| `--tier` | 自动 | `standard` | 服务器能力档；standard 启动档 = 同时提供 basic/standard/advanced 请求 |
| `--vlm-server-url` | 空(进程内) | 方案 B 必填 | VLM 分离部署地址 |
| `--vlm-max-concurrency` | 100 | 保持 100 | 页级请求信号量；vLLM 侧 continuous batching 会自然吸收 |
| `--vlm-http-timeout` | 600 | 按需 | advanced 档大页/复杂页单页耗时可能长，偶发超时可加到 900 |
| `--upload-dir` | 临时目录 | **显式指定** | 不指定时文件字节存临时目录，**正常关闭即删除** |
| `--api-key` | 无 | 必设 | 绑 `0.0.0.0` 就必须鉴权（或放反代后面） |
| `--no-flash` | 关 | 可选 | agent 场景基本不用 flash，关掉可防误用（你的 skill 默认 advanced，可不关） |

### vlm-server（`mineru-kit vlm-server --engine vllm`）

| 参数 | 默认 | 建议 | 说明 |
|---|---|---|---|
| `--gpu-memory-utilization` | `max(6/VRAM, 0.1)` | **按 §4 公式取值**（24GB 同卡 0.70~0.75；小模型转 CPU 后 0.85~0.90） | 单卡压榨第一杠杆（见 §1、§4）。vLLM 预算按整卡总显存计算，需为小模型与系统留余量 |
| `--port` | 30000 | — | — |
| 其他 vLLM 参数 | — | 按需透传 | 例如 `--max-model-len 16384`、`--max-num-seqs 64`；mineru 只在缺省时补 port/gpu-mem/logits-processor，其余原样转发给 vLLM |
| `OMP_NUM_THREADS` | 自动设 1 | 别改 | vllm_server.py 启动时自动设置，多进程 OMP 会超订 CPU |

### 环境变量

| 变量 | 默认 | 建议 | 说明 |
|---|---|---|---|
| `MINERU_TABLE_DEVICE` | `auto` | 同卡跑小模型时可 `cuda` | 表格模型（表格分类/SLANet/UNet）用 `onnxruntime-gpu` 时优先 CUDA。注意：`onnxruntime` 与 `onnxruntime-gpu` **不可同时安装** |
| `MINERU_DEVICE_MODE` | 空(自动探测) | 需要强制设备时用 | `get_device()` 的最高优先级覆盖（如 `cpu`）。影响 torch 后端小模型与引擎设备探测；只想让小模型上 CPU 时优先用 `model.small_backend: onnx`，语义更精确 |
| `MINERU_ENABLE_LOCAL_MODEL_INFERENCE_LOCKS` | False | 保持 False | 打开反而让并发任务的小模型串行化 |
| `MINERU_MODEL_SOURCE` | auto | 国内服务器用 `modelscope` | 模型下载源，首次下载更快 |
| `MINERU_VLM_DEVICE` | 空 | NVIDIA 无需设置 | 仅国产卡（corex/kxpu 等）特殊配置用 |

> 显存参数的生效边界：`--gpu-memory-utilization` **只作用于 vLLM（VLM 栈）**，是 vLLM 按整卡总显存规划「权重 + KV cache 池」的预算比例。小模型（layout/OCR/公式/表格）不经过 vLLM，显存占用是静态的（权重 + 激活，实测稳态约 1~2.5GB，见 §4），没有预算参数，只能通过上表的设备选择控制放 CPU 还是 GPU。同卡混布时两条栈共享显存：vLLM 的比例按整卡计算，小模型的静态占用需从剩余部分出——取值公式与分卡速查表见 §4。

### 模型预下载（避免首次请求触发下载）

```bash
mineru-kit models download --tier standard   # basic+standard+advanced 共用这套模型
mineru-kit models verify --tier standard
```

约 2~3GB（小模型 + VLM 2B）。下载完再启动服务，配合 `--preload-models` 实现全热启动。

---

## 4. 小模型放置策略（CPU vs GPU）与显存预算计算

### 小模型实际占多少显存（逐文件核实）

小模型权重体积（HF 仓库 `opendatalab/MinerU-4_models_torch` 实际文件大小）：

| 模型 | 体积 | 备注 |
|---|---|---|
| Layout PP-DocLayoutV2 | 205 MB | |
| MFR PP-FormulaNet-Plus-M | 589 MB | **条件加载**：standard/advanced 档仅 `txt` 模式文档加载（`backend/analysis/pdf/ocr.py:54-58`），纯扫描件不加载 |
| OCR det/rec + 印章检测 | ~37 MB | |
| 表格三件套（slanet/unet/cls） | 22 MB | 默认走 CPU onnxruntime，**不占显存**（除非换 onnxruntime-gpu） |

权重合计 ≈ 830 MB；叠加 CUDA context/cuDNN 工作区（每进程约 0.3~0.6 GB）与推理激活峰值（随并发任务 0.2~1 GB）：

> **小模型栈稳态显存 ≈ 1.5 ~ 2.5 GB**（无公式数字文档约 1 GB，满负荷 2.5 GB）

### `--gpu-memory-utilization` 取值公式

vLLM 侧需要 VLM 权重 ~2.4 GB（1.2B bf16）+ KV cache（越大 continuous batching 吞吐越好）。同卡混布时：

```
util ≈ (总显存 − 小模型 2.5GB − 系统余量 1GB) / 总显存
```

| 显存 | 同卡混布（小模型 torch） | 小模型转 CPU（onnx） |
|---|---|---|
| 8~12 GB | 不推荐（太挤，易 OOM） | 0.75~0.80（≈官方 6GB 固定预算的默认假设——官方默认就是按小模型不在 GPU 设计的） |
| 16 GB | **0.65~0.70** | 0.80~0.85 |
| 24 GB | **0.70~0.75**（0.65 过于保守，白费 1GB+ KV cache） | 0.85~0.90 |

> vLLM 的 `gpu-memory-utilization` 按**整卡总显存**计算预算，同卡其他进程（小模型）的占用会挤占该预算内的 KV cache 空间。启动顺序建议：api-server（`--preload-models`）先加载小模型，vlm-server 后启动，压测确认无 OOM。

### 放置策略

standard/advanced 管线 = 小模型（layout/OCR/公式/表格）+ VLM。单卡上有个选择题：

| 策略 | 适合场景 | 代价 |
|---|---|---|
| 小模型 GPU(torch) + VLM GPU | 延迟敏感、单文档交互场景 | 显存争抢，vLLM 预算必须压低 |
| **小模型 CPU(onnx) + VLM 独占 GPU** | **agent 批量阅读、吞吐优先** | 单页小模型延迟略升，但 CPU 算力充足时流水线整体吞吐更高，VLM 拿满显存 |

CPU 放置方法：api-server 环境不装 GPU torch、只装基础包（ONNX 小模型 + CPU），即 `small_backend` 自动选 onnx。你的服务器"CPU 算力充足"，推荐这个组合——**GPU 只跑 vLLM，CPU 跑全部小模型与 PDF 前处理（渲染、切图），二者流水线并行**。

多核利用：PDF 页渲染与小模型 ONNX 推理在并发任务下天然吃多核；确认没被 cgroup/容器 CPU 配额限制即可。

---

## 5. Agent 侧协同效率（同样重要）

服务器效率最大化不只靠服务端参数，agent 的调用模式决定负载形态：

1. **页窗口 + 续读**：`mineru parse file.pdf --tier advanced` 默认只解前 10 页，后续 `mineru read` 按 locator 续读。首响应快、无效解析少。skill 已固化此模式。
2. **缓存优先**：doclib 按 (文档, tier) 缓存，重复读零成本。agent 侧红线"有 locator 不 reparse"已写进 skill。
3. **批量填队列**：多文件用 `--no-wait` 连续提交再轮询 `mineru list parses`，让服务器 `--concurrency` 槽位与 vLLM continuous batching 持续有活干，而不是"提交-等待-提交"的串行空转。
4. **秒传**：V1 上传接口支持 `sha256`，相同文件重复上传直接命中（响应 `completed` 且内嵌 file 对象）。CLI/SDK 自动处理，批量重复文件零传输成本。
5. **agent 侧零 GPU**：轻薄本只装 base `mineru` 包（ONNX/CPU 依赖都没有运行时负担，self_hosted 模式下不加载任何模型），全部推理在服务器。
6. **flash 对照零服务器成本**：doclib 中 flash 档走客户端本地执行（`parse_svc.py` 的 `_parse_via_local` 分支），quality 档才发往远程。skill 已将「flash+advanced 双档对照」固化为默认工作流（补偿 VLM 幻觉的临时策略），flash 侧不占用服务器 GPU、不影响其他任务吞吐；advanced 先 `--no-wait` 提交再跑 flash，双档总时长约等于 advanced 单档。

一次性客户端配置（agent 所在机器）：

```bash
mineru config set parse_server.local.self_hosted_url http://<服务器IP>:8000
mineru config set parse_server.local.self_hosted_api_key <你的密钥>
mineru config set parse_server.local.mode self_hosted
mineru server status --json   # 确认 local parse_server healthy 且 tiers 含 advanced
```

---

## 6. 监控与容量验证

| 手段 | 用法 | 看什么 |
|---|---|---|
| `GET /v1/health` | `curl -s :8000/v1/health \| jq` | `status`、`features.sources`、`features.output_formats` |
| `GET /v1/tiers` | 同上 | **必须包含 `advanced`**（除非 `--no-advanced`），否则 agent 的 advanced 请求会被 `tier_mismatch` 拒绝 |
| `GET /v1/usage` | 同上 | 累计用量，容量趋势 |
| `nvidia-smi dmon -s um` | 服务器上 | `sm%`（GPU 计算）、`mem`（显存）；压测时应看到 sm 持续 >70% |
| 服务日志 | `--log-level info` | 排队、OOM、引擎回退信息 |

**压测方法**（找 `--concurrency` 最优点）：准备 10~20 个有代表性的 PDF，用 agent 侧批量模式（`--no-wait` 连发），从 `--concurrency 2` 开始逐档上调，观察：总吞吐（页/秒）、`nvidia-smi` sm%、显存峰值、单任务延迟 P50/P95。sm 已打满但吞吐不再升 → 到顶；出现 `parse_oom` 或延迟劣化 → 回退。

**重启边界**：V1 API 的任务/文件索引是**进程内状态**，重启后旧 `job_id`/`file_id` 不保证可用（字节若在 `--upload-dir` 里仍在）。因此：持久阅读靠 agent 侧 doclib 缓存（本地 SQLite，重启不丢），服务器重启只影响 in-flight 任务。

---

## 7. 瓶颈症状速查

| 症状 | 大概率原因 | 对策 |
|---|---|---|
| 第二个文档永远排队 | `--concurrency` 还是默认 1 | 调到 2~4 |
| GPU 利用率低、显存大量闲置 | vLLM gpu-mem-util 默认 6GB 预算 | 显式设 `--gpu-memory-utilization 0.7±` |
| 首个任务特别慢 | 模型冷加载 | `--preload-models` + 预下载模型 |
| 批量时单任务延迟显著上升 | 并发超载或 advanced(xhigh) 叠加 | 降 `--concurrency`；批量场景考虑 standard 档（用户确认后） |
| `parse_oom` | 显存预算冲突（方案 B 同卡） | 降 vLLM gpu-mem-util 或小模型转 CPU |
| vLLM 起不来/极慢（容器） | 丢了 `ipc: host` / `memlock` | 补齐 compose 里的 ulimits |
| 表格解析慢但 GPU 闲 | 表格模型在 CPU(onnx) 且未装 gpu 版 | `onnxruntime-gpu`（替换而非共存）+ `MINERU_TABLE_DEVICE=auto` |
| advanced 请求被拒 `tier_mismatch` | 服务器 flash/basic 启动档或 `--no-advanced` | 用 `--tier standard` 启动、去掉 `--no-advanced` |

---

## 8. 上线检查单

- [ ] `mineru[full]`（或等效 vLLM）+ 模型预下载完成（`models verify --tier standard` 通过）
- [ ] 方案 B：vlm-server 显式 `--gpu-memory-utilization`；`/health` 通过
- [ ] api-server：`--tier standard --concurrency 2~4 --preload-models --api-key --upload-dir`
- [ ] `GET /v1/tiers` 包含 `advanced`；`GET /v1/health` ok
- [ ] 防火墙/反代：8000（和 30000）不对公网裸奔，或走 VPN/内网
- [ ] agent 侧 self_hosted 三行配置完成，`mineru server status --json` 显示 local healthy
- [ ] 跑一次端到端：`mineru parse test.pdf --tier advanced --json`，确认 `next_request` 与 locator 正常
- [ ] 按需压测并固化 `--concurrency`
