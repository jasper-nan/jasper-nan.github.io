---
title: "Hermes Skill 创建指南与 CalDAV 日历同步实战：从个人到家庭的 AI 助手配置"
description: "详解 Hermes/OpenClaw 技能系统的创建方法，以及如何通过 CalDAV 实现家庭日历同步，让 AI 助手真正成为全家人的效率工具。"
date: 2026-06-09
tags:
  - Hermes
  - OpenClaw
  - CalDAV
  - AI 助手
  - 家庭自动化
  - 技能开发
---

> 最近给老婆配置 CalDAV 日历同步技能时，发现很多人对 Hermes（原 OpenClaw）的 Skill 系统还不够熟悉。这篇文章从 Skill 创建原理讲起，到 CalDAV 日历同步的实战配置，再到如何给家庭成员快速部署，希望能帮你把 AI 助手从"个人玩具"变成"家庭基础设施"。

* * *

## 一、Hermes Skill 是什么？为什么重要？

### 1.1 从插件到技能：AI 助手的进化

传统的软件扩展叫"插件"（Plugin），强调的是**功能叠加**。而 Hermes 的 Skill 是**工作流封装**——它不只是加一个新功能，而是把"如何完成一件事"的完整逻辑打包交给 AI。

举个例子：
- **插件思维**：装一个日历插件，能看日程
- **Skill 思维**：说一句"帮我查下周三的会议"，AI 自动调用 CalDAV 查询、解析时间语义、返回格式化结果

Skill 的核心价值在于**降低使用门槛**。你不需要记住命令，用自然语言表达意图就行。

### 1.2 Skill 的底层结构

每个 Skill 本质上是一个目录，包含：

```
~/.hermes/skills/<category>/<skill-name>/
├── SKILL.md          # 技能定义文件（必需）
├── references/       # 参考文档
├── templates/        # 模板文件
├── scripts/          # 可执行脚本
└── assets/           # 图片等资源
```

最核心的就是 `SKILL.md`，它采用 YAML Frontmatter + Markdown 的格式：

```yaml
---
name: caldav-sync
description: Manage calendar events via CalDAV
version: 1.0.0
author: Your Name
---

# CalDAV Sync

## Configuration

Set these environment variables:

```bash
export CALDAV_URL="https://caldav.example.com:9999"
export CALDAV_USERNAME="your-username"
export CALDAV_PASSWORD="your-password"
```

## Usage

- Add event: `caldav_add --title "Meeting" --start "2024-01-01T10:00:00" ...`
- List events: `caldav_list [--days 7]`
```

Hermes 在启动时会扫描 `~/.hermes/skills/` 目录，自动加载所有有效的 Skill。

### 1.3 Skill 的三种类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **内置技能** | Hermes 官方提供，随安装附带 | web_search, terminal, file |
| **社区技能** | 通过 `hermes skills install` 安装 | caldav-sync, blogwatcher |
| **自定义技能** | 用户自己创建的 SKILL.md | 个人记账、家庭日历 |

* * *

## 二、手把手创建一个 Skill

### 2.1 最简单的 Skill：Hello World

```bash
# 创建目录
mkdir -p ~/.hermes/skills/custom/hello-world

# 编写 SKILL.md
cat > ~/.hermes/skills/custom/hello-world/SKILL.md << 'EOF'
---
name: hello-world
description: A simple hello world skill to demonstrate the structure
version: 1.0.0
---

# Hello World

This skill demonstrates the basic structure of a Hermes skill.

## Usage

Simply say "hello" and the agent will respond with a greeting.
EOF
```

重启 Hermes 或执行 `hermes skills reload`，这个技能就生效了。

### 2.2 带脚本的 Skill：系统监控

如果 Skill 需要执行代码，可以在 `scripts/` 目录放脚本：

```bash
mkdir -p ~/.hermes/skills/custom/system-monitor/scripts

cat > ~/.hermes/skills/custom/system-monitor/scripts/check-disk.sh << 'EOF'
#!/bin/bash
df -h / | awk 'NR==2 {print "磁盘使用率: "$5" ("$3"/"$2")"}'
EOF

chmod +x ~/.hermes/skills/custom/system-monitor/scripts/check-disk.sh
```

然后在 `SKILL.md` 中引用：

