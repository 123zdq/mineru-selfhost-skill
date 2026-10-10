# 官方资源索引

> 基于 MinerU 4.0.11 校验 · AI 生成，供 Agent 使用

本 skill 未覆盖的细节，按场景查阅以下官方资源。文档站为 MinerU 4.0（本 skill 要求的版本）；链接失效时从文档站首页导航。

## 文档站与仓库

| 资源 | 链接 | 何时查 |
|---|---|---|
| 官方文档站（中文） | https://opendatalab.github.io/MinerU/zh/ | 总入口 |
| GitHub 仓库 | https://github.com/opendatalab/MinerU | 源码、Release |
| GitHub Issues | https://github.com/opendatalab/MinerU/issues | 疑似 bug、已知问题、报错关键字搜索 |
| DeepWiki 代码问答 | https://deepwiki.com/opendatalab/MinerU | 按代码行为提问 |
| 官方 skill 源文件 | https://github.com/opendatalab/MinerU/blob/master/skills/mineru/SKILL.md | 本地解析场景的官方规则（本 skill 的上游） |

## Agent 侧 / CLI 用法

| 页面 | 链接 | 何时查 |
|---|---|---|
| CLI 工具总览 | https://opendatalab.github.io/MinerU/zh/usage/cli_tools/ | 子命令、参数不确定时 |
| 基础使用 | https://opendatalab.github.io/MinerU/zh/usage/quick_usage/ | 转换输出的基本示例 |
| 档位与运行环境 | https://opendatalab.github.io/MinerU/zh/usage/tiers/ | 档位语义、引擎选择、格式与档位的关系 |
| 命令行参数进阶 | https://opendatalab.github.io/MinerU/zh/usage/advanced_cli_parameters/ | GPU 指定、引擎参数透传边界 |

## 协议与输出

| 页面 | 链接 | 何时查 |
|---|---|---|
| 输出文件格式 | https://opendatalab.github.io/MinerU/zh/reference/output_files/ | 各入口支持的导出格式、结果协议细节 |
| 迁移到 4.0 | https://opendatalab.github.io/MinerU/zh/reference/migration_4/ | 旧版行为对照（3.x 客户端、旧路由） |
| 更新日志 | https://opendatalab.github.io/MinerU/zh/reference/changelog/ | 行为变化、版本差异 |

## 服务端部署与运维（远程服务器侧）

用户自建服务器由用户运维；以下链接用于理解服务器行为或向用户转述排查建议时参考，不要代替用户操作服务器。

| 页面 | 链接 | 何时查 |
|---|---|---|
| V1 HTTP API | https://opendatalab.github.io/MinerU/zh/usage/http_api/ | 上传/任务/轮询/下载协议、鉴权与部署注意 |
| Python SDK | https://opendatalab.github.io/MinerU/zh/usage/sdk_api/ | `MinerUApiParser` 客户端行为、批量处理 |
| Docker 部署 | https://opendatalab.github.io/MinerU/zh/quick_start/docker_deployment/ | 服务形态（api/vlm/router/webui）、compose 配置 |
| 扩展模块 | https://opendatalab.github.io/MinerU/zh/quick_start/extension_modules/ | `[torch]`/`[full]` extras 与硬件要求 |
| 模型源配置 | https://opendatalab.github.io/MinerU/zh/usage/model_source/ | 模型下载源（HF/ModelScope）问题 |
