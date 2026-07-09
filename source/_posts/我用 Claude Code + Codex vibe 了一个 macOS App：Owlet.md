---
title: 我用 Claude Code + Codex vibe 了一个 macOS App：Owlet
date: 2026-07-10 02:30:00
description: 这篇文章记录我用 Claude Code 和 Codex 以 vibe coding 方式做一个 macOS menu-bar 应用的全过程：从最初想“看看我每天烧了多少 token”，到做出 Owlet 这个能实时监控 Claude Code 和 Codex 使用量的小工具。过程中有 AI 双开协作的节奏、把家乡青山湖风景写进配色的执念，也有对 menu-bar app 电量的偏执。
categories:
  - [AI]
tags:
  - Claude Code
  - Codex
  - vibe coding
  - macOS
  - Swift
  - Agent
  - Owlet
cover: /images/owlet-vibe-coding-cover.webp
---

过去一个月，我的 Mac 菜单栏里多了一只小猫头鹰。

它叫 Owlet，是我用 Claude Code 和 Codex 一起 vibe 出来的 macOS 小工具：常驻菜单栏，实时显示 Claude Code 和 Codex 的 token 消耗，附带一个贴合刘海的灵动岛面板、一个桌面灵动卡、还有一个看历史用量和排名的 dashboard。

做这个 app 的动机很私人。今年我开始从前端往 Agent / AI Infra 方向转，每天大部分时间都在跟 Claude Code 和 Codex 泡在一起。泡着泡着就产生了一个很朴素的疑问：**我每天到底烧了多少 token？钱花哪儿了？哪个项目最费钱？** 现有的工具都不直接回答这个问题，那我就自己做一个。

于是就有了 Owlet。

![Owlet 主视觉](/images/owlet-vibe-coding-cover.webp)

## 一句话总结

**Claude Code 负责架构、长上下文和需要判断力的重构；Codex 负责快速试错和小范围实现；我负责拍板、验收和把家乡青山湖的颜色塞进 UI。** 三者分工明确时，vibe coding 真的能把一个 idea 从 0 推到能用。

## 为什么是一只猫头鹰

Owlet 的英文意思是“小猫头鹰”。图标是一只像素风猫头鹰，缩在菜单栏里，眼睛一眨一眨地观察你的 AI 会话。

选它有两个原因。一是气质贴合：这个 app 做的就是“盯着你用 AI 的过程”，猫头鹰那种夜间警觉、悄无声息的感觉，比什么“机器人”“仪表盘”都更对味。二是可爱——macOS 菜单栏天天见的东西，不能太硬核，得让人愿意让它一直待着。

定下名字那晚我对 Claude 说：

> “根据现在帧动画里猫头鹰 icon 的形象重新命名 app。”

最后从几个候选里挑了 Owlet。短小、好读、不容易撞名，而且一听就知道跟 owl 有关。

这就是独立开发的好处：审美和命名这种在别人公司要开三天会的决定，自己一个人一晚上就能拍板。

## Claude Code 是架构师，Codex 是快枪手

![Claude Code + Codex 双开工作台](/images/owlet-vibe-coding-workbench.webp)

这个项目里我同时开了两条 AI 线：Claude Code 和 Codex。

**Claude Code 是我的主架构师。** 它的长上下文能力太适合这种代码量不大但跨模块关联多的项目。比如设计实时 session 监控链路时，我需要同时理解：

- Claude Code 的 hook 机制（`owlet-hook.py` 怎么装、怎么往 Unix socket 发事件）
- macOS 上 `NSPanel` 和 SwiftUI 的边界行为
- 事件驱动的文件监听该怎么 debounce
- SQLite 里 session、hourly、daily 三张表怎么 upsert 不丢数据

这种时候我会把问题抛给 Claude Code，让它先给方案，我再挑毛病。

**Codex 则是我的快枪手。** 有些改动范围很小、很具体，不需要长篇讨论，直接让 Codex 上。比如：

> “Codex 判断了有没有刘海的逻辑，删了。”
>
> “这个黑框是为啥，这也是 Codex 今天下午加的。”

Codex 的特点是快，但偶尔会留下一些“看起来能动，但边界没处理干净”的痕迹。那个黑框就是典型：它在刘海屏设备上加了一层黑色背景，意图是隐藏刘海，结果在非刘海屏上反而露了馅。最后还是我拉着 Claude Code 一起复盘，把透明 panel 的圆角和 masksToBounds 重新理了一遍才解决。

这件事让我明确了一条分工：**需要理解系统边界和做取舍的，找 Claude Code；需要快速验证一个想法的，找 Codex。**

## 一个 idea 长出四块屏幕

![Owlet 的灵动家族：灵动岛、灵动卡、灵动栏、灵动板](/images/owlet-lingdong-family.webp)

Owlet 的 UI 表面不算多，但每一块都有自己的坑：

