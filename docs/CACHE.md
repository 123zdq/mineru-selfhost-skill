# MinerU CLI 解析结果缓存与持久化机制研究报告

> 研究对象：agent 侧 `mineru` CLI（doclib 模式）调用解析后，结果如何缓存、缓存多久、如何持久化。
> 依据：`mineru/doclib/`（`app.py`、`services/parse_svc.py`、`services/cleanup_svc.py`、`background/compaction.py`、`core/db.py`、`migrations/001_init.sql`）与 `mineru/config.py`，结论均从源码核实。

> [!NOTE]
> 本文档由 AI 生成，面向 Agent 阅读（亦可供维护者参考）。
> 基于 MinerU **4.0.11** 源码研究（opendatalab/MinerU，2026-10 快照）。

---

## 1. 两层存储架构

doclib 的缓存是「**SQLite 做索引 + 文件系统放内容**」的双层结构：

```
~/.mineru/                                ← MINERU_HOME，可整体搬迁/备份
├── doclib.db                             ← SQLite（WAL 模式）：元数据与任务索引
├── doclib.endpoint.json                  ← 后台服务端点发现文件（UDS/TCP）
└── doclib/                               ← data_dir（config: doclib.data_dir）
    ├── parsed/                           ← ★ 解析结果内容缓存（永久层）
    │   └── ab/                           ← sha256 前 2 位分桶
    │       └── ab12cd3.../               ← 文档内容 sha256（完整 64 位）
    │           ├── advanced/             ← 每个质量档位独立目录
    │           │   └── 1-10_1760000000000.json   ← <页范围>_<完成时刻毫秒>.json
    │           └── standard/
    │               └── 1-5_1759999000000.json
    └── temp/                             ← 临时层（可清理）
        └── read-assets/                  ← read --format image 按需生成的页/块图片
```

关键分工：

| 层 | 存什么 | 生命周期 |
|---|---|---|
| `doclib.db` 的 `parses` 表 | 每个解析批次的索引行：`(sha256, tier, page_range)` → 状态、`done_at`、文件名时间戳 | 跟随内容层，由 compaction/invalidate/cleanup 维护 |
| `parsed/.../*.json` | **完整 Middle JSON**（协议 v2.0：全部页面 blocks + 文本 + 几何 + 元数据信封） | 无 TTL，永久保留直至显式清理 |
| `fts_contents`（FTS5 虚表） | 全文检索索引（markdown 文本 + title/author） | tier 门控：只保留最高质量档的索引 |
| `temp/read-assets/` | read 时物化的页/块图片 PNG | 默认 7 天后可被 `cleanup temp` 删除 |

路径全部可配：`MINERU_HOME` 环境变量改根目录；`doclib.data_dir`、`doclib.sqlite.path` 可在 mineru 配置文件里单独指定。

---

## 2. 写入时机与内容格式

解析任务完成时（`parse_svc.py` `process_doc`）：

1. 把 `ParseResult.to_dict(skip_defaults=True)` 以 `indent=2`、`ensure_ascii=False` 写入 `parsed/<sha[:2]>/<sha>/<tier>/<page_range>_<done_at>.json`——**整批原子写入单个 JSON 文件**（不是数据库 blob）。
2. Markdown **不持久化**。源码注释明确："markdown is generated on read"——每次 `mineru read` 都从缓存的 Middle JSON 现场渲染。好处：渲染器升级后旧缓存也能用新格式输出；代价是渲染 CPU 开销（很小，纯文本变换）。
3. 全文索引：当场生成 markdown 文本（用完即弃）写入 FTS，且**只在更高档位时替换**（`TIER_ORDER` 门控：flash < basic < standard < advanced）。
4. 数据库把 `parses` 行标记 `done`，记录 `done_at`（即文件名里的毫秒时间戳，二者严格一致——这是之后重新定位缓存文件的寻址依据）。

### 图片的存储方式（重要）

Middle JSON 协议规定：**本地/文档内嵌图片以 `image_base64` 直接存进缓存 JSON**（远程 URL 图片只存受限 `image_url` 引用，解析时不下载）。

推论：

- 纯文本型 PDF 的缓存 JSON 体积 ≈ 文本量的数倍，通常远小于原文件；
- **扫描件/图片型 PDF 的缓存 JSON 内含全部页面图片的 base64（约 +33% 膨胀），单文档缓存可能接近甚至超过原文件大小**。容量规划必须按这个最坏情况估。

---

## 3. 缓存键与命中逻辑

### 内容寻址（sha256），不是路径寻址

- `files` 表：`path → sha256`（同一文件多处副本/改名自动共享同一份解析缓存）
- `docs` 表：`sha256` 主键 + `short_id`（locator 里的 `doc:ab12cd3` 就是它）
- `parses` 表：`(sha256, tier, page_range)` 唯一确定一个缓存批次

