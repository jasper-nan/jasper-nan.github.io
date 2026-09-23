---
title: 我把 Linux 桌面切成了 47 个二次元世界：从原神到芙莉莲的主题工程
description: 25 个手作主题、4K 暗带壁纸验收流水线、终端/开机画面/AI 终端全家桶联动、每工作区一个世界——Omarchy 主题工程的完整折腾实录。
date: 2026-09-23T12:40:00+08:00
tags: [Linux, Omarchy, Hyprland, 主题美化, 二次元]
---

> 切一次主题，壁纸、顶栏、四个终端、AI 助手、系统监视器、Obsidian、浏览器、输入法、开机画面、登录界面——全部换装。这就是我这台 Arch 的日常。

* * *

[上篇文章](/p/omarchy-genshin-themes-11-characters-open-source/)写完原神主题开源后，这事儿就收不住了。如今主题库膨胀到 **47 个**：14 个原神角色 + 11 部动漫 IP + 22 个官方自带。这篇补上后半场的硬货——联动全家桶、开机画面、以及最近折腾出来的"每工作区一个世界"。

一、主题版图

| 系列 | 数量 | 成员 |
|---|---|---|
| 原神 | 14 | 雷电将军、甘雨、胡桃、钟离、纳西妲、温迪、芙宁娜、神里绫华、魈、夜兰、可莉、刻晴、琴、宵宫 |
| 动漫 | 11 | 芙莉莲、电锯人、咒术回战、崩坏·星穹铁道、鸣潮、海贼王、火影忍者、孤独摇滚、进击的巨人、柯南、鬼灭之刃 |
| 官方 | 22 | Omarchy 自带 |

选 IP 不看心情看**数据**：写脚本探测 Wallhaven 上各 IP 的 4K 素材量，热门 IP（电锯人 43 张、海贼王 21 张）直接进，素材太薄的（比如某特工全家桶只有 1 张 4K）忍痛跳过。二次元壁纸站 4K 资源分布极不均匀，先查库存再开工能省一半无效功。

![动漫主题预览](https://raw.githubusercontent.com/jasper-nan/omarchy-anime-themes/main/docs/preview-grid.png)

二、壁纸不是找来的，是"造"出来的

直接铺 4K 壁纸有个致命问题：**顶栏和状态栏字看不清**。美术风格千奇百怪，浅色天空下白字直接消失。

我的解法是给每张壁纸加工上下两条渐变暗带（顶部 260px、底部 150px），颜色取自该主题配色。这样顶栏浮在暗带上，对比度永远达标。

但"看起来行了"不算数，我给每张壁纸写了一套**像素级验收**：

| 检查项 | 方法 | 标准 |
|---|---|---|
| 顶栏对比度 | 裁切顶 26px 求相对亮度，算 WCAG 对比度 | ≥ 4.5 (AA 级) |
| 底部暗带 | 裁切底 20px 求亮度 | < 0.08 (够暗) |
| 中缝异常 | 中部横条求标准差 | 自然图 > 0.2 |

第三条是被 bug 教出来的：某次 ImageMagick 的 `-gravity center` 残留到合成阶段，暗带全打在了画面中间，肉眼一看"横着一条杠"。从此每张壁纸都要采样中缝方差——均匀横杠 ≈ bug，自然纹理 ≈ 健康。

还有一个更阴的坑：**截断的 JPEG**。下载中断的文件 ImageMagick 只报 warning 不报错，尺寸校验照常通过，处理出来半张图是纯灰。最后靠中部像素采样揪出来——接近 `(128,128,128)` 纯灰即弃。看图工具骗你，像素不会。

![配色卡示例](/images/showcase/palette-frieren.jpg)

每个主题 = 一份 `colors.toml`（22 个语义色位）+ 图标主题 + 4 张左右加工过的壁纸 + 一张程序生成的抽象渐变备用图（没图也能有氛围）。

三、联动全家桶：一次切换，全屋换装

Omarchy 的主题系统有个精妙设计：`omarchy theme set` 之后，所有应用的配置由官方模板引擎从 `colors.toml` 统一渲染。我的配置里挂了这些：

- **四个终端**（foot/alacritty/kitty/ghostty）：foot 加了 85% 透明度 + Hyprland 原生背景模糊，终端浮在角色立绘上
- **AI 终端**（pi）：md 标题色、引用边条、列表符号映射到主题色——和终端一个妈生的
- **btop / Obsidian / Neovim / VSCode / Chromium**：官方模板全套覆盖
- **输入法 fcitx5**：候选框配色跟主题走

![星穹铁道配色卡](/images/showcase/palette-honkai-star-rail.jpg)

最远的一环是**开机画面和登录界面（Plymouth + SDDM）**：每个主题生成一枚角色名书法字 logo（Noto Serif CJK 渲染 + 主题色光晕），挂在 `theme-set` 钩子上，切主题时连开机画面一起换。下次重启，迎接你的不是千篇一律的厂商 logo，而是"雷电将军"四个鎏金大字。

![开机画面效果](/images/showcase/boot-preview.jpg)

提权这块有个取舍值得说：改开机画面需要 root，但我不想为图省事留永久的 NOPASSWD 后门。最终用官方的 `omarchy sudo passwordless [分钟]` 临时免密——切主题时静默同步，窗口外则发一条桌面通知提醒。安全性和顺滑度兼得。

四、每工作区一个世界

最新玩具：**工作区 1 是宵宫的花见坂，工作区 2 是芙莉莲的旅途，工作区 3 登上星穹列车**。

```lua
-- ~/.config/hypr/workspace-themes.lua (核心 15 行)
hl.on("workspace.active", function(ws)
  local id = tonumber(tostring(ws):match("HL%.Workspace%((%d+)"))
  local slug = map[id]          -- 从映射表查主题
  if not slug then return end
  hl.exec_cmd("omarchy theme set " .. slug)
end)
```

配置文件一行一个映射，Super+数字切工作区，整个世界随之切换。

这里翻出新坑：新版 Hyprland 的 IPC 是 **Lua 化的**。老的 `hyprctl dispatch workspace 2` 直接语法报错，正确写法是 `hyprctl dispatch 'hl.dsp.focus({ workspace = "2" })'`；事件也不再走 socket2 文本流，而是在 Lua 配置里 `hl.on("workspace.active", ...)` 注册回调。第一版 socat 守护进程被这些变化坑得七荤八素，最后 15 行原生 Lua 解决战斗——连 systemd 服务都不用。

* * *

五、开门迎客

两个主题仓库已开源，自带安装脚本：

- **[omarchy-genshin-themes](https://github.com/jasper-nan/omarchy-genshin-themes)** — 14 个原神角色
- **[omarchy-anime-themes](https://github.com/jasper-nan/omarchy-anime-themes)** — 11 部热门动漫

```bash
git clone https://github.com/jasper-nan/omarchy-anime-themes
cd omarchy-anime-themes && ./install.sh
```

写在最后：这套东西的本质是把"换主题"从一次性手工活变成了**带验收的流水线**——配色有语义位、壁纸有像素验收、成品有开源仓库、坑有笔记沉淀。折腾的乐趣在于过程，但留下来的是可复制的工程。

你的桌面现在是什么样？评论区晒晒，或者告诉我下一个想看的角色 👇
