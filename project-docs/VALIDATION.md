# VALIDATION — H3 本地加速

## V1. 原仓库代码零改动（PASS）

```
$ git diff --numstat | awk '{a+=$1;d+=$2} END {print "insertions:",a," deletions:",d}'
insertions: 0  deletions: 0
```
78 项 `git status` 状态全部来自文件权限位变化（macOS `._*` 资源分叉文件副作用），无任何内容修改。

## V2. integration 分支构建（PASS）

```
$ cd ~/h3c-build/h3-integration && make -j8
clang -o h3 main.o h3_cli.o h3_mutex.o linenoise.o ... -framework Metal ...
ar rcs libh3.a ...
```
仅 1 个警告：`h3_gpu.m:273:54: use of GNU ?: conditional expression extension`。退出码 0。

## V3. 新二进制兼容原权重（PASS）

```
$ ./h3 --info -d ".../h3.c/MiniMax-H3"
h3-metal 0.1.0-dev
Device: Apple M5 Pro (applegpu_g17s)
  physical memory       48.0 GiB
  recommended GPU set   37.4 GiB
  max Metal buffer      28.1 GiB
  Apple GPU family      10
  Metal 4               yes
Native checkpoint inventory (header-only):
  Qwen3-VL encoder   14 files  1058 tensors   62.133 GiB
  FL2VA DiT          13 files   535 tensors   61.728 GiB
  Ref2VA DiT         13 files   535 tensors   61.728 GiB
  video VAE           1 files   560 tensors    9.700 GiB
  audio VAE           1 files  1087 tensors    0.564 GiB
```

权重目录未复制、未改动，新二进制直读。

## V4. bf16 VAE 生效（PASS）

冒烟测试 512×512 / 22 帧 / 6 步 / 45 层：

```
h3: resident video VAE tiles 3x3 at 256 pixels
h3: video VAE prefetched during denoise
h3 profile: video VAE decode pipelined fused=1 tiles=9 chunks=1 wall=5.780s
h3 profile: resident video VAE decoder total  wall=30.893s encode=5.641s wait=8.213s peak=4.988GiB
```

- **VAE 解码 5.78 秒**（fused pipeline 计入）
- **prefetch 生效**：`video VAE prefetched during denoise`（解码器加载与去噪重叠）
- **256px 瓦片生效**：`tiles 3x3 at 256 pixels`
- VAE 峰值显存 4.988 GiB

## V5. 差分对照（上游 vs integration）— 冒烟规模

参数完全一致：512×512、22 帧、6 步、45 层、同 prompt、同 seed（默认 42）。

| 指标 | 上游 8974cc0 | integration 8e682e8（run1） | integration（run2 复测） |
|---|---|---|---|
| Qwen 文本编码 | 75.445 s | ~75 s | 75.218 s |
| DiT 加载 | 116.534 s | 118.285 s | 128.127 s |
| **GPU Euler 去噪** | **13.308 s** | 24.027 s | **22.974 s** |
| audio VAE | 0.732 s | 0.820 s | — |
| **video VAE 解码** | **26.131 s**（独立） | **5.780 s**（流水线） | **6.097 s** |
| video VAE 峰值 | 9.454 GiB | 4.988 GiB | 4.988 GiB |
| DiT 峰值 | 16.825 GiB | 16.825 GiB | 16.825 GiB |

### 观察与已定结论

- ✅ **VAE 解码快 4.3–4.5 倍**（26.1 s → 5.8–6.1 s 流水线 / 23.2 s 独立），峰值显存从 9.45 GiB 降到 4.99 GiB
- ✅ **去噪回归的根因已定位：VAE 预取抢 GPU**。`H3_VAE_PREFETCH=0` 后去噪恢复 12.994 s（上游 13.308 s，差异 0.3 s 即噪声）
  - 预取开启：去噪 22.974 / 24.027 s（两次复现）
  - 预取关闭：去噪 **12.994 s**
  - 预取在去噪期间加载 4.8 GiB 解码器，与本就在等 MPSGraph 的去噪争抢统一内存带宽

### 三组对照（冒烟规模：512×512 / 22 帧 / 6 步 / 45 层）

| 配置 | DiT total | video VAE | 后段合计 |
|---|---|---|---|
| 上游 8974cc0（F32 VAE，无预取） | 130.128 s | 26.131 s（独立） | **156.3 s** |
| integration，预取 ON，bf16 VAE | 153.724 s | 6.097 s（流水线） | 159.8 s |
| integration，预取 OFF，bf16 VAE | 135.322 s | 23.210 s（独立） | **158.5 s** |

### 关键分析：为什么冒烟规模看不出优势

**预取的成本是固定值，VAE 的收益随片长放大。**

