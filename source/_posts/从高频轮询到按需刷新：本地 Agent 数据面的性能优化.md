---
title: 从高频轮询到按需刷新：本地 Agent 数据面的性能优化
date: 2026-07-13 12:30:00
description: Owlet 同时读取 Claude Code JSONL 与 Codex SQLite，并驱动菜单栏、灵动岛、灵动卡和仪表盘。本文复盘最近一轮性能优化：怎样合并高频刷新、拆分不同 UI surface 的 analytics，并避免无变化数据触发 SwiftUI 重绘。
cover: /images/从高频轮询到按需刷新：本地 Agent 数据面的性能优化.webp
categories:
  - [AI]
tags:
  - Agent
  - AI Infra
  - SwiftUI
  - macOS
  - Performance
  - Observability
---

我最近在做 Owlet：一个本地读取 Claude Code 与 Codex 会话记录、展示 Token 和实时活动的 macOS 工具。它的难点不在「能不能把日志读出来」，而在「数据源有任何变化时，怎样既足够快，又不把本地 app 刷成一个持续工作的轮询器」。

这轮改动之后，我更确定一件事：本地 Agent 工具的性能优化，重点不是把某一次 query 跑快，而是把**刷新语义、数据计算和 UI 发布**拆开管理。

## 一句话总结

把“任何事件都立即全量刷新”改成“合并请求、分层计算、变化才发布”。这样既能保住 Agent 正在运行时的及时性，也能显著减少重复 SQLite 查询和无意义的 SwiftUI invalidation。

## 问题不在单个刷新，而在刷新会叠加

Owlet 的数据链路大致是：

```text
Claude Code JSONL / Codex SQLite
  -> Adapter 增量同步
  -> 本地 analytics SQLite
  -> AppState
  -> 菜单栏 / 灵动岛 / 灵动卡 / 仪表盘
```

一开始最直觉的实现是：文件变化一次，触发一次 `refresh()`；刷新完成后，重新计算所有 analytics 并发布所有 `@Published` 状态。

它的问题会在真实使用里迅速放大：一次 Agent 工作往往连续写入多个文件；文件观察、定时活跃探测、用户手动刷新又可能同时到达。若每次都启动完整链路，就会出现 3 类浪费：

| 浪费 | 结果 |
| --- | --- |
| 同一时段的多次刷新互相重叠 | 重复读取来源、争用本地数据库 |
| 小组件只需要今日 Token，却计算了完整 dashboard | 菜单栏和灵动岛被大查询拖慢 |
| 查询结果没有变化，仍重新赋值 | SwiftUI 继续 diff、重绘所有依赖视图 |

这不是 CPU 单点热点，而是一个典型的控制面问题：系统没有定义好“下一次刷新是否仍有必要”。

## 第一层：合并请求，但不丢掉尾部更新

普通 debounce 的常见写法是：每次事件都取消旧 Task，重新等待几秒再刷新。它能降频，但有一个隐蔽问题：如果刷新正在进行时又来了事件，新事件很容易被 `isRefreshing` 直接丢掉。

Owlet 改成了单个长期持有的 auto-refresh Task，加一个 `pendingAutoRefresh` 标记：

```swift
private func scheduleAutoRefresh() {
    pendingAutoRefresh = true
    guard autoRefreshTask == nil else { return }

    autoRefreshTask = Task { @MainActor [weak self] in
        while !Task.isCancelled {
            try? await Task.sleep(for: .milliseconds(200))
            guard let self, !Task.isCancelled else { return }

            self.pendingAutoRefresh = false
            let didRefresh = await self.refresh(preloadAnalytics: .none)
            if !didRefresh { self.pendingAutoRefresh = true }

            guard self.pendingAutoRefresh == false else { continue }
            self.autoRefreshTask = nil
            return
        }
    }
}
```

这里的关键不是 `200 ms` 这个数字，而是状态机：

1. 任意事件只负责标记“有新工作”。
2. 已经有 refresh worker 时，不再创建第二个 worker。
3. worker 刷新后再次检查标记；期间来的新事件会自然形成下一轮。

这相当于一个 **trailing-edge coalescer**。它允许短时间内的事件合并，又保证「刷新期间发生的最后一次变化」不会消失。对本地日志同步来说，这比无限取消的 debounce 更符合数据正确性。

## 第二层：不同 surface，不要共用一份重型刷新

