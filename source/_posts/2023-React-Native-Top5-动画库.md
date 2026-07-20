---
title: 2023 React Native Top5 动画库
date: 2023-04-06 23:53:15
cover: https://cdn.jsdelivr.net/gh/youngjuning/images@main/1680796408627.png
description: 在技术层面，React Native 为我们提供了强大的声明式 API 去创建动画，通常我们称之为 `Animated API`。然而，有时你可能想使用一些第三方库来为你处理动画，而不用直接处理 `Animated API`，本文我们便是探索和讨论 5 个你值得是使用的 React Native 插件。
categories:
  - [前端, React Native]
  - [编程]
tags:
  - React Native
  - React Native 组件库
  - React Native Animation Libraries
---

动画在手机上有很大的影响，它可以创造更好的用户体验，这些动画主要用于与用户的行为互动，使用户更多的参与你的应用程序。

在技术层面，React Native 为我们提供了强大的声明式 API 去创建动画，通常我们称之为 `Animated API`。然而，有时你可能想使用一些第三方库来为你处理动画，而不用直接处理 `Animated API`，本文我们便是探索和讨论 5 个你值得是使用的 React Native 插件。

## 重用 React 组件

[Bit]: https://bit.dev/

使用 [Bit][Bit] 可以跨平台分享和重用 React 组件。作为一个团队在共享组件上进行协作，可以更敏捷地共同构建应用程序。让 Bit 做繁重的工作，这样你就可以轻松地发布、安装和更新你的个人组件而不需要任何开销。

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1c5f1759c924881ba62904a329645ef~tplv-k3u1fbpfcp-zoom-1.image)

## react-native-animatable

[animatable]: https://github.com/oblador/react-native-animatable

我是 [react-native-animatable][animatable] 的忠实粉丝。它为你提供了声明性的包装器，你可以用它来为 React Native 中的元素制作动画，这个库的好处是它的 API 很容易使用，你不需要做任何 linking 操作。所以让我们举个例子，看看这个库是如何工作的😃。

首先，安装该库！

```sh
yarn add react-native-animatable
```

然后，让我们制作一个 slideInDown 动画效果！


![1_amBoBML8nY2DxpuNC5MbNQ.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abc35f9d7b304742b867c58f3d07f7e5~tplv-k3u1fbpfcp-watermark.image)

我们的组件看起来如下：

```jsx | pure
import React, {Component} from 'react';
import {Text, View, Dimensions, SafeAreaView, StyleSheet} from 'react-native';
import * as Animatable from 'react-native-animatable';

class AnimatableScreen extends Component {
  render() {
    return (
      <SafeAreaView style={styles.container}>
        <View style={{alignItems: 'center'}}>
          <Animatable.View
            style={styles.card}
            animation="slideInDown"
            iterationCount={5}
            direction="alternate">
            <Text style={styles.whiteText}>slideInDown Animation</Text>
          </Animatable.View>
        </View>
      </SafeAreaView>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    width: Dimensions.get('screen').width,
    height: Dimensions.get('screen').height,
    justifyContent: 'center',
  },
  card: {
    width: Dimensions.get('screen').width * 0.6,
    height: Dimensions.get('screen').height * 0.35,
    backgroundColor: '#206225',
    alignItems: 'center',
    justifyContent: 'center',
    borderRadius: 8,
  },
  whiteText: {
    color: '#ffffff',
    fontSize: 18,
  },
});

export default AnimatableScreen;
```

我们首先通过 `react-native-animatable` 导入了 Animatable 对象，然后我们将 Animatable 元素与我们想要动画化的元素结合起来，你可以使用 Image 和 Text 元素，另外你也可以使用 `createAnimatableComponent` 制作其他元素的动画，你可以查看 [文档][animatable] 了解更多细节。

```jsx | pure
MyCustomComponent = Animatable.createAnimatableComponent(MyCustomComponent);
```

**Props：**

