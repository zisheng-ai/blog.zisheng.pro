---
title: Antd 快速上手教程
description: 以笔者的经验来看，Ant Design 设计体系下的产品设计理念、使用方式、底层技术、周边工具都保持着高度一致，工具不是越多越好，有一套好用顺手的就行，UI 框架千千万，你不可能都学一遍。Ant Design 无疑能够减少你的学习成本。
cover: https://i.loli.net/2020/03/04/hK2MgsYGvuayp7Q.png
date: 2020-03-04 04:17:10
categories:
  - [前端, React]
tags:
  - Ant Design
  - UI 框架
  - Ant Design
  - babel-plugin-import
  - babel
---

`antd` 是基于 Ant Design 设计体系的 React UI 组件库，主要用于研发企业级中后台产品。

> 文章可参考源码：[antd-with-ts-demo](https://github.com/youngjuning/antd-with-ts-demo)

## Ant Design 设计体系

以笔者的经验来看，Ant Design 设计体系下的产品设计理念、使用方式、底层技术、周边工具都保持着高度一致，工具不是越多越好，有一套好用顺手的就行，UI 框架千千万，你不可能都学一遍。Ant Design 无疑能够减少你的学习成本。

- 设计
  - 设计价值观
  - 全局样式
  - 设计模式
- 组件库
  - [Ant Design of React](https://ant.design/docs/react/introduce-cn): 基于 Ant Design 设计体系的 React UI 组件库，主要用于研发企业级中后台产品。
  - [Ant Design Mobile of React](https://mobile.ant.design/docs/react/introduce-cn): `antd-mobile` 是 Ant Design 的移动规范的 React 实现，服务于蚂蚁及口碑无线业务。
  - [Ant Design Mobile RN of React](https://rn.mobile.ant.design/docs/react/introduce-cn): `@ant-design/react-native` 是 [Ant Design](http://ant.design/) 的移动规范的 React 实现，服务于蚂蚁及口碑无线业务。
  - [Ant Design of Angular](https://ng.ant.design/docs/introduce/zh): 这里是 Ant Design 的 Angular 实现，开发和服务于企业级后台产品。
  - [Ant Design Mobile of Angular](https://ng.mobile.ant.design/#/docs/introduce/zh): 这里是 **Ant Design** 移动规范的 **Angular** 实现，服务于阿里巴巴集团数据无线业务。
  - [Ant Design of Vue](https://www.antdv.com/docs/vue/introduce-cn/): 这里是 Ant Design 的 Vue 实现，开发和服务于企业级后台产品。
- [Icons](https://ant.design/components/icon-cn/): 一整套优质的图标集
- [AntV](https://antv.vision/zh): AntV 是蚂蚁金服全新一代数据可视化解决方案，致力于提供一套简单方便、专业可靠、无限可能的数据可视化最佳实践。
- [Ant Design Pro](https://pro.ant.design/index-cn): 开箱即用的中台前端/设计解决方案
  - [dva](http://dvajs.com/): 一个基于 Redux 的 轻量级数据流方案，概念来自 elm，支持 side effects、热替换、动态加载、react-native、SSR 等，已在生产环境广泛应用。
  - [umi](http://umijs.org/) : 一个可插拔的企业级 react 应用框架。umi 以路由为基础的，支持[类 next.js 的约定式路由](https://umijs.org/zh/guide/router.html)，以及各种进阶的路由功能，并以此进行功能扩展，比如[支持路由级的按需加载](https://umijs.org/zh/plugin/umi-plugin-react.html#dynamicimport)。然后配以完善的[插件体系](https://umijs.org/zh/plugin/)，覆盖从源码到构建产物的每个生命周期，支持各种功能扩展和业务需求，同时提供 [Umi UI](https://umijs.org/zh/guide/umi-ui.html) 通过可视化辅助编程（VAP）提高开发体验和研发效率。

从上面的体系中可以看出，Ant Design of React 可以说是整个 Ant Design 设计体系的核心产品，想要学习 Ant Design Pro，首先就要先熟悉 Ant Design of React。

## 流行趋势

### npm 下载量

如果拿 antd 和 element-ui、iview 这些老牌 Vue.js UI 框架对比，遥遥领先啊有没有：

![紫升](https://user-gold-cdn.xitu.io/2020/3/4/170a4939e1919973?w=1103&h=458&f=png&s=59582)

如果拿 ant-design-vue 来和 element-ui、iview这些老牌 vue UI框架对比，也是很有竞争力的：

![紫升](https://user-gold-cdn.xitu.io/2020/3/4/170a4939e21c45c6?w=1115&h=454&f=png&s=76675)

### GitHub Star

![紫升](https://user-gold-cdn.xitu.io/2020/3/4/170a4939e23e1d0b?w=1119&h=207&f=png&s=36028)

## 特性

- 🌈 提炼自企业级中后台产品的交互语言和视觉风格。
- 📦 开箱即用的高质量 React 组件。
- 🛡 使用 TypeScript 开发，提供完整的类型定义文件。
- ⚙️ 全链路开发和设计工具体系。
- 🌍 数十个国际化语言支持。
- 🎨 深入每个细节的主题定制能力。

## 支持环境

- 现代浏览器和 IE11 及以上（需要 [polyfills](https://ant.design/docs/react/getting-started-cn#兼容性)）。
- 支持服务端渲染。
  - [Next.js](https://nextjs.frontendx.cn/): **Next.js** 是一个轻量级的 React 服务端渲染应用框架。
- [Electron](https://electronjs.org/)：使用 JavaScript，HTML 和 CSS 构建跨平台的桌面应用程序
  - [electron-react-boilerplate](https://github.com/electron-react-boilerplate/electron-react-boilerplate): 可扩展的跨平台应用程序的基础

## 安装

```shell
$ yarn add antd
```

## 按需加载

Antd 系列的 UI 组件库都需要引入 [babel-plugin-import](https://github.com/ant-design/babel-plugin-import) 库来实现懒加载

```js
// .babelrc or babel-loader option
{
  "plugins": [
    ["import", {
      "libraryName": "antd",
      "libraryDirectory": "es",
      "style": "css" // `style: true` 会加载 less 文件
    }]
  ]
}
```

然后只需从 antd 引入模块即可，无需单独引入样式。等同于下面手动引入的方式。

```jsx
// babel-plugin-import 会帮助你加载 JS 和 CSS
import { DatePicker } from 'antd'
```

## 快速上手

> 在开始之前，推荐先学习 [React](http://reactjs.org/) 和 [ES2015](http://babeljs.io/docs/learn-es2015/)，并正确安装和配置了 [Node.js](https://nodejs.org/) v8 或以上。官方指南假设你已了解关于 HTML、CSS 和 JavaScript 的中级知识，并且已经完全掌握了 React 全家桶的正确开发方式。如果你刚开始学习前端或者 React，将 UI 框架作为你的第一步可能不是最好的主意。

### 1. 创建一个 codesanbox

访问 http://u.ant.design/codesandbox-repro 创建一个 codesandbox 的在线示例，别忘了保存以创建一个新的实例。

### 2. 使用组件

直接用下面的代码替换 `index.js` 的内容，用 React 的方式直接使用 antd 组件。

```tsx
import React from 'react';
import ReactDOM from 'react-dom';
import { ConfigProvider, DatePicker, message } from 'antd';
// 由于 antd 组件的默认文案是英文，所以需要修改为中文
import zhCN from 'antd/es/locale/zh_CN';
import moment from 'moment';
import 'moment/locale/zh-cn';
import 'antd/dist/antd.css';
import './index.css';

moment.locale('zh-cn');

class App extends React.Component {
  state = {
    date: null,
  };

  handleChange = date => {
    message.info(`您选择的日期是: ${date ? date.format('YYYY-MM-DD') : '未选择'}`);
    this.setState({ date });
  };
  render() {
    const { date } = this.state;
    return (
      <ConfigProvider locale={zhCN}>
        <div style={{ width: 400, margin: '100px auto' }}>
          <DatePicker onChange={this.handleChange} />
          <div style={{ marginTop: 20 }}>
            当前日期：{date ? date.format('YYYY-MM-DD') : '未选择'}
          </div>
        </div>
      </ConfigProvider>
    );
  }
}

ReactDOM.render(<App />, document.getElementById('root'));
```

### 3. 探索更多组件用法

你可以在左侧菜单查看组件列表，比如 [Alert](https://ant.design/components/alert-cn/) 组件，组件文档中提供了各类演示，最下方有组件 API 文档可以查阅。在代码演示部分找到第一个例子，点击右下角的图标展开代码。

然后依照演示代码的写法，在之前的 codesandbox 里修改 `index.js`，首先在 `import` 内引入 Alert 组件：

```diff
- import { ConfigProvider, DatePicker, message } from 'antd';
+ import { ConfigProvider, DatePicker, message, Alert } from 'antd';
```

然后在 `render` 内添加相应的 jsx 代码：

```diff
  <DatePicker onChange={value => this.handleChange(value)} />
  <div style={{ marginTop: 20 }}>
-   当前日期：{date ? date.format('YYYY-MM-DD') : '未选择'}
+   <Alert message={`当前日期：${date ? date.format('YYYY-MM-DD') : '未选择'}`} type="success" />
  </div>
```

好的，现在你已经会使用基本的 antd 组件了，你可以在这个例子中继续探索其他组件的用法。如果你遇到组件的 bug，也推荐建一个可重现的 codesandbox 来报告 bug。

### 4. 下一步[#](https://ant.design/docs/react/getting-started-cn#4.-下一步)

实际项目开发中，你会需要构建、调试、代理、打包部署等一系列工程化的需求。您可以阅读后面的文档或者使用以下脚手架和范例：

- [Ant Design Pro](http://pro.ant.design/)
- [antd-admin](https://github.com/zuiidea/antd-admin)
- [d2-admin](https://github.com/d2-projects/d2-admin)
- 更多脚手架可以查看 [脚手架市场](http://scaffold.ant.design/)

## 使用 Day.js 替换 momentjs 优化打包大小

你可以使用 [antd-dayjs-webpack-plugin](https://github.com/ant-design/antd-dayjs-webpack-plugin) 插件用 Day.js 替换 momentjs 来大幅减小打包大小。这需要更新 webpack 的配置文件如下：

```js
// webpack-config.js
import AntdDayjsWebpackPlugin from 'antd-dayjs-webpack-plugin';

module.exports = {
  // ...
  plugins: [new AntdDayjsWebpackPlugin()],
};
```

## 在 TypeScript 中使用

使用 `create-react-app` 一步步地创建一个 TypeScript 项目，并引入 antd。

### 安装和初始化

创建 [cra-template-typescript](https://github.com/facebook/create-react-app/tree/master/packages/cra-template-typescript) 项目：

```shell
$ npx create-react-app my-app --template typescript
```

然后我们进入项目并启动。

```shell
$ cd antd-demo-ts
$ yarn start
```

此时浏览器会访问 http://localhost:3000/ ，看到 `Welcome to React` 的界面就算成功了。

### 引入 antd

```shell
$ yarn add antd
```

### 自定义 create-react-app 配置

我们需要对 create-react-app 的默认配置进行自定义，这里我们使用 [react-app-rewired](https://github.com/timarney/react-app-rewired) （一个对 create-react-app 进行自定义配置的社区解决方案）。

引入 react-app-rewired 并修改 package.json 里的启动配置。由于新的 [react-app-rewired@2.x](https://github.com/timarney/react-app-rewired#alternatives) 版本的关系，你还需要安装 [customize-cra](https://github.com/arackaf/customize-cra)。

```shell
$ yarn add react-app-rewired customize-cra -D
```

```diff
/* package.json */
"scripts": {
-   "start": "react-scripts start",
+   "start": "react-app-rewired start",
-   "build": "react-scripts build",
+   "build": "react-app-rewired build",
-   "test": "react-scripts test",
+   "test": "react-app-rewired test",
}
```

在项目根目录创建一个 `config-overrides.js` 用于修改默认配置:

```js
module.exports = function override(config, env) {
  // do stuff with the webpack config...
  return config;
};
```

### 使用 babel-plugin-import

[babel-plugin-import](https://github.com/ant-design/babel-plugin-import) 是一个用于按需加载组件代码和样式的 babel 插件（[原理](https://ant.design/docs/react/getting-started-cn#按需加载)），现在我们尝试安装它并修改 `config-overrides.js` 文件。

```
$ yarn add babel-plugin-import -D
```

替换 `config-overrides.js` 文件内容：

```js
const { override, fixBabelImports } = require('customize-cra');

module.exports = override(
  fixBabelImports('import', {
    libraryName: 'antd',
    libraryDirectory: 'es',
    style: 'css', // `style: true` 会加载 less 文件
  }),
);
```

### 使用 antd

```tsx
// src/App.tsxe
import React from 'react';
import { Button } from 'antd';
import './App.css';

function App() {
  return (
    <div className="App">
      <Button type="primary">Button</Button>
    </div>
  );
}

export default App;
```

运行 `yarn start` 访问页面，antd 组件的 js 和 css 代码都会按需加载，你在控制台也不会看到这样的[警告信息](https://zos.alipayobjects.com/rmsportal/vgcHJRVZFmPjAawwVoXK.png)。关于按需加载的原理和其他方式可以阅读[这里](https://ant.design/docs/react/getting-started-cn#按需加载)。

### 自定义主题

按照 [配置主题](https://ant.design/docs/react/customize-theme-cn) 的要求，自定义主题需要用到 less 变量覆盖功能。我们可以引入 `customize-cra` 中提供的 less 相关的函数 [addLessLoader](https://github.com/arackaf/customize-cra#addlessloaderloaderoptions) 来帮助加载 less 样式，同时修改 `config-overrides.js` 文件如下。

```shell
$ yarn add less less-loader -D
```

```diff
- const { override, fixBabelImports } = require('customize-cra');
+ const { override, fixBabelImports, addLessLoader } = require('customize-cra');

module.exports = override(
  fixBabelImports('import', {
    libraryName: 'antd',
    libraryDirectory: 'es',
-   style: 'css',
+   style: true,
  }),
+ addLessLoader({
+   javascriptEnabled: true,
+   modifyVars: { '@primary-color': '#1DA57A' },
+ }),
);
```

这里利用了 [less-loader](https://github.com/webpack/less-loader#less-options) 的 `modifyVars` 来进行主题配置，变量和其他配置方式可以参考 [配置主题](https://ant.design/docs/react/customize-theme-cn) 文档。

修改后重启 `yarn start`，如果看到一个绿色的按钮就说明配置成功了。

### 使用 Day.js 替换 momentjs 优化打包大小

```diff
+ const AntdDayjsWebpackPlugin = require('antd-dayjs-webpack-plugin');
- const { override, fixBabelImports, addLessLoader } = require('customize-cra');
+ const { override, fixBabelImports, addLessLoader, addWebpackPlugin } = require('customize-cra');

module.exports = override(
  fixBabelImports('import', {
    libraryName: 'antd',
    libraryDirectory: 'es',
    style: true,
  }),
  addLessLoader({
    javascriptEnabled: true,
    modifyVars: { '@primary-color': '#1DA57A' },
  }),
+  addWebpackPlugin(new AntdDayjsWebpackPlugin()),
);

```

### decorators

```shell
$ yarn add -D @babel/plugin-proposal-decorators
```

```js
const { addDecoratorsLegacy } = require('customize-cra');
module.exports = override(
	...
  addDecoratorsLegacy(),
  ...
);
```

### 配置 Babel 插件

```
module.exports = override(
  ...,
  ...addBabelPresets(
    [
      "@babel/preset-env",
      {
        targets: {
          browsers: ["> 1%", "last 2 versions"],
          ie: 9
        },
      }
    ]
  )
  ...
);
```

### 允许使用 .babelrc.js 文件进行Babel配置。

```js
// config-overrides.js
const { useBabelRc } = require('customize-cra')

module.exports = override(
  ...
  // 允许使用 .babelrc.js 文件进行Babel配置。
  useBabelRc()
  ...
);

```

```shell
$ yarn add @babel/preset-env -D
```

```js
// .babelrc.js
module.exports = {
  presets: [
    [
      "@babel/preset-env", //兼容ie9
      {
        targets: {
          ie: "9"
        }
      }
    ]
  ],
  plugins: ["@babel/plugin-proposal-decorators"] // 可以用来替换 addDecoratorsLegacy
}
```
