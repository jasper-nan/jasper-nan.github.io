---
title: 一块 4090 的第二职业：MiniMax-H3 本地视频生成流水线
description: 同一块 48G 4090，白班跑 LLM 端点，夜班生成视频——MiniMax-H3 音画一体模型本地部署全记录：ModelScope 抢权重、24 步砍到 4 步（29 分钟 → 5 分钟）、与常驻 LLM 的显存/内存互斥调度，以及骨骼复用的动作复刻流水线。
date: 2026-10-06T12:10:00+08:00
tags: [ComfyUI, 视频生成, MiniMax-H3, 4090]
---

## 一块卡的两份工作

这块 48G 魔改 4090 此前的主业是 LLM 推理端点——前两篇写过它的单流提速（NInfer 考古）和一次深夜 OOM 事故（血训）。这次给它加一份第二职业：**MiniMax-H3**，MiniMax 开源的音画一体生成模型，ComfyUI 0.38 起原生支持（`MiniMaxH3ImageToVideo` / `MiniMaxH3SigmaShift` / `EmptyMiniMaxH3LatentAV` 等节点直接进核心），官方模板里 i2v/t2v 都有。

选它不选 Wan2.2-5B 的理由很直接：开源可本地跑、图生视频**带音频**、实拍对比下来运镜和角色一致性明显更强。代价也明确——它是个大家伙，而且和 LLM 端点互斥，这个后面细说。

## 抢权重：hf-mirror 80KB/s vs ModelScope 30MB/s

权重不用官方原版，用 Comfy-Org/MiniMax-H3 的量化重组版：

| 文件 | 体积 | 角色 |
|---|---|---|
| `qwen3vl_32b_minimax_h3_nvfp4_awq` | 15G | 文本编码器 |
| `minimax_h3_fl2va_pruned_int8_convrot` | 20G | 扩散 transformer |
| `minimax_h3_video_vae_int8_convrot` + `audio_vae_fp32` | — | 双 VAE（视频 + 音频） |
| `minimax_h3_fl2v_turbo_4step` | 小 | 4 步加速 LoRA |

下载环节两个坑，各值一段：

1. **hf-mirror 对这个仓库限速到 80KB/s**——15G + 20G 的体量按这个速度是天数级。换 ModelScope 的同款镜像（`Comfy-Org/MiniMax-H3`），30MB/s 直下，几分钟的事。
2. **`wget -c` 断点续传严禁跨站混源**。hf-mirror 的残块接着 ModelScope 的 URL 续传，文件必坏——症状是 CLIPLoader 报 utf-16 解码错误。混过的残块必须删干净重下，别试图抢救。

## 提速三连：29 分钟 → 5 分钟

基准：1344×768、124 帧（约 5 秒视频），帧数按 H3 的 17k+5 latent 网格取值。

| 优化 | 配置 | 耗时 | 备注 |
|---|---|---|---|
| 基线 | 24 步全量 | **≈29 分钟**（66s/it） | 质量满分档 |
| turbo LoRA | 4 步 + cfg 1.0 + euler/simple | **≈5 分钟**（热机约 4 分钟） | 日常主力档 |
| + sageattention | 启动加 `--use-sage-attention` | 27s/it vs 34s/it（**快 20%**） | 首步 triton 编译 ~100s |

sageattention 有个隐蔽门槛：**装了包不等于生效**，必须在 ComfyUI 启动命令里显式加 `--use-sage-attention`，否则静默走 pytorch attention，你还在纳闷为什么白装了。另外负面提示词走独立的 `CLIPTextEncode`，`MiniMaxH3ImageToVideo` 一步完成提示词编码 + 首帧注入 + latent 生成，不用手工拼三段。

音频链路（`VAEDecodeAudio` + CreateVideo 的 audio 输入）本次还没实测，留待下回——理论上 5 秒视频白送一条音轨，不升白不升。

## 显存与内存账本：一张卡的两份全职