- **animation**：它接受一个字符串，用来指定我们想要的动画类型，例如：`SlideInDown`、`SlideInLeft` 请查看 [docs][animatable] 来探索你可以使用的动画类型。
- **iteractionCount**：它指定了动画应该运行和迭代的次数。我们可以使用 `infinite` 来使动画永远运行。
- **direction**：指定动画的方向，可以使用 `normal`,`alternate-reverse`,`reverse`。

我们也可以使用其他属性，比如 `transition` 允许我们定义我们想要动画化的样式属性，如下图所示：

```jsx | pure
import React, {Component} from 'react';
import {Text, Dimensions, SafeAreaView, StyleSheet} from 'react-native';
import * as Animatable from 'react-native-animatable';

class AnimatableScreen extends Component {
  state = {
    opacity: 0,
  };

  componentDidMount() {
    this.setState({opacity: 1});
  }

  render() {
    return (
      <SafeAreaView style={styles.container}>
        <Animatable.View
          style={[
            styles.card,
            {
              opacity: this.state.opacity,
            },
          ]}
          iterationCount="infinite"
          direction="reverse"
          duration={3000}
          transition="opacity">
          <Text style={styles.whiteText}>Transition Animation</Text>
        </Animatable.View>
      </SafeAreaView>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    alignItems: 'center',
    width: Dimensions.get('screen').width,
    height: Dimensions.get('screen').height,
    justifyContent: 'center',
  },
  card: {
    width: Dimensions.get('screen').width * 0.6,
    height: Dimensions.get('screen').height * 0.35,
    backgroundColor: '#206225',
    alignItems: 'center',
    justifyContent: 'center',
    borderRadius: 8,
  },
  whiteText: {
    color: '#ffffff',
    fontSize: 18,
  },
});

export default AnimatableScreen;
```

[react-native-animatable][animatable] 给我们提供了更多的选项，使我们在动画中拥有更多的控制权，关于这个库还有更多的东西要讲，一篇文章无法涵盖，我真的推荐你去看看文档。

## Lottie

[lottie]: https://airbnb.io/lottie/#/
[lottie-react-native]: http://lottie-react-native/


![1_audbt0qXd6H5RQl7ebkgeQ.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc00afe5e9ef421aa00ca61a5e7b5e8c~tplv-k3u1fbpfcp-watermark.image)

这是迄今为止我最喜欢的库，Lottie 是一个由 Airbnb 创建的动画库，它将 After Effect 动画解析成 JSON 文件，你可以将其导出并在你的应用程序中使用，你可以在 ios、Android、Windows、Web 和 React Native 上使用 Lottie。

在 React Native 中 使用 [Lottie][lottie] 你需要安装适配插件 [lottie-react-native][lottie-react-native]，它是一个简单的接收 JSON 文件的封装元素：

```jsx | pure
import React, {Component} from 'react';
import {StyleSheet, SafeAreaView} from 'react-native';
import LottieView from 'lottie-react-native';

export default class index extends Component {
  render() {
    return (
      <SafeAreaView style={styles.container}>
        <LottieView
          source={require('./animation.json')}
          loop
          autoPlay
          style={{width: 100}}
        />
      </SafeAreaView>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    alignItems: 'center',
  },
});
```

你可以查看 https://lottiefiles.com/，获得免费的 After Effect 动画的 JSON 文件与 Lottie 一起使用，享受不一样的动画体验😎。

## react-native-reanimated

[reanimated]: https://github.com/kmagiera/react-native-reanimated
[animations]: https://reactnative.cn/docs/animations
[william]: https://medium.com/@wcandillon
[youtube]: https://www.youtube.com/channel/UC806fwFWpiLQV5y-qifzHnA

[react-native-reanimated][reanimated] 的作者从零重写了 React Native 的 Animated API，正如作者在 [官方文档][reanimated] 中描述的，react-native-reanimated 为以它基础的 Animated 库的 API 提供了更全面、更底层的抽象，因此在编写动画时可以有更大的灵活性，尤其是在涉及到基于手势的交互时。

react-native-reanimated 提供了一个新的 Animated API 来让你替代 [React Native Animated API][animations]，你可以用它来代替 [React Native Animated API][animations] 在 React native 中创建动画。以下是为什么你可能必须使用 [react-native-reanimated][reanimated] 而不是 [React Native Animated API][animations]：

