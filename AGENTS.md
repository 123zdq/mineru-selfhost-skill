# AGENTS.md

本仓库面向 agent 的阅读与维护指引。

## 文档地图（docs/）

| 文档 | 内容 | 何时读 |
|---|---|---|
| [benchmark.md](docs/benchmark.md) | 自建解析服务压测报告：并发-吞吐-延迟、推荐工作并发、容量估算 | 调整服务端并发、规划容量/扩容时 |
| [troubleshooting.md](docs/troubleshooting.md) | systemd 部署踩坑：`--upload-dir` 崩溃循环、PATH/HOME 环境、`.xlsm` 兼容 | 部署 `mineru-api.service`、排查启动/环境问题时 |

官方文档：https://opendatalab.github.io/MinerU/zh/

> docs/ 为实测文档（压测、踩坑），面向 Agent 阅读（亦供维护者参考）；结论与 MinerU 版本和硬件强相关，环境见各文档内标注（压测基于 MinerU 4.2.0、RTX 2080 Ti）。

## 维护约定

- 升级 MinerU 后：逐项复核 docs/ 结论是否仍成立（版本、硬件强相关），过时结论就地修订并更新版本标注。
- 技能（`skills/doc-parse-mineru-selfhosted/`，frontmatter 标注 `version` 与 `mineru-version`）基于[官方 skill](https://github.com/opendatalab/MinerU/blob/master/skills/mineru/SKILL.md) 定制；升级时先 diff 官方变更，再合并定制点（self_hosted 模式、双档、禁 `--remote`、页级增量缓存等 Gotchas），并同步更新 `mineru-version`。