```markdown
## Tools

- `check_disk`: Check disk usage
  - Script: `scripts/check-disk.sh`
  - Usage: Run directly to get disk usage info
```

### 2.3 带配置的 Skill：CalDAV 日历同步

更复杂的 Skill 需要环境变量配置。以 CalDAV 为例：

```bash
mkdir -p ~/.hermes/skills/openclaw-imports/caldav-sync
```

`SKILL.md` 内容：

```yaml
---
name: caldav-sync
description: Manage calendar events via CalDAV — add, list, query calendar events
version: 1.0.0
author: OpenClaw
---

# CalDAV Sync

Connect to CalDAV server to manage calendar events.

## Configuration

Set these environment variables in `~/.hermes/.env` or your shell profile:

```bash
export CALDAV_URL="https://caldav.laji.fun:9999"
export CALDAV_USERNAME="your-username"
export CALDAV_PASSWORD="your-password"
```

## Tools

- `caldav_add`: Add a new event
  - Usage: `caldav_add --title "Meeting" --start "2024-01-01T10:00:00" --end "2024-01-01T11:00:00" [--description "Notes"]`
- `caldav_list`: List upcoming events
  - Usage: `caldav_list [--days 7]`
- `caldav_query`: Query events in date range
  - Usage: `caldav_query --from "2024-01-01" --to "2024-01-31"`

## Examples

- Add event: "帮我添加明天下午3点到4点的会议"
- List events: "查看最近一周的日程"
- Query range: "查询6月1日到6月15日的所有事件"
```

* * *

## 三、CalDAV 日历同步原理详解

### 3.1 什么是 CalDAV？

CalDAV（Calendar Distributed Authoring and Versioning）是基于 WebDAV 的日历访问协议，让你可以通过网络读写日历数据。

**核心特点**：
- **标准化**：RFC 4791 定义，Apple、Google、Nextcloud 等都支持
- **双向同步**：本地修改同步到服务器，服务器变更同步到本地
- **多客户端**：iPhone、Android、Mac、Web 都能访问同一日历

### 3.2 CalDAV 的工作流程

```
┌─────────────┐     HTTP/HTTPS      ┌─────────────┐
│   Hermes    │ ◄─────────────────► │ CalDAV      │
│   (Client)  │   PROPFIND/REPORT   │ Server      │
│             │ ◄─────────────────► │ (Baikal/    │
│             │   PUT/DELETE        │  Radicale/  │
│             │                     │  Nextcloud) │
└─────────────┘                     └─────────────┘
```

常用操作对应的 HTTP 方法：

| 操作 | HTTP 方法 | 说明 |
|------|-----------|------|
| 发现日历 | `PROPFIND` | 查询用户有哪些日历 |
| 读取事件 | `REPORT` | 按时间范围查询事件 |
| 添加事件 | `PUT` | 上传 .ics 文件 |
| 删除事件 | `DELETE` | 删除指定事件 |
| 修改事件 | `PUT` | 覆盖更新 .ics 文件 |

### 3.3 我们的 CalDAV 服务器

我家用的是自托管方案：

- **服务器**：`https://caldav.laji.fun:9999`
- **软件**：Baikal（轻量级 CalDAV/CardDAV 服务器）
- **部署**：Docker + Nginx 反向代理
- **用户**：每人一个独立账户

**为什么选择自托管？**
1. **隐私**：家庭日程不上第三方云
2. **成本**：一次性配置，无订阅费
3. **灵活**：想加用户就加用户

* * *

## 四、家庭部署实战：给我老婆配置日历技能

### 4.1 场景需求

- 我老婆也需要通过 Hermes 管理日程
- 她的 CalDAV 账户是独立的（`zhaoyilei`）
- 需要快速部署，不能让她折腾配置

### 4.2 一键安装脚本

我给她准备了一段命令，复制粘贴就能用：

```bash
# 创建技能目录
mkdir -p ~/.hermes/skills/openclaw-imports/caldav-sync

# 写入技能文件
cat > ~/.hermes/skills/openclaw-imports/caldav-sync/SKILL.md << 'EOF'
---
name: caldav-sync
description: Manage calendar events via CalDAV
version: 1.0.0
---

# CalDAV Sync

## Configuration

```bash
export CALDAV_URL="https://caldav.laji.fun:9999"
export CALDAV_USERNAME="zhaoyilei"
export CALDAV_PASSWORD="your-password"
```

## Usage

- Add event: `caldav_add --title "Meeting" --start "2024-01-01T10:00:00" ...`
- List events: `caldav_list [--days 7]`
EOF

# 重启 Hermes 生效
hermes skills reload
```