- react-native-reanimated 通过提供多个声明性 API，让你对动画值有更大的控制权，以保持对值变化的跟踪。
- 不再需要使用 `useNativeDriver`，因为 react-native-reanimated 直接在 UI 线程执行动画。

了解更多关系该库的信息，我建议你阅读他们托管在 [GitHub 的文档][reanimated]，我也建议你关注 [William Candillon][william] 和它的 [youtube channel][youtube]，他在 React Native 中做了一些很棒的动画，而且他一直在使用 react-native-reanimated。

## React Native Animations Library (rnal)

[saidhayani@]: https://saidhayani.medium.com/
[rnal]: https://github.com/hayanisaid/rnal

![1_BLFTzAhn1uV5OLtHFUQJQg.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1772cb10093d489b9357f6c5c08f431e~tplv-k3u1fbpfcp-watermark.image)

`rnal` , 是由 [SaidHayani@][saidhayani@] 创建的 React Native 动画库，它的目的是使 React Native 中使用动画变得足够简单，通过提供简单的封装来创建过渡效果，如 `Fade`，`Scale`，或 `rotation` 效果，并可选择创建自定义动画，要创建 `Fade` 效果，你可以只做以下事情：

![1_zuqjwJjTqPUMXAoOYGfYIw.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0259be58f06642e592bd39c55f263bc2~tplv-k3u1fbpfcp-watermark.image)

你可以查看 [docs][rnal] 探索更多你可以使用的选项.

## react-native-motion

[motion]: https://github.com/xotahal/react-native-motion

[react-native-motion][motion] 是一个在 React Native 中制作动画的库，使用起来非常简单，下面是一个使用 [react-native-motion][motion] 制作简单 Shake 动画的例子：

```jsx | pure
import React, {Component} from 'react';
import {Text, StyleSheet, TouchableOpacity, SafeAreaView} from 'react-native';
import {Shake} from 'react-native-motion';

export default class index extends Component {
  state = {
    value: 0,
  };
  _startAnimation = () => {
    this.setState({value: this.state.value + 1});
  };
  render() {
    return (
      <SafeAreaView style={styles.container}>
        <TouchableOpacity style={styles.btn} onPress={this._startAnimation}>
          <Text style={styles.textBtn}>Start animation 🚀</Text>
        </TouchableOpacity>
        <Shake style={[styles.card]} value={this.state.value} type="timing">
          <Text style={styles.whiteText}>{this.state.value}</Text>
        </Shake>
      </SafeAreaView>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    alignItems: 'center',
  },
  btn: {
    alignItems: 'center',
    backgroundColor: '#d3d3d3',
    padding: 10,
    width: '80%',
    borderRadius: 10,
    marginTop: 20,
  },
  textBtn: {
    fontWeight: 'bold',
  },
  card: {
    backgroundColor: '#007fff',
    width: '60%',
    height: 300,
    marginTop: 100,
    justifyContent: 'center',
    alignItems: 'center',
    borderRadius: 10,
  },
  whiteText: {
    color: '#ffffff',
  },
});
```

![1_nhKMy9zlUPvCYWWbiGNFaQ.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15b7fc60d48d46ed94457eda63d5aa5d~tplv-k3u1fbpfcp-watermark.image)

react-native-motion 为我们提供了一个简单的 API 来制作共享的过渡效果，该库的作者写了 [一篇关于共享过渡效果的文章](https://medium.com/react-native-motion/transition-challenge-9bc9fdef56c7)。值得注意的是。

> 译者 Tips：该库已有 3 年未更新，且没有 TypeScript 类型声明，生产环境慎用！

> 原文链接：[Top 5 Animation Libraries in React Native](https://blog.bitsrc.io/top-5-animation-libraries-in-react-native-d00ec8ddfc8d)
> 原文作者：[SaidHayani@][saidhayani@]
> 译文出自：[紫升翻译计划](https://blog.zisheng.pro/categories/%E6%B4%9B%E7%AB%B9%E7%BF%BB%E8%AF%91%E8%AE%A1%E5%88%92/)
