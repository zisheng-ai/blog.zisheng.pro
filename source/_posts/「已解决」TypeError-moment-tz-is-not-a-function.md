---
title: '「已解决」TypeError: moment().tz is not a function'
date: 2023-03-21 12:41:19
description: moment().tz is not a function 通常不是时区格式问题，而是 moment-timezone 没有被正确加载。本文给出可复现的导入方式、排查步骤和替代方案。
categories:
  - [前端, JavaScript]
  - 问题排查
tags:
  - JavaScript
  - moment-timezone
  - 时区
  - TypeError
---

下面这行代码报错时，问题通常不在 `timeZone` 的值，而在 `tz` 这个方法根本没有挂到当前的 `moment` 实例上：

```ts
moment().tz(vm.timeZone).format('YYYY-MM-DD')
```

## 一句话总结

`tz()` 来自 `moment-timezone`，不是 `moment` 本体 API。直接从 `moment-timezone` 导入，最能避免 bundler、默认导入和重复依赖导致的实例不一致。

## 推荐写法

先安装依赖：

```bash
pnpm add moment-timezone
```

然后直接导入：

```ts
import moment from 'moment-timezone'

const date = moment().tz('Asia/Shanghai').format('YYYY-MM-DD')
```

CommonJS 项目则可以使用：

```js
const moment = require('moment-timezone')
```

## 为什么 `import 'moment-timezone'` 有时不够

旧项目常见下面的副作用导入：

```ts
import moment from 'moment'
import 'moment-timezone'
```

它依赖 `moment-timezone` 修改同一份 `moment` 实例。在 monorepo、alias 配置或 lockfile 出现多个 `moment` 副本时，副作用可能修改的是 A 实例，而业务代码拿到的是 B 实例，于是仍然报 `tz is not a function`。直接从 `moment-timezone` 导入可避免这层不确定性。

## 排查清单

1. 运行 `pnpm why moment` 或对应包管理器命令，确认没有意外安装多份 `moment`。
2. 检查 `moment-timezone` 是否在生产依赖中，而不是只在 `devDependencies`。
3. 检查 bundler alias 是否把 `moment` 指到不同路径。
4. 确认时区名是 IANA 格式，例如 `Asia/Shanghai`，而不是 `GMT+8` 这类展示字符串。

## 需要新代码时的选择

Moment 已进入维护模式。新项目如果只需时区转换，可以评估 `date-fns-tz`、Luxon 或原生 `Intl.DateTimeFormat`；但在已有 Moment 代码库里，局部替换的成本未必值得。先把导入链路修正确，再决定是否迁移。

## 我的判断框架

遗留项目：继续使用 `moment-timezone` 并统一入口导入，风险最低。

新项目：优先选择现代日期库或 `Intl`，避免新增 Moment 依赖。
