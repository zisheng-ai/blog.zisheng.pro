---
title: 修复中文输入法按回车误发送：React Chat Composer 的 IME 事件陷阱
date: 2026-07-21 19:35:00
description: 在 Chat Composer 里，Enter 通常代表发送，但在中文、日文、韩文等 IME 输入过程中，它还承担确认候选词的职责。本文从真实故障出发，拆解 composition 与 keyboard 事件时序、Safari/WebKit 的兼容陷阱，并给出适用于 React 和 Agentic Chat 的稳健修复与测试方案。
categories:
  - [前端]
tags:
  - React
  - IME
  - CompositionEvent
  - KeyboardEvent
  - Agentic Chat
cover: /images/ime-composition-enter-submit.webp
---

最近在做 Agentic Chat 时，我遇到一个看似很小、实际会直接破坏输入体验的问题：使用中文输入法输入英文片段，候选框还开着，此时按 Enter 本来只是确认候选词，Composer 却把这次 Enter 当成了“发送消息”。

这个 Bug 特别容易被纯英文开发环境漏掉。按钮点击正常，英文键盘正常，甚至多数浏览器里只判断 `isComposing` 也能正常；但到了 macOS 输入法和 WebKit 的真实事件时序，消息仍然会提前发出去。

## 一句话总结

Chat Composer 不能把所有 `Enter` 都视为提交。发送前至少要同时检查 `KeyboardEvent.isComposing`、IME 的兼容值 `keyCode === 229`，并处理部分 WebKit 环境中 `compositionend` 早于提交用 `keydown` 到达的时序差异。

最终的判断不是“用户有没有按 Enter”，而是：**这次 Enter 的语义是确认输入法候选，还是提交已经完成的消息？**

## 先理解 IME：输入框里的文字不一定已经提交

IME 是 Input Method Editor。中文拼音、日文假名、韩文以及部分语音和手写输入，并不是每按一个键就立即产生最终字符，而是先进入一个 composition session：

```text
compositionstart
  ↓
compositionupdate × N
  ↓
compositionend
```

在这个阶段，输入框中看起来已经出现了 `auto`，但它可能仍处于输入法的组合态。用户按 Enter 的意图是从候选列表中选择 `automation`，不是把整条聊天消息发给 Agent。

浏览器通过 `KeyboardEvent.isComposing` 暴露当前按键是否发生在 composition session 内。MDN 对它的定义很直接：在 `compositionstart` 之后、`compositionend` 之前触发的键盘事件，该值应为 `true`。

因此最基础的发送逻辑应该是：

```tsx
function handleKeyDown(event: React.KeyboardEvent<HTMLTextAreaElement>) {
  if (event.key !== "Enter") return;

  if (event.nativeEvent.isComposing) {
    return;
  }

  if (!event.shiftKey) {
    event.preventDefault();
    sendMessage();
  }
}
```

这能解决大多数标准事件顺序，但还不够。

## 为什么只判断 isComposing 仍然会误发送

问题出在浏览器和操作系统对 composition 与 keyboard event 的排序并不完全一致。

理想情况下，用户按 Enter 确认候选时，`keydown` 发生在 composition 结束之前：

```text
keydown Enter (isComposing: true)
compositionend
keyup Enter
```

此时 `isComposing` 足以阻止发送。

但 WebKit 在 macOS 上长期存在另一种时序：

```text
compositionend
keydown Enter (isComposing: false)
keyup Enter
```

对业务代码来说，收到 `keydown` 时 composition 已经结束，`isComposing` 自然变成 `false`。如果发送条件只是：

```ts
event.key === "Enter" && !event.nativeEvent.isComposing
```

这次用于确认候选的 Enter 就会穿透保护，直接提交表单。

WebKit Bugzilla 的相关记录明确提到：macOS 上接受 IME 输入的 Enter 可能在 `compositionend` 之后才以 `keydown` 到达，因此 `isComposing === false`。这不是 React 自己的状态问题，而是底层事件顺序已经让上层失去了原始语义。