所以：文档移动、复制、重命名都不使缓存失效；**只有内容变化（sha256 变）才会**——此时生成新 doc，旧 doc 变成待清理的孤儿（见 §4）。

### 命中条件（`mineru read` / 续读时）

一个 `done` 行被视为有效缓存，必须同时满足（`_json_batch_is_readable`）：

1. JSON 文件存在于预期路径；
2. 内容可解析为当前 Middle JSON 协议（含 3.x 旧协议自动转换）；
3. 记录的页范围 ⊆ 文件里实际可用的页。

任一不满足 → 视为缓存损坏，触发重新解析（不会报错给用户）。

`mineru read doc:{short_id}` 不指定档位时取**最高质量档**的缓存（advanced > standard > basic > flash）。

---

## 4. 缓存到底保留多久？——完整答案

**done 缓存没有 TTL，默认永久保留。** 退出路径只有以下四条：

### 路径 1：用户 invalidate（标记废弃，不删文件）

```
mineru invalidate "report.pdf" --tier advanced
```

把 `done` 行改为 `superseded`，同时重建 FTS 索引。**JSON 文件原地保留**，但 read/parse 查询只认 `done` 状态，superseded 批次从此不可读——下次请求该内容会 `not_cached` 并触发重新解析。superseded 行及其文件的物理清除交给下一次 compaction（见路径 2）；若该文档该档位此后再无 ≥2 个 done 批次触发合并，文件会一直留在磁盘（无害但占空间），最终兜底是 orphan cleanup（路径 3）。

### 路径 2：Compaction 自动合并（唯一的自动删除）

后台任务（默认每小时跑一次，`doclib.compaction_interval_sec=3600`）：

- 找出同一 `(sha256, tier)` 有**多个 done 批次**的文档（比如你分页续读产生了 `1-10`、`11-20`、`21-30` 三批）；只有 1 个 done 批次的组合不动；
- 在内存合并全部页面 → 重新格式化为连续段（如 `1-30`）；**合并后行数不减少则放弃**（`len(merged_ranges) >= len(rows)` 直接返回）；
- **原子写新文件**（先 `.tmp-<ns>` 再 `os.replace`）→ 替换 DB 行（删该文档该档位全部 done + superseded 行，插入合并后的 done 行）→ **删除目录下所有不在保留集里的旧 JSON**；
- 任一源文件损坏则整轮放弃（fail-safe，不会丢数据）。

这是 parsed/ 目录下唯一会**自动**删除 JSON 的机制。注意它顺带承担了 superseded 行的物理清除：invalidate 留下的废弃行和文件，在下一次满足合并条件时一并删除。

### 路径 3：源文件消失 → orphan → 手动 cleanup

源文件被删除后的完整时间线：

1. **删除瞬间无任何反应**——doclib 不监听文件系统事件，靠 scan/watch 周期刷新（默认 300 秒）或手动 `mineru scan` 发现；
2. **发现后仅标记**：`files` 行标 `deleted` 并记录 `deleted_at`（进入 deleted-files 的 7 天清理倒计时）。例外：挂在 `--removable` watch 下的文件若 watch 根整体不可达，标 `unreachable` 且不计时——「移动硬盘拔了 ≠ 文件删了」；
3. **标记后能力矩阵**：文本 read / search / locator 续读全部照常（走缓存，不碰源文件）；文档内图片块照常（base64 内嵌缓存）；**页面截图和重新解析失败**（`no_accessible_file`，重解析有双重防护：SQL 只取 active 行 + 取到后实时 `os.path.exists` 验证）；
4. **orphan 判定不过滤 status**——`deleted` 行仍算对 doc 的引用。因此完整清理链是三步：`cleanup deleted-files --no-dry-run`（删行）→ `cleanup orphan-docs`（dry-run 确认）→ `cleanup orphan-docs --no-dry-run`（真正删 FTS + parses + docs 行 + rmtree 内容目录）。跳过第一步直接跑 orphan-docs 是「删不掉」的最常见原因；
5. 同内容多路径时，删一个路径无影响，其余 active 路径继续支撑一切功能；文件还原（同内容）后缓存全量复活。

### 路径 4：forget（主动遗忘）

`mineru forget <path>` 删除 files 行；若因此 doc 无引用，同样走 orphan 路径。

### 临时层的保留期

| 内容 | 默认保留 | 清理命令 |
|---|---|---|
| `temp/`（含 read-assets 图片、未完成解析的残留） | **7 天**（`cleanup_temp_files` 的 `older_than_days=7`） | `mineru cleanup temp [--older-than 14]` |
| `files` 表中 `deleted` 状态的行 | **7 天**（`cleanup_deleted_older_than`） | `mineru cleanup deleted-files` |

