---
title: 小说站 Facebook Pixel 深度配置：用 scroll depth 事件训练算法找「真实读者」
date: 2026-07-04 22:00:00
description: 在小说站 Facebook 投流中，PageView 是最糟糕的优化信号——它把「看了 3 秒就走的人」和「读完整章的人」混在一起。这篇文章记录我如何用 ChapterPixel 组件实装 6 层 scroll depth 事件，让 Facebook 算法精准区分不同深度的读者，最终降低 CPM、提升会话 RPM。
categories:
  - [AI]
tags:
  - Facebook Pixel
  - Next.js
  - AdSense
  - 广告套利
  - 小说站
  - scroll tracking
---

做小说站投流，最容易踩的坑是：用 `PageView` 作为广告优化事件。

`PageView` 的语义是「有人来了」，而不是「有人读了」。Facebook 算法拿这个信号去优化，会找到「擅长点广告」的人群，而不是「愿意读小说」的人群。结果就是 CTR 看着不错，但用户进来翻一页就走，AdSense 会话 RPM 极低，ROAS 算下来根本不打平。

## 一句话总结

在章节页实装 scroll depth 事件（ViewContent / ChapterRead50 / ChapterCompleted），让 Facebook 算法学会找「真正读完一章」的人，而不是「会点广告」的人。

## 为什么 PageView 是最差的优化信号

小说站的盈利公式：

```
Profit/visit = (会话 RPM × 翻页数/session) / 1000 − Facebook CPC
```

CPC 固定的情况下，唯一能拉高 ROAS 的是「翻页数/session」。而翻页数取决于用户是不是真的在读——这正是 `PageView` 衡量不到的。

Facebook 的优化逻辑是：给我 50 个「转化事件」的样本，我帮你找更多像这批人的用户。用 `PageView` 做样本，找到的是「会点广告的人」；用 `ChapterCompleted`（读完整章）做样本，找到的是「深度读者」。

## Pixel 事件层级设计

我给章节页设计了 6 层事件，覆盖从「进来了」到「读完了」的完整漏斗：

| 事件 | 触发 | AEM 优先级 | 用途 |
|---|---|---|---|
| `PageView` | 全局自动 | 4（最低）| 数据量打底，冷启动用 |
| `ViewContent` | 进入章节即触发 | 3 | 广告优化主信号，积累 500+ 后切优化目标 |
| `ScrollDepth25` | scroll ≥ 25% | 漏斗分析 | 判断 ch1 开头 hook 是否抓住人 |
| `ChapterRead50` | scroll ≥ 50% | 2 | 强意图信号，AEM 次级优先 |
| `ScrollDepth75` | scroll ≥ 75% | 漏斗分析 | 章节中后段内容质量诊断 |
| `ChapterCompleted` | scroll ≥ **90%** | 1（最高）| 最强信号，Lookalike seed |

> **为什么用 90% 而不是 100%？** 章节页底部有广告位、推荐书目、Share 栏。读完正文后用户视线就停了，不会继续滚到绝对底部。90% 才是实际的「读完」代理指标，100% 在现实中几乎永远不触发。

## 实现：ChapterPixel 组件

一个轻量的 client component，挂在 chapter page 里，用单个 passive scroll listener + `useRef` 防重复触发：

```tsx
'use client'
import { useEffect, useRef } from 'react'

interface Props {
  chapterTitle: string
  chapterOrder: number
  bookSlug: string
}

export default function ChapterPixel({ chapterTitle, chapterOrder, bookSlug }: Props) {
  const fired25 = useRef(false)
  const fired50 = useRef(false)
  const fired75 = useRef(false)
  const fired90 = useRef(false)

  useEffect(() => {
    const fbq = (window as Window & { fbq?: (...args: unknown[]) => void }).fbq
    // ViewContent 进入即触发 — 区分「进来了」和「立刻离开」
    fbq?.('track', 'ViewContent', {
      content_type: 'chapter',
      content_name: chapterTitle,
      content_ids: [`${bookSlug}-ch${chapterOrder}`],
    })

    const payload = { content_name: chapterTitle, chapter_order: chapterOrder }

    const onScroll = () => {
      const total = document.body.scrollHeight - window.innerHeight
      if (total <= 0) return
      const ratio = window.scrollY / total
      const fire = (window as Window & { fbq?: (...args: unknown[]) => void }).fbq

      if (!fired25.current && ratio >= 0.25) {
        fired25.current = true
        fire?.('trackCustom', 'ScrollDepth25', payload)
      }
      if (!fired50.current && ratio >= 0.5) {
        fired50.current = true
        fire?.('trackCustom', 'ChapterRead50', payload)
      }
      if (!fired75.current && ratio >= 0.75) {
        fired75.current = true
        fire?.('trackCustom', 'ScrollDepth75', payload)
      }
      if (!fired90.current && ratio >= 0.9) {
        fired90.current = true
        fire?.('trackCustom', 'ChapterCompleted', payload)
      }
    }

    window.addEventListener('scroll', onScroll, { passive: true })
    return () => window.removeEventListener('scroll', onScroll)
  }, [chapterTitle, chapterOrder, bookSlug])

  return null
}
```

chapter page 里直接挂上去：

```tsx
// app/book/[slug]/chapter/[n]/page.tsx
<ChapterPixel
  chapterTitle={chapter.title}
  chapterOrder={chapter.order}
  bookSlug={slug}
/>
```

几个设计细节值得注意：

- **`useRef` 防重复**：scroll 事件高频触发，必须用 ref 确保每个深度只上报一次
- **`passive: true`**：不阻塞渲染线程，对 Core Web Vitals 无影响
- **`fbq` optional chaining**：Pixel 加载有延迟，防止脚本未就绪时报错
- **ViewContent 在 mount 而非 scroll**：「进入章节」本身就是有效的意图信号，不需要等到用户滚动

## FB 后台还需要手动配置

代码侧搞定后，还有两步后台操作才能真正用上这些信号：

**1. 创建自定义转化**（ChapterRead50 / ChapterCompleted）

Events Manager → 自定义转化 → 新建，选择「自定义事件」→ 选对应事件名。`PageView` 和 `ViewContent` 是 Facebook 标准事件，不需要这步。

**2. 配置 AEM（聚合事件衡量）**

iOS 14+ 之后大部分 iOS 用户拒绝追踪，Facebook 的解法是「申报最多 8 个事件 + 优先级」，信号丢失时归因到优先级最高的可用事件。

路径：Events Manager → 你的 Pixel → 聚合事件衡量 → 配置网络事件

推荐优先级：
```
ChapterCompleted (1) > ChapterRead50 (2) > ViewContent (3) > PageView (4)
```

`ScrollDepth25` 和 `ScrollDepth75` 不加进 AEM——它们是漏斗分析用的，不是广告优化目标。AEM 只配置你真正用于 Campaign 优化的事件。

## 要不要做 / 我的判断框架

**值得立刻做：** 任何在跑 Facebook 流量的小说站都应该实装。几十行代码，对投流效果的影响是系统性的——算法找的人不一样，后续所有优化都站在更好的基础上。

**可以再等：** CAPI（Conversions API，服务端回传）。iOS 信号丢失严重时 CAPI 能补回 30%+ 的归因，但实现复杂（静态导出站需要额外部署 serverless function）。当前阶段流量体量不大，先把浏览器 Pixel 跑稳，等 ViewContent 积累到 500+ 再评估是否上 CAPI。

**重点关注：** ViewContent 积累到 500 次后，把 Campaign 优化目标从「落地页点击」切换为「ViewContent」，这是整个链路里 CPM 下降最明显的节点。
