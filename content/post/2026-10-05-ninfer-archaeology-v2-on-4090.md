---
title: 4090 上的 NInfer 考古：从 v3 容器里挖出 v2 真身
description: 官方 artifact 全是 v3，4090 fork 只认 v2——用 git 元数据在 HuggingFace 仓库历史里考古出旧版模型文件，单流 decode 从 37 拉到 90 tok/s。
date: 2026-10-05T12:00:00+08:00
tags: [LLM, 推理优化, 4090, HuggingFace]
---

## 背景：一个"只支持未来格式"的推理引擎

NInfer 是一个 5090 专属（sm_120a）的 Qwen 推理引擎，靠 tcgen05 和手写 kernel 把 decode 速度拉到不可思议的数字。社区有人为 3090/4090 做了 fork（sm_89），但有个尴尬的问题：**官方发布的 `.ninfer` 模型文件全是 v3 容器格式，而这个 fork 只认 v1/v2**——直接下载就是一句报错：

```
artifact magic is not NInfer v1 or v2
```

下什么挂什么。HF 上能搜到的四五个现成 artifact（不同作者、不同基座）无一例外全是 v3。

## 考古姿势：git 元数据绕过 API 限频

HuggingFace API 对匿名请求限频很凶，批量查历史 commit 容易 429。正解是把它当 git 仓库看：

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone https://hf-mirror.com/<org>/<model>
git log --oneline -- <model-file-path>
```

`GIT_LFS_SKIP_SMUDGE=1` 保证只拉元数据不拉权重（18G 的东西扫历史不需要真的下载）。在历史里找到可疑的旧 commit 后，两个证据定案：

1. 该 commit 下模型文件大小 = **18,210,531,328 字节**——与 v2 时代的体量完全吻合；
2. 直接走 `resolve/<commit>/<path>` 下载，fork 吃下，报错消失。

经验：**HF 的 git 层比 API 层耐用得多**，限频、校验、历史考古都优先走 git。

## 构建四补丁

fork 在 Linux/4090 上 Docker 构建有四个坑，逐个补：

1. build 层加 `git curl`（后续步骤要用）；
2. CMake < 3.30 遇到 CMP0169 直接拒绝，注入 3.31.6 二进制；
3. 镜像 CUDA 13.1 > 宿主驱动 13.0，运行时加 `NVIDIA_DISABLE_REQUIRE=1`；
4. `ninfer-serve` 默认绑 127.0.0.1，容器里端口映射直接不通，必须 `--host 0.0.0.0`。

## 结果：37 → 90 tok/s

同一块 48G 魔改 4090，同一模型（Qwen3.8-27B）：

| 引擎 | decode | 备注 |
|---|---|---|
| vLLM FP8 | ~37 tok/s | continuous batching，多并发王者 |
| NInfer v2 artifact | **90-91 tok/s** | 单流，MTP 接受率 31%，draft-tokens=4 + lm-head-draft |

2.45 倍。原理不神秘：单流 decode 大家都被显存带宽卡脖子，NInfer 用激进 MTP 投机解码让一次权重搬运摊出多个 token，再用 int8 KV 减半读取量——**这是单流场景唯一能"破墙"的路**。

## 边界要讲清楚

单模型、并发 1-8、没有 continuous batching。多用户高并发的聚合吞吐，vLLM 依然完胜（我们 8 并发实测聚合 ~295 tok/s）。选型结论一句话：**聊天/单 agent 要延迟用 NInfer，多 agent 要吞吐用 vLLM**。