- 预取成本 ≈ 10 s（一次加载 4.8 GiB 解码器的固定开销），不随去噪步数/帧数变化
- 冒烟规模 VAE 只有 26 s，省下的 3 s 抵不过预取的 10 s → 净亏
- 生产规模（768×1344 / 124 帧 / 8 步）：去噪 1087–1272 s，VAE 77–264 s。此时预取 10 s 成本摊薄到 <1%，而 VAE 节省 180 s 以上 → **净赚**

**因此**：
- 小片/预览（≤32 帧、≤640²）→ 用 `H3_VAE_PREFETCH=0`，避免预取回归
- 生产片（≥100 帧、≥768×1344）→ 保持预取默认开启
- 冒烟规模的"净亏 2.3%"**不可外推为生产结论**

### 尚未完成

生产规模（768×1344 / 124 帧 / 8 步）的本机 A/B 对照**尚未执行**。社区在同型机上报 1.35×，但那是他机数据。本机生产规模实测是升级决策前的必做项。

---

## V6. 中间点实验（Opus P0-2）：768×1344 / 49 帧 / 8 步 / 50 层

按 Opus 要求跑中间点，确定预取策略的切换分界。参数：`--width 768 --height 1344 --frames 49 --steps 8 --layers 50`（实际产出 56 帧 / 2.333 s）。

### 结果对照（完整墙钟）

| 阶段 | 预取 ON | 预取 OFF | 差值 |
|---|---|---|---|
| Qwen 文本编码 | 77.058 s | 77.180 s | ≈0 |
| DiT 加载 | 128.053 s | 128.208 s | ≈0 |
| **GPU Euler 去噪** | **465.277 s** | **480.415 s** | **ON 快 15.1 s** |
| DiT total | 598.566 s | 609.030 s | ON 快 10.5 s |
| **video VAE 解码（关键路径）** | **74.047 s**（流水线，3 chunks） | **102.013 s**（独立） | **ON 快 28.0 s** |
| output tail | 84.877 s | 104.560 s | **ON 快 19.7 s** |
| DiT 峰值显存 | 20.253 GiB | 20.253 GiB | 相同 |
| VAE 峰值显存 | 4.988 GiB | 4.968 GiB | ≈相同 |

**后段合计（DiT total + output tail）**：
- ON：598.566 + 84.877 = **683.4 s**
- OFF：609.030 + 104.560 = **713.6 s**
- **ON 快 30.2 s（−4.2%）**

> 注：ON 的 `resident video VAE decoder total wall=597.802s` 是**含等待的全程计时**（解码器在去噪期间就预取并驻留），不是关键路径耗时。关键路径看 `output tail streamed wall=84.877s`。

### 关键结论：预取在中间点明确胜出 ✅

| 规模 | 预取对去噪的影响 | 预取对总时长的影响 |
|---|---|---|
| 冒烟 512²/22 帧 | 慢 10 s（惩罚） | 慢约 3.6 s |
| **中间点 768×1344/49 帧** | **快 15 s（收益）** | **快 30.2 s（−4.2%）** |

**Opus 的推断被本机实测证实**：预取成本固定（4.8 GiB 解码器加载），收益随解码工作量放大。交叉点在 512²/22 帧与 768×1344/49 帧之间。

**策略确立**：
- 预览/小片（≤32 帧、≤640²）→ `H3_VAE_PREFETCH=0`
- 中间点及以上（≥49 帧或 ≥768×1344）→ **保持预取默认开启**

### 画面一致性验证（PASS，强证据）

ON 与 OFF 两轮输出做逐帧比对：

```
$ ffmpeg -i prefetch-ON.mp4 -i prefetch-OFF.mp4 -filter_complex "[0:v][1:v]psnr=stats_file=/tmp/psnr.log" -f null -
n:56 mse_avg:0.00 psnr_avg:inf psnr_y:inf psnr_u:inf psnr_v:inf   ← 全部 56 帧

$ 帧哈希
prefetch-ON : 0845e442d3ebaccbc18e0ee0ba7d446a
prefetch-OFF: 0845e442d3ebaccbc18e0ee0ba7d446a   ← 完全相同
```

**预取开关只改变时序，不改变任何一帧像素。** 这印证了 README 的说法（"Prefetch and cache are bit-identical"），也意味着预取策略可以按性能自由切换，没有画质风险。

### 画面质量目检

首帧（768×1344，含人物面部与大量发丝细节）：无矩形马赛克、无 16px 瓦片接缝、发丝细节完整、面部无异常。256px 瓦片默认生效，未出现社区普查中 #70 描述的"浓密毛发矩形马赛克"。

### 待补

ON 轮次的完整日志（含 `video VAE decode pipelined` 行）已重跑并落盘 `outputs/midpoint/on.log`，用于计算总墙钟。OFF 轮完整日志在 `outputs/midpoint/off.log`。