注意：`cleanup` 系列是**手动/agent 主动触发**的命令，不是后台定时任务。不跑 cleanup，temp 和 deleted 行也一直留着。

---

## 5. 持久性与崩溃恢复

| 问题 | 答案 | 源码依据 |
|---|---|---|
| 重启后缓存还在吗？ | **在。** SQLite + JSON 文件都在磁盘，doclib 后台服务只是无状态地打开它们 | `app.py` 启动只做 `initialize()`（幂等建表/迁移） |
| 崩溃安全吗？ | SQLite WAL + `synchronous=NORMAL`，JSON 写入先 tmp 后 `os.replace` 原子替换 | `db.py` PRAGMA、`compaction.py`、`process_doc` |
| 解析到一半崩溃？ | 下次启动把遗留 `parsing` 状态改判 `failed`、清空行锁，任务可重提；半成品 JSON 残留在 temp（7 天后清） | `app.py:106-110` 启动恢复逻辑 |
| 和 api-server 的任务存储什么关系？ | **完全不同层**：V1 API 的 upload/job 索引是进程内状态，重启即失效（`--upload-dir` 里的字节除外）；doclib 才是持久层。这正是 agent 用 CLI 而不是裸 HTTP 的核心理由之一 | `docs/zh/usage/http_api.md` 生产注意事项 |
| 多实例互斥？ | `instance_lock.py` + endpoint.json 保证单实例 | — |

**备份/迁移**：整个 `~/.mineru` 目录拷走即可（`doclib.db` + `doclib/parsed/` + 配置）。换机器时注意路径里的 sha256 寻址与绝对路径无关，直接可用；但 `files` 表里记录的是原机器的绝对路径，watch/scan 语义要重新对位（缓存命中不受影响，因为 parse/read 按 sha256 走）。

---

## 6. 对你的场景（远程服务器 + agent 轻薄本）的具体推论

1. **缓存全在轻薄本本地**：远程服务器只出算力，解析结果落回本机 `~/.mineru/doclib/parsed/`。断网后所有已解析文档照样可读、可搜、可引用——服务器只需在「新解析」时在线。
2. **advanced 档缓存的磁盘账**：你的 skill 默认 advanced。扫描件走 advanced 会把 VLM 版面结果 + base64 页图全量落到本地 SSD。粗略估算：100 个平均 5MB 的扫描 PDF ≈ 600MB~1GB 本地缓存。轻薄本若 SSD 紧张，可定期：
   ```bash
   mineru cleanup orphan-docs --no-dry-run   # 清掉已删文件的缓存
   mineru cleanup temp --older-than 7        # 清图片临时资产
   ```
3. **分页续读不会撑爆缓存**：compaction 每小时自动把 `1-10`、`11-20`… 合并成单个 `1-N` 批次并删旧文件，无需人工干预。
4. **重复解析零成本是设计保证**：内容寻址 + 页范围覆盖校验，同一文档（哪怕换了路径）的同档位解析绝不重跑——这正是把「有 locator 不 reparse」写进 skill 红线的底层依据。
5. **invalidate 不释放磁盘**：想让某文档缓存真正消失，`invalidate` 之后还需 orphan 化（删/移源文件）+ `cleanup orphan-docs --no-dry-run`，或直接删 `parsed/<xx>/<sha>/` 目录（下线操作，不推荐 agent 做）。
6. **FTS 只索引最高档**：搜索索引跟随你读过的最高质量档（advanced），flash watch 索引会被 advanced 解析自动顶掉——符合「搜索可用、精读再用 read」的 agent 习惯。

---

## 7. 一页速查

| 问题 | 答案 |
|---|---|
| 缓存存在哪 | `~/.mineru/doclib.db`（索引）+ `~/.mineru/doclib/parsed/<sha[:2]>/<sha>/<tier>/*.json`（内容） |
| 缓存键 | 内容 sha256 × tier × page_range；路径无关 |
| 缓存了什么 | 完整 Middle JSON（含 base64 页图）；markdown 不落盘、read 时渲染；FTS 文本进 SQLite |
| 默认保留多久 | **无 TTL，永久**；唯一自动删除是 compaction 合并去重 |
| 什么让它失效 | 内容变化（新 sha256 新 doc，旧的转 orphan）；用户 invalidate（仅标记 superseded） |
| 临时文件 | `doclib/temp/`，默认 7 天，`cleanup temp` 手动清 |
| 重启/崩溃 | 全持久（WAL + 原子写）；半途任务启动时改判 failed |
| 备份迁移 | 整个 `~/.mineru` 目录拷贝；`MINERU_HOME` 可重定位 |