菜单栏、灵动岛和灵动卡真正关心的是“现在有没有活跃 Agent、今天用了多少 Token”。仪表盘才需要项目分布、模型分布、小时趋势、最近会话等完整数据。

此前这些 UI surface 共用一份 analytics reload，打开菜单栏也可能触发全套 dashboard 查询。优化后把 snapshot 分成两类：

```swift
enum AnalyticsReloadScope {
    case none, surface, board, all
}
```

| Scope | 计算内容 | 消费者 |
| --- | --- | --- |
| `surface` | 今日 Token、近 7 日日聚合 | 菜单栏、灵动岛、灵动卡 |
| `board` | 会话、项目、模型、日/小时趋势 | 灵动台仪表盘、会话页 |
| `all` | 两者 | 首次启动、用户显式刷新 |

当用户点开菜单栏时，先从已经落到本地 SQLite 的数据快速加载 `surface` snapshot，再在后台走 adapter 增量同步。用户看到的是立刻可用的旧但可信数据；同步完成后再应用新结果。

这个策略背后的取舍是：**本地可观测性 UI 优先 freshness 的连续性，不必执着于每次打开都同步到最新字节。** 只要后续同步能很快收敛，体验比“打开后空白等待一次全量链路”好得多。

## 第三层：查询完成不等于应该发布状态

SwiftUI 的刷新成本不只来自 view body 本身。一个 `@Published` 数组被重新赋值，会使依赖它的图表、列表、卡片都重新进入 diff。

所以 snapshot 要实现 `Equatable`，并在主线程发布前做一次值级短路：

```swift
private func applyBoardAnalyticsSnapshot(_ snapshot: BoardAnalyticsSnapshot) {
    guard cachedBoardAnalyticsSnapshot != snapshot else { return }
    cachedBoardAnalyticsSnapshot = snapshot

    if totals != snapshot.totals { totals = snapshot.totals }
    if recentSessions != snapshot.recentSessions { recentSessions = snapshot.recentSessions }
    if hourlyUsage != snapshot.hourlyUsage { hourlyUsage = snapshot.hourlyUsage }
    // 其余字段同理
}
```

这一步很朴素，但它把“数据库读到一个结果”与“UI 需要看到一次新状态”明确区分开了。特别是 3 秒一次的活动探测中，绝大多数轮次的活跃会话并没有变化；这时只要不写回 `activeSession`，就不会无端唤醒整棵视图树。

## 再补一层：活跃时快，空闲时慢

Agent 正在工作时，用户在意灵动岛是否实时；空闲时，每秒检查一次只是在浪费电和 I/O。

因此 periodic refresh 采用两档节奏：

```text
检测到活跃 Agent：每 3 秒检查并按需同步
没有活跃 Agent：30 秒才做一次兜底刷新
文件观察事件：立即合并到下一次刷新
```

文件观察负责绝大多数实时性，低频轮询只负责兜底。这个组合比“纯轮询”可靠，也比“完全相信文件事件”更能应对日志轮转、外部工具写入和观察器偶发失效。

## 我会怎样验证这类优化

这里有一个容易犯的错：没有 trace 就先宣布“快了 X%”。这次我没有编造一个漂亮数字，而是把验证拆成三个层次：

1. **行为正确性**：并发事件不会丢最后一次更新；手动刷新与自动刷新不会并发重入。
2. **数据正确性**：没有 delta 时不改 analytics；有 delta 时会重新读取对应 scope。
3. **运行时证据**：用 Instruments 观察主线程占用、SwiftUI update 原因，以及菜单栏打开和仪表盘切换时的 SQLite 查询量。

性能优化应该以“减少了哪一类工作”为单位陈述，而不只是看某一次 demo 的体感。

## 要不要用 / 我的判断框架

如果你在做的是会持续观察本地日志、数据库、文件系统或多个 Agent runtime 的桌面工具，我建议尽早建立这三条边界：

- **值得上**：数据源有 burst 写入，且多个 UI surface 订阅同一份状态。优先做刷新合并和 scope 拆分。
- **可以再等**：只有一个页面、一次刷新只读极少量数据。先保持简单，不要过早拆 Task。
- **重点关注**：每次把新数组重新塞进 `@Published` 的地方。它往往不是数据变慢的根因，却是界面无故重绘的最后一跳。

对我来说，这也是从前端转向 Agent / AI Infra 时很有意思的一点：真正的性能不是“把某个函数写得更快”，而是替整个系统定义一套更少做无效工作的调度规则。
