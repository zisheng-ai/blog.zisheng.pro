---
title: 用 Claude Code + Remotion 复刻 RunCat 奔跑的猫帧动画
date: 2026-07-03 20:00:00
description: RunCat 菜单栏那只随 CPU 使用率加速奔跑的小猫，本质是 5 帧 PNG 的循环 sprite 动画。这篇文章先扒开它的源码，拆解帧循环和速度公式，再用 Claude Code + Remotion 把这套逐帧动画在 React 里声明式复刻出来、渲染成可无限循环的 GIF，并讲清楚 Remotion「纯函数帧模型」的适用边界，以及这条工作流里 AI 到底加速了什么。
categories:
  - [AI]
tags:
  - Claude Code
  - Remotion
  - 帧动画
  - 前端开发
  - RunCat
  - 创意编程
---

macOS 菜单栏那只跟着 CPU 使用率越跑越快的小猫（[RunCat](https://github.com/Kyome22/menubar_runcat)），大概是我用过最治愈的系统监控工具。作为一个从前端转 Agent 开发的人，我一直好奇：这种逐帧 sprite 动画，能不能用现代 React 工具链声明式地复刻出来？答案是 **Remotion**——一个用 React 写视频的框架。这篇文章我用 Claude Code + Remotion 把这只猫从头搭一遍，顺带把「帧动画」这件事讲透。

## 一句话总结

RunCat 的猫本质只有 **5 张 PNG**，靠「取模循环 + 定时器间隔」跑起来。Remotion 的**纯函数帧模型**天生适合表达这种逐帧动画：把「第几帧显示哪张图」写成一个纯函数即可。但要认清边界——**Remotion 产出的是渲染好的 GIF/视频资产，不是常驻菜单栏的原生进程**。想要真·菜单栏猫得靠 Tauri/Electron，Remotion 负责的是把动画本身优雅地造出来。

## 先扒开 RunCat：它到底怎么动

很多人以为菜单栏动画是什么黑魔法，扒完源码发现朴素得可爱。RunCat 的核心就三件事。

### 1. 5 帧 PNG 循环

仓库 `Assets.xcassets` 里躺着 `cat0.png` 到 `cat4.png`，一共 5 帧。帧循环逻辑一行搞定：

```swift
index = (index + 1) % frames.count   // 4 之后回到 0，无限循环
statusItem.button?.image = frames[index]
```

这个 `% frames.count`（取模）就是所有 sprite 循环动画的灵魂——记住它，等下 Remotion 里还是它。

### 2. CPU 使用率决定「跑多快」

猫跑多快，取决于定时器的触发间隔。RunCat 的速度公式是：

```swift
interval = 0.2 / max(1.0, min(20.0, usage / 5.0))
```

拆开看：

- `usage / 5.0`：把 CPU 百分比缩放一下
- `min(20, max(1, ...))`：钳制在 1~20 倍之间，防止闲时卡成 PPT、满载时快到抽搐
- `0.2 / 倍率`：基础间隔 0.2 秒，倍率越大间隔越短

所以 **CPU 越高 → 倍率越大 → 间隔越短 → 换帧越快 → 猫跑得越急**。空闲时 0.2 秒一帧，满载时能到 0.01 秒一帧。就这么个反比关系，体感却极其直观。

### 3. 定时器驱动

```swift
Timer(timeInterval: self.interval, repeats: true) { self?.next() }
```

每隔 `interval` 触发一次 `next()`，`next()` 里就是上面那句取模换帧。**这是典型的「有状态命令式」动画**：一个 `index` 变量，定时 `+1`。

记住这个「命令式、有状态」的特征——因为 Remotion 的模型跟它**恰好相反**，这是本文最值得玩味的地方。

## 为什么用 Remotion 做帧动画

Remotion 的核心信条只有一句：**每一帧都是 `useCurrentFrame()` 的纯函数**。

官方原话是——把你的组件想象成一个「把帧号转换成图像」的函数。每一帧独立渲染，不依赖上一帧的状态。这意味着 CSS 动画、`setInterval`、`requestAnimationFrame` 那套在 Remotion 里几乎全部失效，你必须重新用「帧号 → 画面」的思路建模。

对比一下就懂了：

| | RunCat（命令式） | Remotion（声明式） |
| --- | --- | --- |
| 状态 | 维护一个 `index` 变量 | 无状态，只读 `frame` |
| 换帧 | 定时器 `index++` | `Math.floor(frame / n) % 5` |
| 时间 | 真实时钟（`Timer`） | 逻辑帧号（`useCurrentFrame`） |
| 产物 | 实时菜单栏画面 | 渲染成 GIF/视频 |

看出来了吗？RunCat 那句 `(index + 1) % 5` 的「取模」精神，在 Remotion 里变成了**用当前帧号直接算出该显示第几张 sprite**——不需要状态，因为帧号本身就是「时间的坐标」。这就是纯函数帧模型的优雅之处。

## 动手：Claude Code + Remotion 复刻奔跑的猫

### 第一步：让 Claude Code 起项目

我的做法是直接把需求丢给 Claude Code，让它搭脚手架：

```
初始化一个 Remotion 项目，创建一个叫 RunningCat 的 Composition：
- 144x144，30fps
- 加载 public/cat/ 下的 0.png~4.png 五帧 sprite
- 根据传入的 load（模拟 CPU 使用率）参数，用 RunCat 的速度公式
  interval = 0.2 / clamp(load/5, 1, 20) 换算成换帧节奏
- 像素图，用 imageRendering: pixelated 保持锐利
```

`npx create-video@latest` 起模板后，Claude Code 会把组件、`Root.tsx` 注册、`public/` 资源路径一次性搭好。**帧素材**可以直接扒 RunCat 仓库 `resources/cat/png/` 里的 5 张图，或者让 AI 图像工具生成一套你自己的 5 帧奔跑序列。

### 第二步：核心 sprite 组件

这是全文的重点。RunCat 的「取模换帧」在 Remotion 里长这样：

```tsx
// src/RunningCat.tsx
import {
  AbsoluteFill, Img, staticFile,
  useCurrentFrame, useVideoConfig,
} from 'remotion';

const CAT_FRAMES = [0, 1, 2, 3, 4].map((i) => staticFile(`cat/${i}.png`));

export const RunningCat: React.FC<{ load: number }> = ({ load }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  // 照搬 RunCat 的速度公式
  const speed = Math.max(1, Math.min(20, load / 5));
  const secondsPerSprite = 0.2 / speed;            // 每张 sprite 停留的秒数
  const framesPerSprite = Math.max(               // 换算成「视频帧」
    1,
    Math.round(secondsPerSprite * fps),
  );

  // 灵魂取模：帧号 → 第几张 sprite
  const spriteIndex =
    Math.floor(frame / framesPerSprite) % CAT_FRAMES.length;

  return (
    <AbsoluteFill style={{ justifyContent: 'center', alignItems: 'center' }}>
      <Img
        src={CAT_FRAMES[spriteIndex]}
        style={{ height: 144, imageRendering: 'pixelated' }}
      />
    </AbsoluteFill>
  );
};
```

关键就那一行 `Math.floor(frame / framesPerSprite) % CAT_FRAMES.length`：

- `frame / framesPerSprite`：当前帧对应到「第几个 sprite 周期」
- `Math.floor(...)`：取整，同一张图在多个视频帧里保持不变
- `% 5`：跑完 5 张回到第 0 张——**和 RunCat 那句 `% frames.count` 是同一个灵魂**

### 第三步：注册 Composition

```tsx
// src/Root.tsx
import { Composition } from 'remotion';
import { RunningCat } from './RunningCat';

export const RemotionRoot: React.FC = () => (
  <Composition
    id="RunningCat"
    component={RunningCat}
    durationInFrames={60}     // 见下方「无缝循环」的坑
    fps={30}
    width={144}
    height={144}
    defaultProps={{ load: 50 }}
  />
);
```

**无缝循环的坑**：GIF 要循环不跳帧，`durationInFrames` 必须是「一个完整动画周期」的整数倍，也就是 `framesPerSprite * 5` 的倍数。比如 `load=50` 时 `speed=10`、`framesPerSprite = round(0.02*30)=1`，一个周期 5 帧，那 `durationInFrames` 取 5 的倍数（如 60）才能首尾对齐。这个细节 Claude Code 不一定主动帮你算，得自己盯一下。

### 第四步：渲染成 GIF

```bash
# 基础渲染
npx remotion render RunningCat out/cat.gif --codec=gif

# 透明背景（sprite 是抠好的 PNG 时必备）
npx remotion render RunningCat out/cat.gif --codec=gif --image-format=png

# 降帧压体积：只渲染每 2 帧，30fps → 15fps
npx remotion render RunningCat out/cat.gif --codec=gif --every-nth-frame=2

# 无限循环（GIF 默认就是无限循环，这里显式写出）
npx remotion render RunningCat out/cat.gif --codec=gif --number-of-gif-loops=null
```

跑完你就得到一只无限循环、背景透明的奔跑猫 GIF，可以塞进博客、README、PPT 或者任何网页。

## 进阶：让猫「真的」随负载变速

上面 `load` 是个固定值，猫匀速跑。想复刻 RunCat 那种「负载飙升猫就抽风」的动态感，有两条路。

**渲染成动态 GIF**：用 `interpolate` 让 `load` 随帧号变化，模拟一次 CPU 尖峰：

```tsx
import { interpolate } from 'remotion';

const load = interpolate(
  frame,
  [0, 45, 90, 135, 180],
  [10, 95, 30, 100, 10],   // 闲 → 爆 → 缓 → 再爆 → 闲
  { extrapolateRight: 'clamp' },
);
```

> **一个容易被忽略的深度坑**：当速度随帧动态变化时，`Math.floor(frame / framesPerSprite)` 里的 `framesPerSprite` 每帧都在变，除法结果不再单调递增，sprite 会跳帧。严格正确的做法是**对速度做相位累积**（把每帧的换帧增量累加起来），而不是简单除法。RunCat 靠有状态的 `index++` 天然规避了这个问题，但 Remotion 的纯函数模型逼你把「累积」显式写出来——这正是命令式转声明式最反直觉的地方。做匀速 demo 可以偷懒用除法，做严肃动态动画就得老实累积。

**驱动实时网页版**：如果你要的不是视频而是一个活的网页挂件，用 `@remotion/player` 把同一个组件搬进浏览器，再把真实性能指标喂给 `load`，就是一个跑在网页里的实时 RunCat。同一份组件，既能渲染成 GIF，又能实时播放——这是 Remotion 相比纯 CSS 动画的一大优势。

## Claude Code 在这条流程里加速了什么

说句实在话，Remotion 的 API 不难，难的是「纯函数帧模型」的思维转换。Claude Code 的价值分三层：

- **脚手架层（省时间）**：起项目、配 `Composition`、连 `public/` 资源、写渲染脚本，这些样板活它一把梭。
- **翻译层（省脑子）**：把 RunCat 那句 Swift 的命令式 `index++` 翻译成 Remotion 的声明式取模，这种「换范式重写」是 AI 的强项——你给它源码逻辑，它给你目标框架的地道实现。
- **调试层（省试错）**：无缝循环的帧数对齐、动态变速的相位累积这类坑，描述清楚现象后它能直接给修法，比自己翻文档快得多。

但**「取模换帧」的原理、「相位累积」的坑、无缝循环的数学**——这些判断你得心里有数，否则 AI 给的代码你既 review 不了也调不动。这也印证了我一直的观点：**AI 拉满执行力，判断框架还得是自己的**。

## 要不要这么搞：我的判断

- **值得动手**：你想给博客/README/演示做一个有辨识度的循环动画，或者想亲手理解「逐帧 sprite 动画」和「声明式帧模型」——RunCat 复刻是绝佳的练手项目，5 帧图、一个取模，麻雀虽小五脏俱全。
- **别用错工具**：你要的是一个**常驻 macOS 菜单栏、读真实 CPU** 的原生小猫——那 Remotion 不是答案，老老实实用 Swift（RunCat 本体）或 Tauri/Electron。Remotion 造的是动画资产，不是系统进程。
- **重点收获**：真正值钱的不是这只猫，而是**「命令式有状态动画 → 声明式纯函数帧」的思维转换**。理解了 `useCurrentFrame` 这套模型，你能用 React 声明式地造出几乎任何数据驱动的视频——这才是 Remotion 值得前端认真学一次的原因。

一只 5 帧的猫，串起了 sprite 动画原理、纯函数帧模型、和 AI 辅助的范式翻译。有时候最好的学习项目，就是把一个你每天都在看的小东西，亲手拆开再装回去。

> 参考来源：[menubar_runcat 源码](https://github.com/Kyome22/menubar_runcat) · [Remotion useCurrentFrame 文档](https://www.remotion.dev/docs/use-current-frame) · [Remotion 渲染 GIF 文档](https://www.remotion.dev/docs/render-as-gif)