## keyCode 229 为什么仍然有价值

`keyCode` 已经是 legacy API，正常键盘处理应该优先使用 `event.key` 和 `event.code`。但 UI Events 规范仍然保留了一条与 IME 直接相关的兼容规则：如果 `keydown` 正由 Input Method Editor 处理，返回值是 `229`。

因此，IME guard 中使用 `keyCode === 229` 不是把所有键盘逻辑退回旧 API，而是把它作为一个窄范围的兼容信号：

```ts
const nativeEvent = event.nativeEvent;

const isImeEvent =
  nativeEvent.isComposing ||
  nativeEvent.keyCode === 229;
```

它能覆盖一部分 `isComposing` 不可靠、但浏览器仍把按键标记为 IME processing 的情况。

不过它仍不是完整答案。有些环境可能在 `compositionend` 后把这次 Enter 暴露成普通的 `keyCode 13`。因此还需要记录组件自己的 composition 状态，并给刚结束的 composition 留一个很短的保护窗口。

## 稳健修复：三层 IME Guard

我最后采用了三层判断：

1. 组件内的 `composingRef`，记录 `compositionstart` 到 `compositionend`。
2. 浏览器提供的 `nativeEvent.isComposing` 与 `keyCode === 229`。
3. `compositionend` 后一个短暂的 grace period，挡住乱序到达的 Enter。

React 实现如下：

```tsx
import { useRef } from "react";

export function ChatComposer() {
  const composing = useRef(false);
  const compositionJustEnded = useRef(false);
  const compositionTimer = useRef<ReturnType<typeof setTimeout> | null>(null);

  return (
    <textarea
      onCompositionStart={() => {
        composing.current = true;
        compositionJustEnded.current = false;

        if (compositionTimer.current) {
          clearTimeout(compositionTimer.current);
        }
      }}
      onCompositionEnd={() => {
        composing.current = false;
        compositionJustEnded.current = true;

        if (compositionTimer.current) {
          clearTimeout(compositionTimer.current);
        }

        compositionTimer.current = setTimeout(() => {
          compositionJustEnded.current = false;
        }, 80);
      }}
      onKeyDown={(event) => {
        const nativeEvent = event.nativeEvent;

        const isImeEnter =
          event.key === "Enter" &&
          (
            composing.current ||
            compositionJustEnded.current ||
            nativeEvent.isComposing ||
            nativeEvent.keyCode === 229
          );

        if (isImeEnter) {
          event.preventDefault();
          return;
        }

        if (event.key === "Enter" && !event.shiftKey) {
          event.preventDefault();
          sendMessage();
        }
      }}
    />
  );
}
```

这里的 `80ms` 不是协议常量，而是一个很窄的事件时序缓冲。它只保护 composition 刚结束后紧随而来的 Enter，不会改变用户完成输入后再次按 Enter 发送的习惯。如果产品允许调整发送方式，提供 `Cmd/Ctrl + Enter` 模式仍然是很好的补充，但不能用它掩盖默认 Enter 模式的 IME Bug。

## 在 assistant-ui 里应该在哪里拦截

应用侧使用 assistant-ui 的 `ComposerPrimitive.Input`。这类 headless primitive 通常已经包含基础的 `isComposing` 判断，但宿主应用仍可能需要为特定桌面 WebView、输入法和系统版本补充 guard。

关键是把处理放在 Composer 输入边界，而不是 `sendMessage()` 里事后猜测：

```tsx
<ComposerPrimitive.Input
  onCompositionStart={handleCompositionStart}
  onCompositionEnd={handleCompositionEnd}
  onKeyDown={handleComposerKeyDown}
/>
```

原因有三点：

- 输入组件最接近真实 DOM event，能拿到 composition 信息。
- 在这里 `preventDefault()`，可以阻止 primitive 后续的 submit handler。
- 发送函数不应该知道用户使用了拼音、假名还是物理键盘，它只负责提交已确认的消息。

