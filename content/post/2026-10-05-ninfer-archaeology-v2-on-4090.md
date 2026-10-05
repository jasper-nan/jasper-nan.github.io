---
title: 4090 上的 NInfer 考古：从 v3 容器里挖出 v2 真身
description: 官方 artifact 全是 v3，4090 fork 只认 v2——用 git 元数据在 HuggingFace 仓库历史里考古出旧版模型文件，再叠加 MTP 调优，单流 decode 从 37 一路拉到 251 tok/s。含双引擎盲测、工具调用对照与完整压测数据。
date: 2026-10-05T12:00:00+08:00
tags: [LLM, 推理优化, 4090, HuggingFace]
---

## 背景：一个"只支持未来格式"的推理引擎

NInfer 是一个 5090 专属（sm_120a tcgen05）的 Qwen 推理引擎，靠手写 kernel 把 decode 速度拉到不可思议的数字。社区有人为 3090/4090 做了 fork（sm_89），但有个尴尬的问题：**官方发布的 `.ninfer` 模型文件全是 v3 容器格式，而这个 fork 只认 v1/v2**——直接下载就是一句报错：

```
artifact magic is not NInfer v1 or v2
```

下什么挂什么。HF 上能搜到的现成 artifact——DreamFast、gearwave、kvnxiao、Yuuyuuyuuyuu 几个作者、不同基座——无一例外全是 v3 容器，下了也白下（我还真下了一份 18G 的 DreamFast 留在盘里，等 fork 升级 reader）。

## 考古姿势：git 元数据绕过 API 限频

HuggingFace API 对匿名请求限频很凶，批量查历史 commit 容易 429。正解是把它当 git 仓库看：

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone https://hf-mirror.com/<org>/<model>
git log --oneline -- <model-file-path>
```

`GIT_LFS_SKIP_SMUDGE=1` 保证只拉元数据不拉权重（18G 的东西扫历史不需要真的下载）。在历史里定位到旧 commit `3526913` 后，两个证据定案：

1. 该 commit 下模型文件大小 = **18,210,531,328 字节**——与 v2 时代的体量完全吻合；
2. 直接走 `resolve/3526913/<path>` 下载，fork 吃下，报错消失。

顺带一提：这批 orcarotor 系仓库是 gated=auto，自助开门用 `POST /{repo}/ask-access`，返回 303 即成。

经验：**HF 的 git 层比 API 层耐用得多**，限频、校验、历史考古都优先走 git。

## 构建四补丁

fork 在 Linux/4090 上 Docker 构建有四个坑，逐个补：

1. build 层加 `git curl`（后续步骤要用）；
2. CMake < 3.30 遇到 CMP0169 直接拒绝，注入 3.31.6 二进制；
3. 镜像 CUDA 13.1 > 宿主驱动 13.0，运行时加 `NVIDIA_DISABLE_REQUIRE=1`；
4. `ninfer-serve` 默认绑 127.0.0.1，容器里端口映射直接不通，必须 `--host 0.0.0.0`。

## 第一轮：vLLM 37 → NInfer 90 tok/s

同一块 48G 魔改 4090（nvidia-smi 实测 49140MiB），同一模型 Qwen3.8-27B：

| 指标 | vLLM FP8 | NInfer v2 artifact | 倍数 |
|---|---|---|---|
| 单流 decode | ~37 tok/s | **90.5-91.5 tok/s** | 2.45x |
| TTFT | — | ~120 ms | — |
| Prefill | — | ~620 tok/s | — |
| MTP 接受率 | —（MTP 1-2） | 31%（draft-tokens=4） | — |
| 8 并发聚合 | **~295 tok/s** | 不适用 | — |

原理不神秘：单流 decode 大家都被显存带宽卡脖子（每搬一次权重出一个 token），NInfer 用三根杠杆破墙——**激进 MTP 投机**（一次权重搬运摊出多个 token，唯一破墙法）+ **sm_89 手搓融合 kernel** + **int8 KV 减半读取量**。

启动命令：

```bash
docker run -d -e NVIDIA_DISABLE_REQUIRE=1 --gpus all -p 18080:8080 \
  -v ~/lx/ninfer-artifacts:/workspace:ro ninfer-4090:sm89 \
  ninfer-serve /workspace/qwen3_8_27b_v2.ninfer \
  --model-id qwen3_8_27b --host 0.0.0.0 \
  --kv-dtype int8 --spec mtp --draft-tokens 4 --lm-head-draft \
  --max-context 262144 --preserve-thinking
```

## 第二轮：draft-tokens 4→6 + rk4v4-e8，251 tok/s

把启动参数从 `int8 KV / draft4 / 262144` 调成 `rk4v4-e8 KV / draft6 / 409600`，收益立竿见影：

| 场景 | 调优前 | 调优后 | 备注 |
|---|---|---|---|
| 中文散文 decode | 200 tok/s | **251 tok/s** | MTP 接受率 94%，6.65 tok/round |
| Agent 实流（工具调用类） | — | 179-194 tok/s | 接受率 65-94% |
| 纯推理链 decode | 117 tok/s | 109 tok/s | ⚠️ 接受率掉到 32%，负收益 |
| 显存常驻 | 27.6G | 25.8G | 4-bit KV 更省 |
| 长上下文针插测试 | — | 360K 通过 | 4-bit KV 质量在线 |

**代价要说清楚**：draft6 对"纯思考链"流量是亏的（draft token 在推理场景猜不中），但对聊天/agent 流量是大赚。400K 以上属 rope 外推未验证区，别再往上拉。

## 双引擎盲测：智商有没有掉？

速度翻了几倍，质量不能缩水。4 道题双引擎盲比（数学 / 代码 / 陷阱题 / 逻辑谜题）：

- **4:4 平手，全对**，思考链风格一致；
- 推理负载实测 NInfer 108-161 tok/s（vLLM 同批 ~55 tok/s）；
- 中文散文质量与 FP8 原版无感差异。

工具调用 5 用例双引擎对照 **5:5 平**：OpenAI tools 单工具选择 / 多工具选择 / 多参数精度 / 工具回环全通，**Anthropic `/v1/messages` 的 tool_use 块也通**（Claude Code 可直连），该轮实测 121-180 tok/s。

## 边界与选型

单模型、并发 1-8、没有 continuous batching。选型结论一句话：

| 场景 | 选择 | 理由 |
|---|---|---|
| 聊天 / 单 agent（要延迟） | NInfer | 单流 90-251 tok/s，TTFT 120ms |
| 多 agent 高并发（要吞吐） | vLLM | continuous batching，8 并发聚合 ~295 tok/s |

我最终的日常配置：NInfer 常驻当 OpenCode/pi 的本地端点，vLLM 留作回滚底——实测切换一条 docker 命令的事，但三个月没回滚过。
