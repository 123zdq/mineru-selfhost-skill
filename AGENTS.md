# AGENTS.md

本仓库面向 agent 的阅读与维护指引。

## 文档地图（docs/）

| 文档 | 内容 | 何时读 |
|---|---|---|
| [SETUP.md](docs/SETUP.md) | 安装与配置指南 | 部署、接入、验收时 |
| [EFFICIENCY.md](docs/EFFICIENCY.md) | 效率压榨研究：三层并发模型、部署拓扑、参数调优 | 调优服务器、规划容量时 |
| [CONCURRENCY.md](docs/CONCURRENCY.md) | 并发机制深度研究：`--concurrency` 与内部并发原语 | 调整并发、排查吞吐问题时 |
| [CACHE.md](docs/CACHE.md) | 缓存与持久化研究：doclib 双层存储、页级增量缓存 | 排查缓存、磁盘占用、备份迁移时 |

官方文档：https://opendatalab.github.io/MinerU/zh/

> docs/ 下研究文档由 AI 生成、面向 Agent 阅读（亦供维护者参考），基于 MinerU **4.0.11** 源码研究（官方仓库 @ `31e4e3d0`）。

## 维护约定

- 升级 MinerU 后：对照各文档头部的版本标注逐项复核结论，过时结论就地修订并更新版本号。
- 技能（`skills/mineru-selfhosted/`）基于[官方 skill](https://github.com/opendatalab/MinerU/blob/31e4e3d0/skills/mineru/SKILL.md)定制（v4.0.11 @ `31e4e3d0`）；升级时先 diff 官方变更，再合并定制点（self_hosted 模式、双档、禁 `--remote`、页级增量缓存等 Gotchas）。
