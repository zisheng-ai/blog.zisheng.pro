---
title: 前端组件化实战之 Button
description: 以 Button 为例拆解 React Native 组件库的主题、交互与 API 设计，讨论组件开发中如何在设计语义、可复用性和工程约束之间做取舍。
date: 2023-03-03 10:36:18
categories:
  - [前端, 组件化]
tags:
  - 前端
  - 组件化
  - 组件库
  - React Native
cover: https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b79b60e52f9b4e199372d26724e81a59~tplv-k3u1fbpfcp-zoom-crop-mark:1512:1512:1512:851.awebp
---

> ⚠️本文为掘金社区首发签约文章，未获授权禁止转载

> 大家好，我是[紫升🎋](https://blog.zisheng.pro)，一只住在杭城的木系前端🧚🏻‍♀️，如果你喜欢我的文章📚，可以通过点赞帮我聚集灵力⭐️。


## 前言

在 [《每个前端都应该拥有自己的组件库,就像每个夏天都有西瓜🍉》](https://juejin.cn/post/6983854006124675108) 一文中，紫升带领小黑从零搭建了一个组件库项目，完成了项目结构、构建、测试、文档等基础工程化工作并完成了第一个组件 Icon。本期延续上期的组件工程化的主题，夏日炎热，点上一杯杨枝甘露，和紫升赴一场 Button 开发之约吧。赴约后，你将会收获以下的内容：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58285f3cce144972b2ccb88e11efd73f~tplv-k3u1fbpfcp-zoom-1.image)

> PS：配合[仓库](https://github.com/youngjuning/vant-react-native)和[组件库文档](https://vant-react-native.js.org/)阅读本文效果更佳喲！

## Button 与设计心理学

作为前端工程师，入行至今接触最多的就是设计师了。耳濡目染下虽说没学会什么设计工具，但是对设计与人的心理有了一定认识。

紫升认为任何事物都不可能凭空出现，自有其传承。使用广泛的基础界面元素 Button 也不例外，我们生活中就有随处可见的按钮。举个栗子🌰，每天上班下班必然要按的电梯按钮、手机音量按钮、小米 9 鸡肋的小爱同学唤起按钮。要搞清楚为什么需要按钮，我们有必要探究下生活中这些按钮的作用。

### 点一下按钮的快感

想象一下把键盘按键换成触摸屏，你最在乎的一定是完美还原物理键的敲击感，像紫升用手机虚拟键盘就喜欢设置按键震动和音效。通过打击（点击）获得快感是较为普遍的人性。按钮在按下、松开时有丰富的质感和交互感，完美满足了人们点一下的快感。

![紫升](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/542a6bbb880e4594ae384ee0bd1e511d~tplv-k3u1fbpfcp-watermark.image)

### 现实的实用性

从 BB 机到诺基亚再到如今的智能机，实体按钮削减到只剩下音量键和开关机键。按键虽然光秃秃没有任何标识，但我们就是知道它的功能。试想一下没有这个来自远古时代的开关键，你手里的手机就是一块板砖。

### 疯狂暗示用户，达到不可告人目的

小米 9 单独唤起小爱同学的按键经常会被误按，之前我还不理解这么蠢的设计的目的。在简单研究了点设计心理学我明白了。小爱的设计者为了 产品日活和 AI 训练就是故意这个设计的。

小米 10 虽然移除了单独的唤起键，却把原来的电源键改成了一键多用。每次想要重启手机还得先唤起一下小爱同学。不得不说，小爱同学小米亲女儿。

![紫升](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/633381b8c2c342ed94bf024dbc19cf76~tplv-k3u1fbpfcp-watermark.image)

吐槽归吐槽，小米这个按钮确实起到了培养用户习惯的任务。当用户知悉某个按钮能指向某个操作，或者获取某类信息后，长此以往用户就会形成使用习惯。如果某操作能够为用户和厂商持续带来价值，那就可以让按钮的位置更加醒目，持续培养用户点击习惯。

### 指引用户操作

这个在 Web 开发中是最常见的使用场景，每个可交互页面上都有这类按钮的出现，用来指引用户下一步该怎么做。比如表单的提交和重置。

![紫升](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2c2a8abbe84471f96a67e18196c53fb~tplv-k3u1fbpfcp-watermark.image)

虽然按钮也常作为表单元素，但是区别于其他表单元素，按钮因其天然地自说明性，不需要 Label 对其进行辅助说明，啰嗦这么多，掘友们应该在看到一个按钮时，应该也会有从设计上品鉴的意识了，欢迎将对下图的品鉴在评论区告诉紫升。

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88c6754b28984fc3866319c35790fc03~tplv-k3u1fbpfcp-watermark.image)

## 组件主题化

在开始开发具体组件之前，我们必须先约定好组件主题化的规范。之前 antd-mobile-rn 就因为设计问题，中途花费大力气重构。几乎所有的组件库都会将色彩、布局这些以 css 变量的形式提供给使用者和开发者为，React Native 不同的是样式基于 CSS in JS，不过道理相通，参照 vant 的[设计资源](https://github.com/youzan/vant/blob/dev/src/style/var.less)，我们抽出了一套 JavaScript 常量：

```ts
// packages/themes
export interface Theme {
  'animation-duration-base': string;
  'animation-duration-fast': string;
  'animation-timing-function-enter': string;
  'animation-timing-function-leave': string;
  'font-size-xs': number;
  'font-size-sm': number;
  'font-size-md': number;
  'font-size-lg': number;
  'font-weight-bold': number;
  // 变量过多，这里仅展示部分变量
}
```

有了这些 JS 常量，我们就可以设计主题系统。基于 CSS in JS 的主题化设计一般是基于 React Context 实现，需要提供 ThemeProvider 传入主题上下文，ThemeConsumer、WithTheme（高阶类组件）、withTheme（高阶函数组件） 或 useTheme（React Hooks）作为消费者获取上下文。自己实现也不难，不过更文任务比较紧急，我们先基于 [cssinjs/theming](https://github.com/cssinjs/theming) 实现功能，后期有需要再回来造轮子也不迟。下面👇就是我们基于 theming 的 `createTheming` 函数创建自定义主题上下文。

```ts
import { createTheming } from 'theming';
const context = React.createContext(defaultTheme);
const theming = createTheming(context);

export const { ThemeProvider, withTheme, useTheme } = theming;
```

> 主题功能是通用的，因此我将主题相关的能力都放在 `@vant-react-native/theme` 包中发布。

## Button 的实现

React Native 内置的 Button 组件的样式是固定的，只能进行一些简单的设置。且内置的 Button 组件在 Android 和 ios 两个平台上的表现并不一致。所以我们需要根据更底层的组件进行封装。我们对比 ant-design-mobile-rn 和 react-native-elements 后采用了前者使用的 `TouchableHighlight` 组件。由于继承自 TouchableHighlight，所以我们组件的 Props 类型如下：

```ts
import { TouchableHighlightProps } from 'react-native';
interface ButtonProps extends TouchableHighlightProps {
}
```

### 按钮类型

vant 的 [Button](https://youzan.github.io/vant/#/zh-CN/button#an-niu-lei-xing) 支持 `default`、`primary`、`info`、`warning`、`danger` 五种类型，默认为 `default`。现在，组件的基本定义如下：

```ts
// ...
import React, { FunctionComponent } from 'react';
import { Text, View } from 'react-native';

interface ButtonProps {
  type?: 'default' | 'primary' | 'info' | 'warning' | 'danger';
}

const Button: FunctionComponent<ButtonProps> = props => {
  // ...
};
// ...
```

我们的组件为了适应主题化需求，样式不能是写死在组件里的，而是要通过上下文获取样式常量。我们思路是首先使用 `useTheme` 从上下文中获取主题，然后由于样式定义较多，我们为每个组件编写一个 `useStyle` hook 放在单独的 style.ts 文件中：

```ts
import { StyleSheet } from 'react-native';
import { Theme, useTheme } from '@vant-react-native/theme';

export const useStyle = props => {
  const theme = useTheme<Theme>();

  const getBackgroundColor = () => {
    switch (props.type) {
      case 'primary':
        return theme['success-color'];
      case 'info':
        return theme['primary-color'];
      case 'warning':
        return theme['warning-color'];
      case 'danger':
        return theme['danger-color'];
      default:
        return theme.white;
    }
  };

  const getTextColor = () => {
    if (props.type === 'default') {
      return theme.black;
    } else {
      return theme.white;
    }
  };

  const getBorderRadius = () => {
    if (props.round) {
      return theme['border-radius-max'];
    }
    if (props.square) {
      return 0;
    }
    return theme['border-radius-sm'];
  };

  const styles = StyleSheet.create({
    container: {
      alignItems: 'center',
      backgroundColor: getBackgroundColor(),
      borderColor: getBorderColor(),
      borderRadius: theme['border-radius-sm'],
      borderWidth: theme['border-width-base'],
      flexDirection: 'row',
      flex: 1,
      justifyContent: 'center',
      opacity: 1,
      paddingHorizontal: 15,
    },
    indicator: {
      marginRight: theme['padding-xs'],
    },
    textStyle: {
      color: getTextColor(),
      fontSize: 14,
    },
    wrapper: {
      borderRadius: theme['border-radius-sm'],
      height: 44,
    },
  });
  return styles;
};
```

基于 `useStyle` 我们便可完成一个支持多类型的 Button 组件：

```tsx
const Button: FunctionComponent<ButtonProps> = props => {
  const styles = useStyle(props);
  const { style, ...restProps } = props;
  return (
    <TouchableHighlight style={[styles.wrapper, style]} {...restProps}>
      <View style={styles.container}>
        {typeof props.children === 'string' ? (
          <Text style={styles.textStyle}>{props.children}</Text>
        ) : (
          props.children
        )}
      </View>
    </TouchableHighlight>
  );
};
```

> 注意：子组件可能是字符串，也可能是组件，所以需要判断类型。

实现效果如下：

![紫升](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a0b412c227e4dc994b818294165240d~tplv-k3u1fbpfcp-watermark.image)

### 朴素按钮

朴素按钮的文字为按钮颜色，背景为白色，我们通过 `plain` 属性将按钮设置为朴素按钮。调研了 antd 和 react-native-elements 发现它们都是定义了很多样式，然后在组件内通过逻辑判断计算具体样式的值。个人很不喜欢这种方式，不是彻底的 CSS in JS，我的处理方式是将所有有关样式计算的都封装在每个组件的 `useStyle` 钩子中，比如当引入朴素按钮属性时，相对于普通按钮改变的有容器背景色、容器边框和字体颜色。所以我们将这三个属性的值都通过一个单独的函数计算。对比 antd 的源码，会发现不仅代码更易读，甚至代码量也少了。

```ts
const getBackgroundColor = () => {
  if (props.plain) {
    return theme.white;
  }
  // ...
};

const getTextColor = () => {
  if (props.plain) {
    switch (props.type) {
      case 'primary':
        return theme['success-color'];
      case 'info':
        return theme['primary-color'];
      case 'warning':
        return theme['warning-color'];
      case 'danger':
        return theme['danger-color'];
      default:
        return theme['gray-3'];
    }
  } else if (props.type === 'default') {
    return theme.black;
  } else {
    return theme.white;
  }
};
```

实现效果如下：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0fc1bedf56fe4dd0a31e6d18b048665c~tplv-k3u1fbpfcp-watermark.image)

### 细边框

vant 实现细边框是通过设置 `hairline` 属性可以展示 0.5px 的细边框。但是手机上由于分辨率的影响，贸然设置 0.5 会导致边框不显示的兼容问题。好在 React Native 为我们提供了 `StyleSheet.hairlineWidth` 常量来兼容最细边框问题，下面是官方对它的定义：

> hairlineWidth 这一常量始终是一个整数的像素值（线看起来会像头发丝一样细），并会尽量符合当前平台最细的线的标准。可以用作边框或是两个元素间的分隔线。然而，你不能把它“视为一个常量”，因为不同的平台和不同的屏幕像素密度会导致不同的结果。
>
> 如果模拟器缩放过，可能会看不到这么细的线。

由于 `hairline` 只影响了容器 `borderWidth` 属性，我们不需要编写单独的样式计算函数：

```ts
const styles = StyleSheet.create({
  // ...
  container: {
    // ...
    borderWidth: props.hairline ? theme['border-width-hairline'] : theme['border-width-base'],
  },
});
```

实现效果如下：

![紫升](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43d80d537b07487d9043d718e6541f7e~tplv-k3u1fbpfcp-watermark.image)

### 禁用状态

表单元素或者说可触摸可点击的元素一般都有禁用状态，vant 中是通过 disabled 属性来禁用按钮，禁用状态下按钮不可点击。TouchableHighlight 继承地有 `disabled` 属性，我们只需要设置一些禁用状态下的按钮样式就可以，查看 vant 源码我们发现只需要修改透明度为 0.5 即可：

```ts
const styles = StyleSheet.create({
  container: {
    // ...
    opacity: props.disabled ? 0.5 : 1,
    // ...
  },
});
```

实现效果如下：

![紫升](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/892589086d07426a8346319d8466bccf~tplv-k3u1fbpfcp-watermark.image)

### 加载状态

vant 是通过 `loading` 属性设置按钮为加载状态，加载状态下默认会隐藏按钮文字，可以通过 `loading-text` 设置加载状态下的文字。我们借助 React Native 的 ActivityIndicator 组件可以轻松实现这个特性：

```tsx
// ...
<TouchableHighlight {...restProps}>
  <View style={styles.contentWrapper}>
    {props.loading ? (
      <>
        <ActivityIndicator size="small" color={indicatorColor} style={styles.indicator} />
        {props.loadingText ? <Text style={styles.textStyle}>{props.loadingText}</Text> : null}
      </>
    ) : null}
  </View>
</TouchableHighlight>
// ...
```

样式如下：

```ts
export const useIndicatorColor = (props: ButtonProps): string => {
  const theme = useTheme<Theme>();
  if (props.plain) {
    switch (props.type) {
      case 'primary':
        return theme['success-color'];
      case 'info':
        return theme['primary-color'];
      case 'warning':
        return theme['warning-color'];
      case 'danger':
        return theme['danger-color'];
      default:
        return theme.black;
    }
  } else if (props.type === 'default') {
    return theme.black;
  } else {
    return theme.white;
  }
};
```

实现效果如下：

![紫升](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27c4340df2254245a3c830bf667a2e28~tplv-k3u1fbpfcp-watermark.image)

### 按钮形状

默认的按钮有值为 2 的圆角，vant 中通过 `square` 设置方形按钮，通过 `round` 设置圆形按钮。按例，我们通过判断设置样式：

```ts
const getBorderRadius = () => {
  if (props.round) {
    return theme['border-radius-max'];
  }
  if (props.square) {
    return 0;
  }
  return theme['border-radius-sm'];
};
const styles = StyleSheet.create({
  container: {
    borderColor: getBorderColor(),
  },
  wrapper: {
    borderRadius: getBorderRadius(),
  },
});
```

实现效果如下：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1402517566e49d1a1920fd9b3f7bb20~tplv-k3u1fbpfcp-watermark.image)

### 按钮尺寸

Antd RN 只提供了 large、small 两个尺寸，而在 vant 中支持 large、normal、small、mini 四种尺寸，默认为 normal。虽然写到这里已经很疲倦了，杨枝甘露也早喝完了，但是为了完整复原，还是续上一杯咖啡继续肝。根据 vant 设计稿我们新增三个样式获取函数并动态化指定样式：

```ts
const getSizeHeight = () => {
  switch (props.size) {
    case 'large':
      return 50;
    case 'small':
      return 32;
    case 'mini':
      return 24;
    default:
      return 44;
  }
};
const getSizePadding = () => {
  switch (props.size) {
    case 'small':
      return 8;
    case 'mini':
      return 4;
    default:
      return 15;
  }
};
const getSizeFontSize = () => {
  switch (props.size) {
    case 'large':
      return 16;
    case 'small':
      return 12;
    case 'mini':
      return 10;
    default:
      return 14;
  }
};

const styles = StyleSheet.create({
  container: {
    paddingHorizontal: getSizePadding(),
  },
  textStyle: {
    fontSize: getSizeFontSize(),
  },
  wrapper: {
    height: getSizeHeight(),
  },
});
```

实现效果如下：

![紫升](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59c01002f25b477d85ed26fa1e21c86b~tplv-k3u1fbpfcp-watermark.image)

### 自定义颜色

如果不是自己亲自复刻 Vant，是没想到一个 Button 能玩出这么多花，支持特性这么多耐心和代码管理都是一个挑战。当然了，紫升采取的样式管理方式比较偏激，大家有好的方式也可以在评论区讨论。

通过 `color` 属性自定义按钮的颜色。我们可以得出需求，不管 type 是什么，`color` 属性需始终覆盖原有样式，color 能影响的就是背景色、字体颜色和边框颜色，所以我们修改 `getBackgroundColor`、`getTextColor`、`getBorderColor` 样式函数在合适的地方加上以下代码即可：

```ts
if (props.color) {
  return props.color;
}
```

实现效果如下：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8efffdbfd6424618be6152895a53e8ee~tplv-k3u1fbpfcp-watermark.image)

## 双击事件的实现

我们从 React Native 内置的 TouchableHighlight 组件继承了很多事件，其中 onPress、onLongPress 分别代表单击和长按。但唯独“双击 666”的双击事件没有姓名。之前在实际业务曾经封装过双击事件，这次我们就直接就内置了。

实现思路是延时执行单击事件（默认 200 毫秒），然后记录点击次数和两次时间间隔，当识别为第二次点击且时间间隔小于单击延时时间。那么就取消单击事件延时，并立即执行双击事件。完整代码如下：

```tsx
let lastTime = 0;
let clickCount = 1;
let timeout = null;
const _onPress = (event: GestureResponderEvent) => {
  const now = Date.now();
  if (timeout) {
    clearTimeout(timeout);
  }
  timeout = setTimeout(() => {
    props.onPress(event);
    clickCount = 1;
    lastTime = 0;
  }, props.delayDoublePress);
  if (clickCount === 2 && now - lastTime <= props.delayDoublePress) {
    clearTimeout(timeout);
    clickCount = 1;
    lastTime = 0;
    props.onDoublePress(event);
  } else {
    clickCount++;
    lastTime = now;
  }
};
```

大家会发现这里的实现糅合了函数防抖、节流以及计数器的原理，有兴趣的小伙伴可以自行复习下原理，这里就不展开了。

## API 文档

一个组件的文档，除了 Demo，还需要展示出来可用的 Props，Dumi 内置的 `<API></API>` 组件可以根据组件自动生成 API 文档。首先我们像下面一样编写 Props 注释：

```ts
interface ButtonProps extends TouchableHighlightProps {
  /**
   * @description       Can be set to primary、info、warning、danger
   * @description.zh-CN 类型，可选值为 primary、info、warning、danger
   */
  type?: 'default' | 'primary' | 'info' | 'warning' | 'danger';
  /**
   * @description       Can be set to large、small、mini
   * @description.zh-CN 尺寸，可选值为
   */
  size?: 'large' | 'normal' | 'small' | 'mini';
}
```

然后在 Markdown 中引入 API 组件即可：

```md
<API src="./index.tsx"></API>
```

内置组件 API 没有处理继承的情况，我们后续会自定义一个 API 组件，这里就不展开了，浏览 [Button 文档](https://vant-react-native.js.org/components/button#api) 可以查看现在的效果：

![紫升](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7bedac7240334ff79c4142005a66b158~tplv-k3u1fbpfcp-watermark.image)

## 工程化串讲

由于很难在一篇文章中将组件开发相关的工程化讲完，我们需要在每篇实战中串讲一下。

### 组件创建脚手架

> 小黑：紫升，`lerna create` 命令创建出来的模块并不是我们想要的，以后要创建很多很多组件，我们可以写一个创建组件模块的脚手架吗？

lerna 使用起来是有不少痛点的，`lerna create` 命令没办法指定模板，考虑到之后的几十上百个组件每次创建都要进行项目结构、Typescript 配置、单元测试配置、Babel 配置等等工作步骤，我们有必要写一个脚手架。

#### 模板解析

说到模板解析，相信大家和我一样想到的是 vue-cli 的 template 解析。通过阅读 [vue-cli@2.9.6 generate.js](https://cdn.jsdelivr.net/npm/vue-cli@2.9.6/lib/generate.js) 源码，我们可以分析出尤大是基于 metalsmith、handlebars、consolidate 这三个包来实现模板解析能力的。让人不安的是其中 metalsmith 库有长达 5 年没有维护了，紫升挑选开源项目一般对维护度很敏感，本着轮子要用自己造的原则，我翻看了 Metalsmith 的 Readme 发现这个插件无非是通过递归读文件的方式渲染模板，并且它的静态网站生成的能力对我们模板解析的需求也是多余的。

说干就干，在和 [@林小帅](https://juejin.cn/user/3175045313873943) 同学简单沟通后，我动手造了 [handlebars-template-compiler](https://github.com/youngjuning/handlebars-template-compiler) 这个轮子，其主要原理如下：

1. 使用 recursive-readdir 递归获取所有文件路径

```js
const files = await recursive(rootDir);
```

2. 使用 `handlebars.compile` 方法使用元数据对模板进行渲染

```js
const content = fs.readFileSync(file).toString();
const result = handlebars.compile(content)(meta);
```

3. 使用 `fs.writeFileSync` API 重写文件

另外，通过引入 [glob](https://www.npmjs.com/package/micromatch) 模式匹配实现了 `exclude` 配置以及只处理指定后缀（默认 `**/*.tpl.*`）的文件来避免不必要的渲染。（PS：NPM 一周有了 300 多下载，有需要的掘友值得一试😄）

#### Node CLI（@vant-react-native/scripts）搭建

这里紫升尝试用最简洁的语言为大家描述一个脚手架的诞生，源码在 [packages/scripts](https://github.com/youngjuning/vant-react-native/tree/main/packages/scripts) 目录下，没有接触过 CLI 的掘友请相信我，Node CLI 很容易上手的。接触过的同学也可以查漏补缺借鉴一二。

1. `package.json` 文件的 `bin` 字段是我们脚手架的入口

```json
// 指定可执行文件的位置以及别名
"bin": {
  "vant": "./bin/cli.js"
},
```

2. 定义 `./bin/cli.js` 为可执行文件并调用 `init` 方法。

```js
// 由于我们的脚本是 Node 编写的，所以需要指定 node 所在位置
#!/usr/bin/env node
const { init } = require('../lib');
// 这个地方参考了 create-react-native 的设计
// 本文点赞过 300，下一篇紫升带小黑为大家带来《基于 TypeScript 重构 create-react-native》
init();
```

3. 然后在 `src/index.ts` 中初始化 commander 这个久负盛名的命令行框架

```ts
const init = (): void => {
  const packageJson = require('../package.json');
  program.version(packageJson.version).description(packageJson.description);
  // ...
  program.parse(process.argv);
};
```

4. 为了方便管理命令，我们将命令都放置在 `src/commands` 目录下并通过 `fs.readdirSync` API 动态扫描注册。

```ts
const init = (): void => {
  // 这段代码借鉴自 NeteaseCloudMusicApi 项目，作者的代码很有设计感，推荐阅读。
  fs.readdirSync(path.join(__dirname, 'commands')).forEach((file: string) => {
    if (!file.endsWith('.js')) return;
    require(path.join(__dirname, 'commands', file));
  });
  // ...
};
```

5. 最后在 `commands` 目录下新建一个 `create.ts` 文件编写命令

```ts
import { program } from 'commander';
program
  .command('create <name> [loc]')
  .description('Create a new vant-react-native package')
  .action((name,loc) => {
    console.log('Hello Luozhu');
  })
```

#### 脚手架实现

上一小结，我们初始化了 CLI 并添加了 `create` 命令，这一小节我们就来实现一下脚手架功能。

我们首先在 `packages/scripts` 目录下创建组件模板

```sh
.
├── README.tpl.md # tpl 后缀在生成组件模板的时候会被 handlebars-template-compiler 自动去掉。
├── package.tpl.json
├── src
│   └── index.ts # 没有 tpl 后缀则不会被编译，模板很大时可以节省时间。
└── tsconfig.json
```

然后我们明确我们的模板元数据的数据结构，我这里的数据结构是：

```ts
interface IMeta {
  name: string;
  version: string;
  description: string;
  author: string;
  email: string;
  url: string;
  directory: string;
}
```

有了数据结构，我们就可以使用 inquirer 模块引导用户输入信息。

```ts
import inquirer from 'inquirer';
// ...
// getQuestions 过长，感兴趣的同学可以查看：http://tny.im/UFbg
const answer: IMeta = await inquirer.prompt(getQuestions(name));
// ...
```

下一步，我们使用 `tmp-promise` 模块创建一个系统临时文件夹，并将前文提到的 template 文件夹的内容拷贝进去：

```ts
import tmp from 'tmp-promise';
import fs from 'fs-extra';
import path from 'path';
// ...
const tmpdir = await tmp.dir({ unsafeCleanup: true });
fs.copySync(path.join(__dirname, '../../template'), tmpdir.path);
```

最后，我们对临时文件夹的内容进行编译再拷贝到指定位置即可：

```ts
import htc from 'handlebars-template-compiler';
// ...
await htc<IMeta>(answer, tmpdir.path);
fs.copySync(tmpdir.path, `${process.cwd()}/packages/${locPath}`);
// ...
```

折腾这一顿，让我们来看下成果吧：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ba477bcbd8b4ef6b880c7e8e5a095bc~tplv-k3u1fbpfcp-zoom-1.image)

### Github CODEOWENERS

大型的开源项目最难的不是技术问题，技术大咖永远不会缺。最难的其实是协作和后期维护。试想一下一个成百上千人参与的项目当有新的 pr 时，正常人根本无力去快速检索出需要谁去 review 代码。我们的 vant-react-native 由于是将每个组件单独发包维护，当参与的小伙伴多了也会产生这个困扰。

而 GitHub CODEOWNERS（代码所有者）就是为了解决这个问题的，在 5000+ 贡献者参与的 [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) 项目中我们就可以看到它的身影。官方对代码所有者定义如下：

> 你可以使用 CODEOWNERS 文件定义负责仓库代码的个人或团队。当有人修改代码并打开一个 pull request 时，将自动请求代码所有者进行审查。

CODEOWNERS 文件使用遵循 gitignore 文件中所用大多数规则的模式，CODEOWNERS 文件位置一般位于 `.github/` 目录下。

在 vant-react-native，紫升是仓库的最终负责人，所以是期望每个 pr 都可以分配给自己审查一下的。那么我们这就来实验一下吧，新建一个 `.github/CODEOWNERS` 文件并写入以下内容：

```
# This is a comment.
# Each line is a file pattern followed by one or more owners.

# These owners will be the default owners for everything in
# the repo. Unless a later match takes precedence,
# @youngjuning will be requested for review when someone opens a pull request.
*       @youngjuning

# In this example, @doctocat owns any files in the build/logs
# directory at the root of the repository and any of its
# subdirectories.
/packages/ @luozhu1994
```

一般如果文件具有代码所有者，则在打开拉取请求之前可以看到代码所有者是谁。在仓库中，你可以找到文件并悬停于一个锁图标上，悬浮之后会告诉你该文件所有者是谁：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/726bd5144a104902ae27ad31e1d46f93~tplv-k3u1fbpfcp-zoom-1.image)

然后我们提交一个 pr 看看效果：

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf4cb8b7ef5c41989660e735e621fc4e~tplv-k3u1fbpfcp-zoom-1.image)

### NPM 发包自动化

发包权限一般只有仓库所有者一个人拥有，但是 owner 同时维护好几个 NPM 账号，或者是 owner 忽然很忙将发布权限交给其他人管理员但是不便告知 NPM 账号该怎么办呢？答案是将 NPM 发包 CD（持续部署）化，公司一般会基于 Gitlab 或自建平台实现该功能。作为开源项目，我们当然是使用 GitHub Action。

正常的单包项目，使用 [npm-publish](https://github.com/JS-DevTools/npm-publish) 或 [npm-publish-action](https://github.com/pascalgn/npm-publish-action) 这两个 GitHub Action，这并没有好讲的。但是基于 lerna 的多包单体仓库并没有现成的插件可以用，照例，我们来看下自己实现的步骤：

1. 判断 commit message 是否以 `chore(release):` 开头
    > 通过 GitHub Action `startsWith(github.event.head_commit.message, 'chore(release):')` 实现
2. 通过 NPM publish token 认证登录
    > 通过 `npm config set //registry.npmjs.org/:_authToken=${{ secrets.NPM_TOKEN }}` 认证
3. 执行 `lerna publish from-package --yes` 发布
    > 需要本地先执行 `lerna version` 系列命令提升版本

完整 GitHub Action 实现如下：

```yml
name: npm-publish

on:
  push:
    branches:
      - main

jobs:
  npm-publish:
    runs-on: ubuntu-latest
    if: startsWith(github.event.head_commit.message, 'chore(release):')
    steps:
      - uses: actions/checkout@v2
      - uses: c-hive/gha-yarn-cache@v2 # 缓存 node_modules 加快构建速度
      - name: Install Packages
        run: yarn install --registry=https://registry.npmjs.org/
      - name: Authenticate with Registry
        run: |
          npm config set //registry.npmjs.org/:_authToken=${NPM_TOKEN}
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
      - name: Publish package
        run: lerna publish from-package --yes
```

为了在发布后及时获取通知，紫升使用了 `peter-evans/commit-comment` 插件在发布失败或成功后对相应 commit 进行评论，这样我们就可以收到邮件和站内通知。

```yml
- name: Create commit comment after publish successfully
  if: ${{ success() }}
  uses: peter-evans/commit-comment@v1
  with:
    body: |
      Hello Dear @youngjuning. This commit has been publish to NPM successfully.
      > Created by [commit-comment][1]

      [1]: https://github.com/peter-evans/commit-comment
- name: Create commit comment after publish unsuccessfully
  if: ${{ failure() }}
  uses: peter-evans/commit-comment@v1
  with:
    body: |
      Hello Dear @youngjuning. This commit has been publish to NPM unsuccessfully.
      > Created by [commit-comment][1]

      [1]: https://github.com/peter-evans/commit-comment
```