如果组件库会组合用户传入的 handler 与内部 handler，需要确认组合顺序及 `defaultPrevented` 语义。理想行为是外部 handler 先运行；外部调用 `preventDefault()` 后，内部 Enter-submit 不再执行。

## 不要只测“中文能不能输入”

这个 Bug 的验收重点是事件语义，而不是文字最终有没有出现在 textarea。

至少覆盖下面这些用例：

| 场景 | 期望结果 |
| --- | --- |
| 中文输入法候选打开，按 Enter | 确认候选，不发送 |
| 日文输入法候选打开，按 Enter | 确认候选，不发送 |
| composition 完成后再次按 Enter | 发送一次 |
| 普通英文输入后按 Enter | 发送一次 |
| `Shift + Enter` | 插入换行，不发送 |
| composition 中按 `Shift + Enter` | 交给输入法，不发送 |
| 连续快速确认候选 | 不丢字、不重复发送 |
| Agent 正在运行时输入 | 遵守 queue / steer 策略，不被 IME 改写 |

自动化测试可以构造 `compositionStart`、`keyDown`、`compositionEnd` 序列，但不要只相信 JSDOM。IME 与 WebView 的差异发生在浏览器事件层，至少还需要一轮 macOS Safari / WKWebView、Chrome，以及 Windows 中文输入法的真机测试。

下面是一个简化的测试意图：

```tsx
fireEvent.compositionStart(textarea);
fireEvent.keyDown(textarea, {
  key: "Enter",
  keyCode: 229,
  isComposing: true,
});
fireEvent.compositionEnd(textarea);

expect(sendMessage).not.toHaveBeenCalled();

await waitForCompositionGuardToExpire();
fireEvent.keyDown(textarea, { key: "Enter", keyCode: 13 });

expect(sendMessage).toHaveBeenCalledTimes(1);
```

## 更通用的工程判断：快捷键不能脱离输入状态

这次问题不只属于聊天框。搜索框、命令面板、评论框、AI Prompt Composer、表格快捷编辑都可能把 Enter、Escape、方向键当作产品命令，但这些按键在 IME、自动补全、mention、slash command 和 accessibility 输入中还有另一层语义。

我现在会把键盘处理按下面的优先级设计：

```text
IME composition
  → suggestion / mention / command popover
  → editor-local behavior
  → product shortcut
  → global shortcut
```

越靠前的输入状态，越应该先消费事件。全局快捷键不能因为“捕获到了 Enter”就认为用户要求执行命令。

这也是 Agentic Chat 与普通表单的差别：Composer 一旦误发送，触发的不只是一次 HTTP 请求，而可能是一个拥有文件、终端或外部系统权限的 Agent run。输入确认和任务执行之间必须有可靠边界。

## 要不要用 / 我的判断框架

**值得上：** 所有 Enter-to-submit 的 Chat、搜索、评论和命令输入框，都应该至少判断 `isComposing`；面向 macOS Safari、WKWebView 或 Tauri WebView 的产品，再补 `keyCode 229` 和 composition 结束保护。

**可以再等：** 如果产品强制点击按钮发送，或者唯一快捷键是明确的 `Cmd/Ctrl + Enter`，grace period 的优先级可以降低，但 composition 状态仍应被组件正确维护。

**重点关注：** 不要把 `80ms` 当成神奇数字散落在多个组件里。把 IME guard 封装成 hook 或 Composer adapter，集中处理计时器清理、平台差异和测试。未来替换编辑器或 UI primitive 时，也只需要重新验证这一层。

参考资料：

- [MDN：KeyboardEvent.isComposing](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/isComposing)
- [W3C：UI Events](https://www.w3.org/TR/uievents/)
- [WebKit Bug 165004：macOS 上 keyboard 与 composition 事件顺序](https://bugs.webkit.org/show_bug.cgi?id=165004)
