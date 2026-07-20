---
title: React Native 字体缩放：别为了布局稳定一刀切关掉无障碍
description: React Native 的 allowFontScaling 能修复字号变大后的布局问题，但全局关闭会伤害无障碍体验。本文给出排查路径、组件级处理方式与适合全局限制的边界。
date: 2020-02-20 15:38:31
categories:
  - [前端, React Native]
tags:
  - React Native
  - 无障碍
  - 字体缩放
  - 排障
---

系统字体调大后，React Native 页面出现截断、按钮被顶出屏幕，是很常见的线上问题。最直接的修复是给 `Text` 或 `TextInput` 设置 `allowFontScaling={false}`；但这也会让低视力用户失去系统字号适配。关键不在于「能不能关」，而在于先判断页面是否真的需要固定的视觉尺度。

## 一句话总结

默认尊重系统字体缩放；只有品牌字标、极小的辅助标识等无法随字号变化的元素，才做组件级限制。布局错位应优先通过弹性布局、换行和最小高度修复。

## 先确认问题来自字体缩放

在 iOS 的“辅助功能 → 显示与文字大小 → 更大字体”，或 Android 的“显示 → 字体大小”中调大字号，再检查：

- 文本是否被固定高度容器裁切；
- 图文横向排列是否缺少 `flexShrink` 或换行；
- 输入框是否只按默认行高计算高度；
- 是否把字号、行高和容器高度写死成同一组像素值。

这一步很重要：如果不在真实的大字体环境复现，给某个组件加 `allowFontScaling={false}` 往往只是碰巧掩盖了布局问题。

## 推荐：只对必要组件限制

例如品牌 Logo 旁的短标签，设计上不希望它改变尺寸，可以显式关闭：

```tsx
import { Text } from 'react-native'

export function BrandMark() {
  return <Text allowFontScaling={false}>ZISHENG</Text>
}
```

对于信息密度较高、但仍希望兼顾可读性的文本，使用上限比完全关闭更合适：

```tsx
<Text maxFontSizeMultiplier={1.3}>
  这段内容会跟随系统字号，但最大放大到 1.3 倍。
</Text>
```

`maxFontSizeMultiplier` 适合导航标题、列表副标题等空间受限区域；正文、表单说明和错误提示通常不应设置得过小。

## 用布局解决，而不是禁止用户放大

常见的横向信息行可以这样写：

```tsx
<View style={{ flexDirection: 'row', alignItems: 'center', gap: 8 }}>
  <Icon name="info" />
  <Text style={{ flex: 1, flexShrink: 1, lineHeight: 22 }}>
    文案允许换行，容器高度随内容增长。
  </Text>
</View>
```

避免为文本行设置固定 `height`，并为多行文本留出合理 `lineHeight`。需要单行省略时，再明确设置 `numberOfLines={1}`，不要让裁切变成偶然结果。

## 不建议的全局写法

下面的写法确实能快速让全站不再缩放，但会覆盖所有文本，包括正文、表单和错误提示：

```ts
// 不建议作为默认方案
Text.defaultProps = { ...Text.defaultProps, allowFontScaling: false }
TextInput.defaultProps = { ...TextInput.defaultProps, allowFontScaling: false }
```

它只适合明确以固定字号为产品要求的封闭场景，例如内部大屏、受控硬件终端。普通 App 应把它视为最后手段，并在设计评审中确认无障碍取舍。

## 我的判断框架

值得上：组件级 `maxFontSizeMultiplier`、弹性布局与真机大字号回归测试。

可以再等：为了省事全局关闭 `allowFontScaling`。它会把短期布局成本转移给用户。
