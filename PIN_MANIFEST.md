# H3C 加速分支 — 构建锁定清单（PIN MANIFEST）

> 本文件锁定 `integration` 分支的可复现状态。任何"这次跑得快/慢"的结论必须能追溯到这份清单。
> 维护纪律：**永不 `git pull`**。要升级就改这里的所有 pin 值，作为一次显式决策。

## 一、源码

| 项 | 值 |
|---|---|
| 仓库 | `https://github.com/Bambushu/h3.c` |
| 分支 | `integration` |
| **锁定 commit** | `8e682e8bd44e04a1e2006695dce305c2bc915ea6` |
| 提交时间 | 2026-10-21 08:35:46 +0700 |
| 提交说明 | `Default the video VAE to bf16 and prefetch its decoder` |
| 相对上游基线 | `8974cc0`（antirez/h3.c，2026-08-11，之后零提交） |
| 变更规模 | 47 files / 8,699 insertions / 604 deletions |

## 二、二进制

| 项 | 值 |
|---|---|
| 构建命令 | `make -j8` |
| 构建时间 | 2026-10-21 |
| **h3 二进制 SHA-256** | `42a280abe8b3dc26a1000518fcd39ca3e8f0c87076f0b852871d718995b8ed6c` |
| 大小 | 546 KB |
| 编译警告 | 1 条，`h3_gpu.m:273` GNU `?:` 扩展，无害 |

### 校验方法

```bash
cd <repo> && make -j8
shasum -a 256 h3
# 应输出 42a280ab...
```

## 三、模型权重（不复制，共用原目录）

| 项 | 值 |
|---|---|
| 路径 | `/Users/zhuozhi/Documents/自媒体/AI 漫剧/h3.c/MiniMax-H3` |
| Qwen3-VL encoder | 14 files / 1058 tensors / 62.133 GiB |
| FL2VA DiT | 13 files / 535 tensors / 61.728 GiB |
| Ref2VA DiT | 13 files / 535 tensors / 61.728 GiB |
| video VAE | 1 file / 560 tensors / 9.700 GiB |
| audio VAE | 1 file / 1087 tensors / 0.564 GiB |

## 四、运行时开关（本机验证过的组合）

| 场景 | 开关 |
|---|---|
| 生产片（≥49 帧 或 ≥768×1344） | `H3_VAE_PREFETCH` 默认开（勿设 0） |
| 预览片（≤32 帧 且 ≤640²） | `H3_VAE_PREFETCH=0` |
| 要求与原始 F32 解码完全一致 | `H3_VAE_PRECISION=f32` |
| Sol-Attn | **默认关闭**，未通过三镜头阶梯前不要开 |

## 五、安全审计基线

四类恶意行为检查结果（详见任务目录 `AUDIT.md`）：

| 检查项 | 结果 |
|---|---|
| 网络出站（curl_easy / NSURLSession / socket / connect / getaddrinfo / CFStream） | 零命中 |
| 敏感路径写入（/etc /usr /Library /.ssh /.aws / credentials / keychain / launchd） | 零命中 |
| 混淆或动态执行（eval / exec / XXTEA / rot13） | 仅 `h3_terminal.c` 的 base64，用于终端图像协议 |
| 进程创建 | 仅 `h3_ffmpeg.c` 调用 ffprobe/ffmpeg |

**未覆盖**：未做逐行人工审查；`Makefile` 改动未逐行核对；`tests/` 未纳入审计范围。

## 六、本机实测基线（用于回归对比）

硬件：MacBook Pro M5 Pro，48 GB 统一内存，macOS 26.5.1，Metal 4

几何：768×1344 / 请求 49 帧（实出 56 帧 / 2.333 s）/ 8 步 / 50 层 / seed 42

| 阶段 | 时间 |
|---|---|
| Qwen 文本编码 | ≈77 s |
| DiT 加载 | ≈128 s |
| GPU Euler 去噪 | 465–480 s |
| video VAE 解码 | 62–77 s |
| DiT 峰值显存 | 20.25 GiB |
| 端到端 | 约 11 分钟/条 |

**回归判定**：同参数下若去噪偏离 465–480 s 区间超过 15%，先查 pin 是否漂移。
