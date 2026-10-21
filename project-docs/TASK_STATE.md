- task_id: h3c-local-acceleration-20261021
- objective: 在本地 M5 Pro 48GB 上把 MiniMax-H3 出片速度提上来，验证"本地优化即够、不必租 AutoDL GPU"，产出可复用加速分支 + 本机实测 + 决策文档
- status: active
- phase: P3-p0-complete
- next_action: P0 三项验证全部通过。下一步=P1（pin commit + sha256 入构建脚本、建 PR#73 单点回退分支）或 P2（折叠 Turbo LoRA 单独测），按 owner 决定
- recovery_token: h3c-accel-p0-all-pass-3of3-ladder
- compression_count: 0
- max_compressions_per_session: 2
- session_generation: 1
- last_verified: 2026-10-21
- handoff_file: none
- read_next:
  - TASK_STATE.md
  - DECISIONS.md
  - VALIDATION.md
  - AUDIT.md
  - ../artifacts/02-plan/H3-本地加速方案-20261021.md

# TASK_STATE — H3 本地加速（替代租 GPU 方案）

## 目标

在本地 M5 Pro 48GB 上把 MiniMax-H3 出片速度提上来，结论是"本地优化就够、不必租 AutoDL GPU"。
产出：可复用的加速分支 + 本机实测数据 + 决策文档（交 Opus 5.5 审核）。

## 硬约束

- 不覆盖 `/Users/zhuozhi/Documents/自媒体/AI 漫剧/h3.c` 现有环境（上游 8974cc0 + 269G 权重）
- 新分支只放 `~/h3c-build/h3-integration`，验证通过后才谈切换
- 权重不复制（复用原 `MiniMax-H3` 目录）
- 任何"已提速"结论必须有本机实测数据支撑，不引用他人机器数字当自家结论

## 当前阶段

P3 = **Opus 的 P0 三项验证全部通过**，等待 owner 决定下一步（P1 工程加固 或 P2 Turbo LoRA）。

## Opus P0 三项验证结果

| 项 | 结果 |
|---|---|
| P0-1 供应链审计 | ✅ 无网络出站 / 无敏感路径 / 无混淆 / 进程仅 ffmpeg（详见 `AUDIT.md`） |
| P0-2 中间点实验 | ✅ 预取 ON 快 30.2 s（−4.2%），画面位元一致（PSNR=inf，帧哈希相同） |
| P0-3 三镜头阶梯 | ✅ **3/3 通过**（条线是 ≥2/3） |

### P0-3 四判据结果

| 判据 | s1-speech | s2-fur | s3-motion |
|---|---|---|---|
| 无马赛克/瓦片接缝 | ✅ | ✅ | ✅ |
| 唇形无漂移 | ✅ | N/A | N/A |
| 无闪烁/呼吸 | ✅ | ✅ | ✅ |
| 运动幅度未降 | ✅ | ✅ | ✅ |

三条片子：`~/h3c-build/h3-integration/outputs/ladder/s{1,2,3}-*.mp4`

**关键**：社区普查提到的 #70（浓密毛发矩形马赛克）在本机**未复现**。256px 瓦片工作正常。

## 已完成事实

| 项 | 结果 | 证据 |
|---|---|---|
| 原仓库代码未改动 | 78 项状态全是文件权限位变化，insertions=0/deletions=0 | `git diff --numstat` 合计 0/0 |
| 上游停止维护 | 最后提交 2026-08-11 (8974cc0)，29 个 PR 全未合 | GitHub API `pushed_at` |
| 社区生态规模 | 234 fork / 62 个带独有提交 / 80 特性分支 / 29 开放 PR | `docs/fork-census-2026-10-21.md` |
| integration 分支可构建 | `make -j8` 通过，仅 1 个 GNU 扩展警告 | 构建日志 |
| 新二进制兼容原权重 | `--info` 正确报出全部 checkpoint，Metal 4 = yes | `--info` 输出 |
| 端到端出片成功 | H.264+AAC / 512×512 / 24fps / 0.925s / ffmpeg 解码无错 | `outputs/smoke-integration.mp4` |
| **本机 A/B 已完成** | 见下节与 `VALIDATION.md` V5 | 三组对照实测 |
| VAE 回归根因定位 | `H3_VAE_PREFETCH=0` 使去噪从 22.97s 回到 12.99s | 隔离测试 |
| Turbo LoRA 适配器可用 | `larryvrh/MiniMax-H3-Turbo-Lora`，每个 743.7 MiB，`fold_turbo_lora.py` 参数正常 | HF API + `--help` |

## 本机 A/B 结论（冒烟规模 512×512/22帧/6步）

| 配置 | DiT total | video VAE | 后段合计 |
|---|---|---|---|
| 上游（F32 VAE） | 130.128 s | 26.131 s 独立 | **156.3 s** |
| integration 预取 ON | 153.724 s | 6.097 s 流水线 | 159.8 s |
| integration 预取 OFF | 135.322 s | 23.210 s 独立 | **158.5 s** |

**冒烟规模集成版反而慢约 2%**。根因=预取固定成本约 10 s 与去噪争抢统一内存带宽。
推断（**未验证**）：生产规模下预取成本摊薄、VAE 收益放大（省 180 s+），应净赚。

## 关键决策

见 `DECISIONS.md`。核心三条：
1. 走高优先级路线 = Bambushu/integration（专为 M5 Pro 48GB 写的分支）+ 折叠 Turbo LoRA，而不是租 GPU
2. 原环境不动，新分支独立目录验证
3. 所有提速数字分"社区实测"与"本机实测"两类，不混用

## 未决 / 风险

- **生产规模（768×1344/124帧/8步）本机 A/B 未做** ← 升级决策前必做
- integration 在相同 seed 下与上游输出不一致（GQA 竞态修复 #63）
- 折叠 Turbo LoRA 的 5–6 步质量未评审
- Sol-Attn 换内核后固定 seed 失效
- Opus 审核意见待回

## 下一步 (next_action)

等 Opus 5.5 审核返回 → 按裁决调整方案 → 若裁决要求补生产规模实测，则跑 768×1344/124帧对照 → 回填 `VALIDATION.md` 与主文档 → 更新 TASK_STATE。

## read_next 白名单

- `.task/TASK_STATE.md`（本文件）
- `.task/DECISIONS.md`
- `.task/VALIDATION.md`
- `artifacts/02-plan/H3-本地加速方案-20261021.md`

## handoff_file

（暂无，尚未触发硬交接）

## 产物位置

- 加速分支源码：`~/h3c-build/h3-integration`（8e682e8）
- 原环境（未动）：`/Users/zhuozhi/Documents/自媒体/AI 漫剧/h3.c`（8974cc0）
- 权重（共用）：`/Users/zhuozhi/Documents/自媒体/AI 漫剧/h3.c/MiniMax-H3`
- 方案文档：`artifacts/02-plan/H3-本地加速方案-20261021.md`
- 冒烟输出：`~/h3c-build/h3-integration/outputs/smoke-*.mp4`、`/tmp/h3-ab/smoke-upstream.mp4`
- 审核会话：`/tmp/opus_review_raw.txt`