| Surface | 作用 | 难点 |
|---|---|---|
| Menu bar | 常驻图标 + token 总数 | 图标动画不能太耗电 |
| Floating island | 刘海区实时会话面板 | `NSPanel` 圆角、多屏、透明度 |
| Dashboard | 历史用量、排名、设置 | SwiftUI 图表和数据库查询 |
| Dynamic card | 桌面小组件式面板 | 像系统 widget，但不能真用 WidgetKit |

最有意思的是命名。某天我突然说：

> “刷新数据放到标题旁边怎么样？还有在灵动家族给这个菜单起个名字，一家人要整整齐齐。”

于是就有了“灵动家族”：灵动岛、灵动卡、灵动栏。这个命名一点都不高大上，但它让产品在聊天时变得很好描述。**“灵动卡又崩了”** 比 **“桌面浮动窗口崩溃了”** 可爱多了。

另一个让我印象深刻的是灵动卡的崩溃排查。同样是 Apple Silicon、同样是 macOS 15，有的机器能打开，有的机器直接崩。最后定位到问题不在业务逻辑，而在 AppKit window level、`NSPanel`、系统材质和透明圆角叠加后的边界行为。那篇文章我已经单独发过，这里不再展开。

## 把青山湖写进代码

![Owlet 的青山湖配色体系：蓝雾、湖青、杉绿、雾白、晨光浅金](/images/owlet-qingshan-lake-palette.webp)

我是杭州临安人，青山湖就在家门口。Owlet 的视觉方向最后定成了：**像原生 macOS 工具，但从青山湖的湖水、杉林、青山、雾白里取色。**

具体 Palette 长这样：

- 蓝雾天空 → 浅背景
- 浮萍湖绿 → 主要强调色
- 水杉深绿 → 深色文字和进度条
- 雾带白 → 面板底色
- 晨光浅金 → 仅用于边缘光和状态提示
- 雨后湿木 → 少量阴影和分隔

我甚至专门给 Claude 下了一条规则：

> “晨光浅金只用于边缘光、状态提示或小面积 accent，不能主导整个 surface。”

因为项目早期用过一段时间黄/金色做主色，结果越调越脏，最后全推翻了。改成青山湖色系后，整个 app 立刻从“网吧风”回到了“能放在 macOS 桌面上不丢人”的状态。

## 性能偏执：一个 menu bar app 的电量焦虑

Owlet 是常驻应用。只要 Claude 不活跃，它就不应该醒来。

为此我们做了几件看起来很细节、但很重要的事：

1. **文件监听用 Dispatch Source，而不是轮询。** 之前有些实现会 sleep-loop 检查 active session，我全部砍掉了。
2. **JSONL 增量读取。** 50 MB 的会话日志不应该造成 50 MB 内存峰值。ClaudeCodeAdapter 用 `(mtime, fileSize, byteOffset)` 做缓存，文件增长时只读新增 bytes。
3. **动画门控。** 菜单栏猫头鹰只有 token 总数变化时才脉冲；dashboard 和 island 只有 active session 存在时才按 0.2s 帧率动画，idle 时最多每秒一帧。
4. **数据库 upsert 而不是 deleteAll + insert。** 这个改动很小，但能避免大量无用写入。

这些规则都被写进了 `CLAUDE.md` 和 `AGENTS.md`，成为后续所有改动的底线。**一个工具要是自己先变成电老虎，就没资格监控别人的 token 消耗。**

## AI 协作的边界：它不是替你做主

整个过程中，AI 没有替我做任何一个真正重要的决定。

- 要不要做这个项目？我决定。
- 功能优先级怎么排？我决定。
- 命名 Owlet？我决定。
- 青山湖配色方向？我决定。
- 性能规则里哪些不能回退？我决定。

AI 做的是：**把我模糊的判断快速翻译成可执行的代码和文档，并在执行过程中帮我规避低级错误。**

这让我越来越确信一个判断框架：在 Agent / AI Infra 时代，人的核心竞争力不是写代码的速度，而是**判断框架**——知道什么该做、什么不该做、什么要取舍、什么是底线。AI 负责把判断拉满执行力，两者结合才是极致交付速度。

## 要不要用 / 我的判断框架

如果你也想尝试 vibe coding 一个小工具，我的建议是：

| 场景 | 判断 |
|---|---|
| 想做一个小而美的个人工具 | **值得上** |
| 项目涉及大量系统边界（macOS 权限、沙盒、codesign） | **重点关注** |
| 需要长期维护和多端同步 | **可以再等** |
| 你享受“说一句话，AI 帮你验证”的快感 | **值得上** |
| 你无法接受 AI 偶尔留下小坑 | **可以再等** |

对我来说，Owlet 是一个完美的实验场：规模可控、用户就是自己、技术栈有挑战性但不需要服务百万用户。如果后续它能帮更多开发者看清 AI 使用成本，那就更好了。

现在菜单栏里那只小猫头鹰还在闪。它看着自己诞生的过程，也看着我每天烧掉的 token。

挺 meta 的，也挺酷的。
