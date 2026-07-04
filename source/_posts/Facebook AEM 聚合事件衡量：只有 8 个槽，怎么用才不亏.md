---
title: Facebook AEM 聚合事件衡量：只有 8 个槽，怎么用才不亏
date: 2026-07-04 22:30:00
description: iOS 14+ 之后，Facebook 广告归因出现了一个新的关键配置：AEM（聚合事件衡量）。每个域名只有 8 个事件槽，优先级顺序决定 iOS 用户的归因质量。这篇文章讲清楚 AEM 是什么、为什么存在、8 个槽怎么分配，以及常见的三个配置错误。
categories:
  - [AI]
tags:
  - Facebook Pixel
  - Facebook Ads
  - AEM
  - iOS 14
  - 独立站
  - 投流
cover: /images/facebook-aem-8-slots.webp
---

2021 年，苹果推出 ATT（App Tracking Transparency）框架，要求所有 iOS App 在追踪用户前必须弹窗获取授权。大多数用户选择拒绝。这一举措直接打掉了 Facebook 广告的 iOS 端归因能力——Facebook 无法再知道"这个 iOS 用户点了广告之后，在网站上做了什么"。

Facebook 的应对方案叫 AEM：**Aggregated Event Measurement（聚合事件衡量）**。

这是一个绕过单用户追踪、改用聚合统计进行归因的机制。理解它很重要，因为不配置 AEM，你的 iOS 广告数据几乎是盲的。

## 一句话总结

AEM 让你预先申报最多 8 个事件、排好优先级，Facebook 用差分隐私的方式完成 iOS 端归因——核心限制是 8 个槽，优先级顺序决定归因质量，用错等于浪费归因机会。

## AEM 是怎么工作的

不用 AEM 的情况（iOS 14+ 之前）：
1. iOS 用户点击广告
2. Facebook Pixel 追踪用户在网站上的所有行为
3. 归因完整，Facebook 知道这个用户做了什么

用 AEM 的情况（iOS 14+ 之后）：
1. iOS 用户点击广告
2. 用户在网站上触发事件（比如 ViewContent、ChapterCompleted）
3. Facebook 查你预先申报的 8 个事件清单
4. 在用户触发的事件里，找优先级最高的那个
5. 把这次转化归因给那个事件（用差分隐私聚合，不识别具体用户）

关键点：**当一个 iOS 用户在一次会话里触发了多个事件，Facebook 只归因优先级最高的那个。**

比如用户进了章节页，触发了 ViewContent（第 3 位）和 ChapterRead50（第 2 位）。Facebook 把这次转化归因给 ChapterRead50，而不是 ViewContent。

这就是为什么优先级顺序很重要：顺序错了，归因结果就歪了，你看到的数据和实际情况不符。

## 8 个槽怎么分配

先说原则：

**原则 1：价值越高的事件，优先级越高**

什么叫"价值高"？代表用户质量更好的事件。对小说站来说：
- 读完整章（ChapterCompleted）> 读了一半（ChapterRead50）> 打开章节（ViewContent）> 任意访问（PageView）

**原则 2：参与度事件不进 AEM**

ScrollDepth25、ScrollDepth75 不是转化信号，是过程信号。放进 AEM 会：
- 浪费宝贵槽位
- 如果优先级设置不当，会干扰高价值事件的归因

**原则 3：自定义事件必须先建「自定义转化」**

PageView、ViewContent 是 Facebook 标准事件，可以直接加进 AEM。但 ChapterRead50、ChapterCompleted 是我们自定义的事件名，需要先在 Events Manager 里建「自定义转化」，才能加进 AEM 优先级队列。

---

对小说站的推荐配置：

| 优先级 | 事件 | 类型 | 含义 |
|--------|------|------|------|
| 1 | ChapterCompleted | 自定义 | 读完整章，最高价值用户 |
| 2 | ChapterRead50 | 自定义 | 读到章节 50%，高质量读者 |
| 3 | ViewContent | 标准 | 打开章节开始读 |
| 4 | PageView | 标准 | 任意页面访问 |
| 5–8 | 留空 | — | 后期扩展（订阅、注册等） |

## 怎么配置（操作路径）

```
Meta Business Manager
→ Events Manager
→ 点击你的 Pixel
→ 聚合事件衡量
→ 配置网络事件
→ 添加你的域名（需要完成域名验证）
→ 添加事件，拖拽排顺序
```

**前置条件：**
1. 域名验证：在 Business Manager 里完成域名所有权验证（添加 DNS 记录或 meta 标签）
2. 自定义事件注册：ChapterRead50 / ChapterCompleted 先在「Events Manager → 自定义转化」里建好
3. Pixel 已安装并正常触发事件

## 三个常见错误

**错误 1：把 ScrollDepth 加进 AEM**

"用户滚动越深说明越感兴趣，放进 AEM 不是更精准吗？"

不对。AEM 的用途是**广告归因**，不是**用户行为分析**。你的目标是让 Facebook 知道"投的广告带来了多少高质量转化"，而不是"这些用户在页面上的所有行为"。ScrollDepth 的位置是 Events Manager 的受众分析维度，不是 AEM 的归因事件。

**错误 2：优先级顺序倒置**

把 PageView 排第一，ChapterCompleted 排第四——这是很常见的错误，通常因为"PageView 数量最多，感觉最稳定"。

结果是：iOS 用户进章节读完了整章，触发了 PageView + ViewContent + ChapterRead50 + ChapterCompleted，但归因给 PageView（优先级最高）。Facebook 以为你的广告目标是"有人来了网站"，而不是"有人读完了一章"。算法从此优化错方向。

**错误 3：自定义事件忘了先建转化**

直接在 AEM 里搜索 ChapterRead50，搜不到。原因是自定义事件需要先在「自定义转化」里定义（基于事件名过滤），才能在 AEM 里使用。

操作顺序：
1. Events Manager → 自定义转化 → 新建
2. 规则：事件名称等于 `ChapterRead50`
3. 保存后，回到 AEM 配置，就能找到这个转化了

## 为什么 AEM 不能完全替代 Pixel

AEM 解决的是 iOS 归因问题，但有几个固有限制：

- **数据延迟**：AEM 的归因数据有 24-72 小时延迟（差分隐私需要聚合足够的数据量才上报）
- **不能识别个体**：只有聚合数据，看不到"具体哪个用户触发了什么"
- **Android 不受影响**：Android 用户的 Pixel 追踪依然正常，AEM 主要解决 iOS 缺口

所以实际监控时：Android 数据看 Pixel 实时，iOS 数据看 AEM 聚合（接受延迟）。两套数据加总才是完整的归因图像。

## 我的判断框架

遇到"要不要把这个事件加进 AEM"，用一个问题判断：

**如果 Facebook 以这个事件为信号去做归因，归因结果能反映真实的广告价值吗？**

- ChapterCompleted：能。完成整章 = 广告带来了真读者 ✓
- PageView：勉强。任意访问，质量参差 △
- ScrollDepth75：不能。过程信号，不代表转化价值 ✗

通过的进 AEM，不通过的留在 Pixel 做受众分析。

---

AEM 是 iOS 14 时代投流的基础设施。不配置等于在 iOS 市场蒙眼投广告——你花的钱 Facebook 无法精准归因，算法学不到正确信号，循环下去效果越来越差。配置本身不复杂，麻烦在于理解"为什么这样排优先级"——希望这篇文章把这个逻辑讲清楚了。