这是整篇最贵的一课。H3 三大件常驻 **~38G 显存**，系统内存也被吃掉一大块（这台机器只有 46G RAM）；而这块卡的白班是 NInfer LLM 端点（常驻 26G）。两者**互斥**：

- H3 跑完占住 38G 不放 → NInfer 26G 启动即崩，而且是崩溃重启循环；
- 反过来，H3 推理时系统 RAM 撞顶，Linux OOM Killer **优先杀 ComfyUI——进程无声死亡：日志无报错、进度条正常结束**，下次请求才发现服务早没了。任何高内存旁路操作（比如同时开十几路 ffmpeg 解码剪片）都可能触发。

调度方案最终定型为一条交接班纪律 + 一个看门狗：

```bash
# 夜班开始前
docker stop ninfer-serve
# 夜班结束后：先重启 ComfyUI 释放 38G 显存，再交还白班
docker start ninfer-serve
```

看门狗 = 每分钟一次 crontab 检查 ComfyUI 进程，不在就拉起，日志落盘。防的是 OOM 无声死亡后的"服务裸奔"——不然用户下次点生成才发现没人应答。

这和前一篇 MTU 看门狗是同一个设计哲学：**凡是无声失败的环节，都要一个周期性探针兜底。**

## 进阶流水线：动作复刻，骨骼一次、换肤 N 次

H3 还有一条更野的玩法：**FunControlNet 动作复刻**——拿一段真人视频，把里面的动作"套"到任意角色上。短视频平台那些"同一支舞 N 个版本"的二创，就是这个原理。

流水线五段：

1. **取材**：yt-dlp 从 B 站下载参考视频（B 站有开放搜索 API，比第三方 CLI 好用）；
2. **切帧**：243 帧 @24fps（落在 17k+5 网格上，约 10 秒）；
3. **骨骼提取**：SDPose wholebody 模型（fp16 主干 + rt_detr 检测头）把真人动作转成骨骼视频；
4. **条件注入**：`MiniMaxH3FunControlNetApply`（model_patches 2.3G）+ Ref2VA transformer（21G int8）+ 4 步 turbo LoRA；
5. **生成**：每条约 10 分钟。

两条昂贵教训：

- **姿态提取和 H3 生成不能挤在同一个 prompt 里跑**——两者叠加内存随机撞顶 OOM。必须拆成两段：先出骨骼 mp4，拷进 `input/` 目录，再用 LoadVideo 作为 control_video 喂第二段。拆开后配合看门狗，批量连跑零失败。
- **骨骼视频是可复用资产**：一支舞的骨骼存一次，换提示词就能批量换角色，每次只花生成时间。这是整条流水线性价比最高的一步。

## API 批量调用的三个坑

想用脚本批量跑（HTTP API 提交 workflow JSON），有三个坑要绕：

1. LoadVideo 节点的输入参数名是 **`file`**，不是 `video`；而且它**只读 `input/` 目录**，SaveVideo 写 `output/`——跨阶段传递文件要自己 cp；
2. `MiniMaxH3ReferenceToVideo` 的 **ref_images 是 Autogrow 动态输入**，API 传参会触发 ComfyUI 的嵌套化 bug（`ref_image_0` 不被正确展开）——**锁脸参考图这条路只能走 UI 手动拖**，纯 API 路线用文字描述人设代替；
3. 节点必填项别靠猜：`GET /object_info/<NodeType>` 一发就有。缺必填项的症状是 `prompt_outputs_failed_validation`，报错不会告诉你是哪个字段缺了。

## 结论

48G 4090 的分工最终定档：**LLM 端点是白班（26G 常驻），视频生成是夜班（38G 独占），交接班靠一条 docker 命令，加班餐是每分钟一次的看门狗探针。**

如果只有一块卡又要兼顾两端，核心纪律就三条：互斥服务用启停调度，别硬共存；无声失败必须有探针兜底；断点续传别跨站混源。
