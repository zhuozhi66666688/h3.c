# AUDIT — integration 分支安全性审计（2026-10-21）

## 审计对象

`Bambushu/h3.c` 分支 `integration` @ `8e682e8`（2026-10-21）
相对上游 `antirez/h3.c` @ `8974cc0`（2026-08-11）

## 变更规模

```
47 files changed, 8699 insertions(+), 604 deletions(-)
```

按文件类型分布：实现代码（`h3_*.c/.m/.h`）、测试（`tests/`）、工具（`tools/fold_turbo_lora.py`、`tools/vae_bench.c`、`tools/vae_encoder_ab.c`）、文档（`docs/`、`README.md`）。

> 注：Opus 审核时预估"大概率几百行"，实测 8699 行插入，明显高于预估。主要增量来自新增工具与测试，核心引擎改动需另行确认。

## 恶意行为面审计结果

按 Opus 建议的四类风险逐项 grep：

| 检查项 | 模式 | 结果 |
|---|---|---|
| **网络出站** | `curl_easy` / `NSURLSession` / `socket(` / `connect(` / `getaddrinfo` / `CFStream` | **零命中**（排除 tests/） |
| **进程创建** | `system(` / `popen(` / `posix_spawn` / `fork(` / `exec*` | 仅 `h3_ffmpeg.c`：调用 `ffprobe` 和 `ffmpeg` 做视频编解码，属核心功能必需 |
| **敏感路径写入** | `/etc/` / `/usr/` / `~/Library/` / `.ssh` / `.aws` / `credentials` / `keychain` / `launchd` | **零命中**（排除 tests/） |
| **混淆/动态执行** | `eval(` / `exec(` / `XXTEA` / `rot13` | base64 编码器仅在 `h3_terminal.c:131,272`，用途是把像素/PNG 编码给终端图像协议（Kitty/Ghostty 的 `--show` 预览功能），非混淆 |

## 判定

**无网络出站、无敏感路径写入、无混淆代码、进程调用仅限 ffmpeg 编解码。** 在这个审计深度下未发现恶意行为。

## 未覆盖的残余风险

1. **未做逐行人工审查**。8699 行插入中，核心引擎（`h3_dit.c`、`h3_metal.m`、`h3_gpu.m`）的改动未经逐行核对。
2. **构建脚本未审计**。`Makefile` 的改动未逐行查看（是否引入额外编译单元或编译期执行）。
3. **测试代码未审计**（`tests/` 被排除在本次 grep 之外）。
4. **第三方 Python 依赖**。`tools/fold_turbo_lora.py` 声称只需 numpy，未验证是否 import 其他包。
5. **无签名/无 hash 锁定**。当前 clone 未做 sha256 记录，不能保证下次拉取得到同一份代码。

## 按 Opus 建议的缓解措施（待执行）

- [ ] ① pin `8e682e8` 并 vendor 进本地仓库，永不 `git pull`
- [ ] ② 用独立沙箱账号跑，主账号（带 SSH key、云凭据）不碰
- [ ] ③ `sandbox-exec` 挡网络出站 + 只允许写 output 目录
- [ ] ④ h3.c 二进制与 LoRA 的 sha256 写进构建脚本

## 审计命令（可复现）

```bash
cd ~/h3c-build/h3-integration

# 网络
grep -rnE "curl_easy|NSURLSession|socket\(|connect\(|getaddrinfo|CFStream" \
  --include="*.c" --include="*.m" --include="*.h" . | grep -v "^./tests/"

# 进程
grep -rnE "\bsystem\(|\bpopen\(|posix_spawn|\bfork\(|\bexec[lv]p?\(" \
  --include="*.c" --include="*.m" . | grep -v "^./tests/"

# 敏感路径
grep -rnE '"/etc/|"/usr/|"/Library/|\.ssh|\.aws|credentials|keychain|launchd|LaunchAgent' \
  --include="*.c" --include="*.m" --include="*.py" . | grep -v "^./tests/"

# 混淆
grep -rnE "base64|eval\(|exec\(|XXTEA|rot13" \
  --include="*.c" --include="*.m" --include="*.py" . | grep -v "^./tests/"
```
