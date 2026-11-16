# 踩坑记录（systemd 部署）

按踩到的顺序记录，均已实测验证。

## 1. `--upload-dir` 默认值导致启动即崩（无限重启循环）

**症状**：服务装成系统服务后无法启动，`journalctl -u mineru-api` 每 10 秒重复：

```
错误: Invalid value for '--upload-dir': Directory '' is not writable.
```

**原因**：`api-server` 的 `--upload-dir` 选项默认值为 `""`，且类型为
`click.Path(file_okay=False, writable=True, resolve_path=True)`——空字符串被 realpath
解析为**进程当前工作目录**。systemd 系统服务默认 `WorkingDirectory=/`，普通用户不可写，
click 参数校验失败，进程立即退出。配合 `Restart=on-failure` 表现为无限崩溃重启循环。

同样的 unit 以 systemd **用户**服务运行时不触发（工作目录继承自家目录，可写），
所以手动测试正常、装成系统服务才炸。

**规避**：显式传可写目录（本仓库的 unit 已带）：

```ini
--upload-dir /home/<用户名>/.mineru/uploads
```

或给 unit 设可写的 `WorkingDirectory=`。

**根因修复**：上游把 `default=""` 改为 `default=None` 即可（click 对 None 跳过校验，
下游代码已兼容空值）。4.0.10、4.1.7 均存在此问题。

**上游 issue**：<待提交后补链接>

**最小复现**（任何 Linux，非 root 用户）：

```bash
cd / && mineru-kit api-server
```

## 2. `StartLimitIntervalSec` 写在 `[Service]` 段会被静默忽略

journal 里持续告警 `Unknown key name 'StartLimitIntervalSec' in section 'Service', ignoring`。
它属于 **`[Unit]`** 段。写错位置时不报错、不生效，重启次数上限仍是默认值——
在坑 1 的崩溃循环里可能导致重试彻底停止，让人误以为是不重试的 bug。

## 3. systemd 的 PATH 不含 `~/.local/bin`

uv tool 安装的 mineru 及 ninja 都在 `~/.local/bin`。`ExecStart` 用绝对路径能启动，
但运行期 FlashInfer JIT 编译子进程找不到 `ninja` 会失败。unit 里显式补：

```ini
Environment=PATH=/home/<用户名>/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

## 4. 模型缓存路径依赖 `HOME`

模型缓存在 `~/.mineru` 与 `~/.cache`。系统服务若不显式指定 `HOME`，
按 root 的环境解析，找不到已下载的模型。unit 需要：

```ini
User=<用户名>
Environment=HOME=/home/<用户名>
```

## 5. `.xlsm` 不支持

服务端报 `Unsupported file type`。接入层需拦截或转存为 `.xlsx`
（压测中唯一的解析失败即来源于此，非负载问题）。