## V6. 产出文件有效性（PASS）

```
$ ffprobe outputs/smoke-integration.mp4
codec_name=h264  512x512  24/1
codec_name=aac
duration=0.925000  size=340847
```

H.264 + AAC 双轨，24 fps，时长 0.925 s（22 帧 @24fps，符合 `5+17n` 帧对齐规则）。

## V7. TurBo LoRA 适配器可用性（PASS，未下载）

`larryvrh/MiniMax-H3-Turbo-Lora` 提供 8 个 `.safetensors`，每个 **743.7 MiB**：

- `minimax_h3_turbo_v4_step600_ema.safetensors` ← PR #14 文档指定的那个
- 其余为 4step / ckpt500 / ckpt850 / ema 变体

`tools/fold_turbo_lora.py --help` 正常，参数为 `--checkpoint / --lora / --out / --scale`。

## V7. 三镜头阶梯验证（Opus P0-3）— PASS

按 Opus 要求的四条判据，用 integration 版（bf16 VAE + 预取默认开 + Sol-Attn 关闭）跑三条代表镜头。

参数统一：768×1344 / 49 帧请求（实出 56 帧 / 2.333 s）/ 8 步 / 50 层 / seed 42

| # | 镜头 | prompt 要点 | 目的 |
|---|---|---|---|
| s1 | 人物口播 | `...She says: This is a test of the new engine.` | 带语音（Opus 硬要求） |
| s2 | 浓密毛发 | `golden retriever shaking its head, dense fur detail` | 对应 #70 马赛克缺陷 |
| s3 | 快速运动 | `dancer spins quickly across a wooden stage, flowing dress` | Sol-Attn 运动僵化失败模式 |

### 四条判据结果

| 判据 | s1-speech | s2-fur | s3-motion |
|---|---|---|---|
| ① 无矩形马赛克/瓦片接缝 | ✅ | ✅ | ✅ |
| ② 唇形无明显漂移 | ✅ 嘴型从张到收完整 | N/A | N/A |
| ③ 无逐帧闪烁/呼吸 | ✅ | ✅ | ✅ |
| ④ 运动幅度未下降 | ✅ | ✅ 甩头动作明显 | ✅ 裙摆大幅旋转 + 动态模糊 |
| **结果** | **PASS** | **PASS** | **PASS** |

**3/3 通过，超过 Opus 设定的 ≥2/3 通过线。**

### 逐项证据

**s1 口播**：第 10 帧张嘴（说到 "test"）、第 40 帧嘴收拢，中间嘴型连续变化。人物直视镜头，无眼神飘移。发丝细节完整。音频 32 kHz 双声道 AAC。

**s2 毛发**：第 25 帧侧头、第 45 帧回正甩头，毛发连续过渡。放大 2 倍检查腹部毛发区域（`crop=300:300:230:950,scale=600:600:flags=neighbor`）：**无块状网格、无矩形接缝、无 16px 瓦片痕迹**。#70 在本机这批配置下未复现。

**s3 运动**：第 25 帧裙摆大幅展开成扇形、有明显动态模糊、舞者姿态清晰。**平均帧间亮度差 9.65**（s1 口播 3.37、s2 毛发 9.11），运动幅度正常，未出现 Sol-Attn 常见的"运动变小、变僵"。

### 性能数据

| 镜头 | DiT total | video VAE 解码 | 峰值显存 |
|---|---|---|---|
| s1-speech | 648.903 s | 76.949 s | 20.255 GiB |
| s2-fur | 595.124 s | 62.048 s | 20.253 GiB |
| s3-motion | 527.905 s | 64.106 s | 20.253 GiB |

三条平均：DiT 约 591 s，VAE 解码约 68 s，端到端约 11 分钟/条（768×1344 / 2.33 秒片）。

### 一个重要发现（非引擎差异，属 prompt 行为）

`s1-speech` 的画面明显优于 `prefetch-ON2`（同参数、同 seed），原因是 **prompt 多了具体台词**：

- `prefetch-ON2`：`...subtle head movement.` 结束 → 模型不确定要不要说话，嘴半张、眼神飘移
- `s1-speech`：`...She says: This is a test of the new engine.` → 模型明确执行口型动画，嘴型从张到收有完整轨迹、直视镜头

**结论：H3 做口播类镜头，prompt 必须带具体台词句子，只说 "speaks one line" 不够。** 这条应写进 `local-h3-video` skill。

---

## 失败记录

### F1. 后台任务用 `nohup &` 导致进程被外层带走（已修复）

第一次冒烟测试用 `nohup ./h3 ... &` 启动，外层 shell 退出后 h3 进程消失，无输出文件。
违反已知纪律（ZCode 后台任务禁用 nohup &）。改用工具的 `run_in_background` 直接跟踪真身，成功。