### 4.3 环境变量配置建议

不要把密码直接写在 SKILL.md 里！推荐做法：

**方法 1：`.env` 文件**

```bash
# ~/.hermes/.env
CALDAV_URL=https://caldav.laji.fun:9999
CALDAV_USERNAME=zhaoyilei
CALDAV_PASSWORD=your-password
```

**方法 2：Shell 配置文件**

```bash
# ~/.zshrc 或 ~/.bashrc
export CALDAV_URL="https://caldav.laji.fun:9999"
export CALDAV_USERNAME="zhaoyilei"
export CALDAV_PASSWORD="your-password"
```

### 4.4 使用示例

配置完成后，自然语言就能操作日历：

| 你说 | AI 执行 |
|------|---------|
| "帮我添加明天下午3点的会议" | 解析时间 → 调用 `caldav_add` → 返回确认 |
| "查看最近一周的日程" | 调用 `caldav_list --days 7` → 格式化输出 |
| "下周三有空吗？" | 查询该日事件 → 判断空闲时段 → 回复 |
| "把明天的会议改到周五" | 查询原事件 → 删除 → 新建 → 确认 |

* * *

## 五、进阶：Skill 版本管理与共享

### 5.1 版本控制

建议把自定义 Skill 用 Git 管理：

```bash
cd ~/.hermes/skills/custom
git init
git add .
git commit -m "init custom skills"
```

### 5.2 家庭共享方案

我们家的 Skill 共享流程：

1. **我创建/更新 Skill** → 推送到私有 Git 仓库
2. **老婆执行更新命令**：
   ```bash
   cd ~/.hermes/skills/openclaw-imports/caldav-sync
   git pull
   hermes skills reload
   ```

3. **或者更简单的**：直接发她一段命令，复制粘贴即可

### 5.3 技能市场安装

如果 Skill 发布到社区：

```bash
# 官方市场
hermes skills install caldav-sync

# 从 URL 安装
hermes skills install https://example.com/skills/caldav-sync.md

# 从 Git 仓库安装
hermes skills install git+https://github.com/user/hermes-caldav-skill.git
```

* * *

## 六、常见问题排查

### Q1: Skill 安装后没有生效？

检查清单：
- [ ] 目录结构是否正确？`~/.hermes/skills/<cat>/<name>/SKILL.md`
- [ ] YAML Frontmatter 是否完整？必须有 `name` 和 `description`
- [ ] 是否执行了 `hermes skills reload`？
- [ ] Hermes 日志是否有加载错误？

### Q2: CalDAV 连接失败？

```bash
# 测试服务器连通性
curl -I https://caldav.laji.fun:9999

# 测试认证
curl -u username:password \
  -X PROPFIND \
  https://caldav.laji.fun:9999/calendars/username/
```

常见问题：
- **证书错误**：自签名证书需要信任
- **404 错误**：URL 路径不对，检查 discovery 路径
- **401 错误**：用户名或密码错误

### Q3: 如何查看 Skill 加载日志？

```bash
# 查看 Hermes 日志
hermes logs

# 或启动时加调试参数
hermes --debug
```

* * *

## 七、总结

Hermes 的 Skill 系统让 AI 助手从"聊天机器人"变成了"工作流引擎"。通过简单的 Markdown 文件，你就能：

1. **封装重复操作** —— 把常用命令变成自然语言指令
2. **连接外部服务** —— CalDAV、Notion、GitHub 等
3. **家庭共享配置** —— 一人创建，全家受益

CalDAV 日历同步只是开始。你可以用同样的思路创建：
- 家庭记账 Skill（连接你的记账系统）
- 智能家居 Skill（控制 Home Assistant）
- 健康管理 Skill（同步运动数据、提醒吃药）

**AI 助手的终极形态，不是它有多聪明，而是它有多懂你的生活。**

---

*本文同步发布于 [jasper-nan.github.io](https://jasper-nan.github.io/)*

*如果你也想搭建家庭 CalDAV 服务器，可以参考我的另一篇文章《群晖 NAS 新手入门完全指南》中的 Docker 部署章节。*