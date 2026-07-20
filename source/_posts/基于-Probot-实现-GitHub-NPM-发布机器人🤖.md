---
title: 基于 Probot 实现 GitHub NPM 发布机器人🤖
date: 2023-03-08 17:21:07
description: GitHub Apps 是 GitHub 中优秀的产品。一个 GitHub App 通过 API 直接使用自己的身份进行操作，这意味着你不需要作为一个单独的用户维护一个机器人或服务账户。
categories:
  - 运维
tags:
  - GitHub
  - npm
  - Probot
  - 机器人
---

## 关于 Github Apps

GitHub Apps 是 GitHub 中优秀的产品。一个 GitHub App 通过 API 直接使用自己的身份进行操作，这意味着你不需要作为一个单独的用户维护一个机器人或服务账户。

GitHub Apps 可以被直接安装到组织或者用户账号上，并且可以赋予它们访问指定仓库的权限。它们带有内置的 webhook 和狭窄的特定权限。设置 GitHub 应用程序时，可以选择希望它访问的仓库。比如你可以设置一个叫 `MyGitHub` 的应用程序，该应用程序有且仅有 `octocat` 仓库的写入 `issues` 的权限。安装 GitHub App 需要你是组织的所有者或对仓库拥有 admin 权限。

## 关于 Probot

Probot 是一个基于 Node.js 构建 GitHub Apps 的框架。它旨在消除所有繁琐的事情，如接收和验证 webhooks 以及做认证动作，这样你就可以专注于你想要实现的功能。Probot 应用程序非常容易编写、部署和分享。大多数流行的 Probot 应用被托管，所以你不需要做任何部署和管理工作。这里有几个基于 Probot 构建的应用：

- [ESLint Disable Watcher][eslint-disable] - 当 pr 中尝试禁用 eslint 规则时进行评论。
- [Developer Certificate of Origin][dco] - 强制 Pull Requests 符合 DCO 规范
- [Issue Check][issue-check] - 根据关键字和任务列表检查 issues 以确保包含必要的信息
- [Pull][pull] - Keep your forks up-to-date.

你可以在[特色应用][featured-apps]或[GitHub probot-app 话题][github-probot-app-topics]浏览更多应用。

## Hello Probot

一个 Probot 应用本质上是导出一个函数的 [Node.js 模块][node-modules]：

```ts
module.exports = app => {
  // your code here
};
```

`app` 参数是 [Probot][probot-class] 类的实例，该实例可以让你访问所有的 GitHub 精华。

`aap.on` 负责监听所有 [GitHub 触发的 webhook 事件][webhooks]，当 GitHub 上发生任何有趣的事情，你的应用程序想知道的时候，它会通知你。

```ts
module.exports = app => {
  app.on('issues.opened', async context => {
    // 一个新的 issue 被打开，我们应该做些什么呢？
    context.log.info(context.payload);
  });
};
```

传递给事件处理程序的上下文包括关于被触发的事件的一切，以及一些有用的属性，以便对事件做出有用的回应。`context.octokit` 是一个经过认证的 GitHub 客户端，可以用来[进行 REST API 和 GraphQL 调用][github-api]，并允许你以编程方式做几乎任何你可以在 GitHub 上通过浏览器做的事情。

下面是一个当新打开 issues 时自动评论的 App。

```ts
module.exports = app => {
  app.on('issues.opened', async context => {
    // context` 从事件中提取信息，可以传递给 GitHub API 调用。这将返回：
    // { owner: 'yourname', repo: 'yourrepo', number: 123, body: 'Hello World !}
    const params = context.issue({ body: 'Hello World!' });

    // 在 issue 上发一条评论
    return context.octokit.issues.createComment(params);
  });
};
```

## 开发一个 Probot app

为了开发一个 Probot app，你首先需要安装 Node.js 10.0.0 或更新的版本。

### 生成一个新的 app

[create-probot-app][create-probot-app] 是开始构建一个新的 app 的最佳方式。它将生成一个新的应用程序，其中包含你所需要的一切，以开始并在生产中运行你的应用程序。

运行下面的命令生成一个项目：

```sh
$ npx create-probot-app my-first-app
```

该命令会问一系列关于你的 app 的问题，看起来就像：

```sh
Let's create a Probot app!
? App name: my-first-app
? Description of app: A 'Hello World' GitHub App built with Probot.
? Author's full name: Katie Horne
? Author's email address: katie@auth0.com
? GitHub user or org name: khorne3
? Repository name: my-first-app
? Which template would you like to use? (Use arrow keys)
❯ basic-js
  basic-ts (use this one for TypeScript support)
  checks-js
  git-data-js
  deploy-js

Finished scaffolding files!

Installing dependencies. This may take a few minutes...

Successfully created my-first-app.

Begin using your app with:
  cd my-first-app
  npm start

View your app's README for more usage instructions.

Visit the Probot docs:
  https://probot.github.io/docs/

Get help from the community:
  https://probot.github.io/community/

Enjoy building your Probot app!
```

创建的最重要的文件是 `index.js`（你的 app 代码所在的位置）和 `package.json`（使你的 app 成为标准 [npm module][npm-module]）。

### 本地运行 app

现在你已经准备好在本地运行 app 了。运行 `npm start` 来开启一个 server 吧：

> 注意：如果你选择了 TypeScript 模板，请确保执行了 `npm run build`！

```sh
$ yarn start
yarn run v1.22.10
$ probot run ./lib/index.js
INFO (server): Running Probot v11.3.0 (Node.js: v14.15.5)
INFO (server): Forwarding https://smee.io/dz7D1zur24cGNj7 to http://localhost:3000/
INFO (server): Listening on http://localhost:3000
INFO (server): Connected
```

### 配置 GitHub App

下列是自动配置 GitHub App 的步骤：

1. 在本地命令行中执行 `npm start`。
2. 访问 http://localhost:3000 查看下一步。
3. 你会看到类似下面的页面。

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e1601ae833f47a1815d79cb1dea2e6f~tplv-k3u1fbpfcp-zoom-1.image)

4. 点击 **Register a GitHub App** 按钮继续。
5. 接着，你需要给你的 App 取一个没有被占用的名字，注意：如果你看到类似 `Name is reserved for the account @tuya` 的提示，这意味着你不能使用已存在的 GitHub organization 的名字作为 app 的名字（除非你是该组织的 owner）

## GitHub Release 时 `npm publish`

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5e7c791feb140cbb537a79c89450842~tplv-k3u1fbpfcp-zoom-1.image)

实现 GitHub CI 自动发布 NPM 包，主要是为了合理管理对外 npm 发布权限。而比较通用的发布时机是在 GitHub release 时。基于上面流程图的分析，我们可以看出 released 状态时执行 `npm publish` 最合适。

我们实现的具体逻辑是，当 Probot app 监听到 `release.released` 事件时，处理发布前的操作。重要的是我们需要根据 `package.json` 中的 `version` 字段匹配出 tag，比如：

- 1.0.0：tag 为 latest 的 1.0.0
- 1.0.0-beta.0：tag 为 beta 的 1.0.0-beta.0
- 1.0.0-alpha.0：tag 为 alpha 的 1.0.0-alpha

## NPM 自动发布实现原理

发布之前我们需要拉取仓库代码、取出版本和 tag、设置 NPM publish Token 等工作。先上核心代码，后面我们详细解析。

```js
app.on('release.released', async context => {
  if (!isTuya(context)) return;
  app.log('npm publishing');
  const { repository: repo } = context.payload;
  const downloadDefaultBranch = `${repo.full_name}#${context.payload.release.tag_name}`;
  const downLoadTempDir = `${os.tmpdir()}/${repo.full_name}`;
  await download(downloadDefaultBranch, downLoadTempDir);
  const { version, scripts } = require(`${downLoadTempDir}/package.json`);
  const tag = /^\d\.\d\.\d-(.*)\.\d$/.exec(version)
    ? /^\d\.\d\.\d-(.*)\.\d$/.exec(version)[1]
    : 'latest';
  // 如果有 build 脚本则先执行 build 脚本
  if (scripts.build) {
    await execSh(`cd ${downLoadTempDir} && npm install && npm run build`);
  }
  try {
    const result = await npmPublish({
      package: `${downLoadTempDir}/package.json`,
      token: process.env.NPM_AUTH_TOKEN,
      registry: 'https://registry.npmjs.org/',
      tag,
    });
    if (result.type === 'none') {
      app.log.error(
        `You can't publish duplicate version ${result.package}@${result.version}`,
      );
    }
  } catch (error) {
    app.log.error(error);
  }
});
```

### NPM Publish Token

#### 申请 NPM Publish Token

**1. 访问 npmjs.com 进入 Access Tokens 页面**

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e077e9f9d0a4dc8bde0a24e121461c3~tplv-k3u1fbpfcp-zoom-1.image)

**2. 点击 Generate New Token 按钮**

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82e09021f85c413d80ac77ed3df6746e~tplv-k3u1fbpfcp-zoom-1.image)

**3. Token 类型选择 Publish**

![紫升](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/256574d0e063410aa50c85d469346809~tplv-k3u1fbpfcp-zoom-1.image)

#### 保证 NPM Publish Token 安全性

NPM Token 是不能被别人看到的，为了达到这个目的，首先项目需要设置为私有的，然后将 Token 放到 `.env` 中，通过 `process.env.NPM_AUTH_TOKEN` 获取。另外谨记不要在日志中打印环境变量。

#### 保证 GitHub App 安全性

如果把 GitHub App 发布为 public 的，那么任何仓库都可以安装该应用，这不是我们想要的结果。解决办法有两个，一是将应用注册为 private 类型的，二是在监听回调中判断是否是允许的组织或者用户。我选择的是第二种方案，校验函数如下：

```js
const isTuya = context => {
  const { full_name } = context.payload.repository;
  return full_name.startsWith('youngjuning') || full_name.startsWith('tuya');
};
```

### 下载源码

我们选择了 download-git-repo 下载 git 仓库，但是该仓库不支持 Promise，我们做一下简单的改造：

```js
const download = require('download-git-repo');

module.exports = (repo, tempDir) => {
  return new Promise((resolve, reject) => {
    download(repo, tempDir, err => {
      if (err) {
        reject(err);
      } else {
        resolve(null);
      }
    });
  });
};
```

### npmPublish

我们选择了 @jsdevtools/npm-publish 执行发布动作，该仓库除了编程调用外，还可以作为 GitHub Action 和命令行工具使用。需要注意的是，我们需要用正则取出我们要发布的 tag：

```js
const tag = /^\d\.\d\.\d-(.*)\.\d$/.exec(version)
  ? /^\d\.\d\.\d-(.*)\.\d$/.exec(version)[1]
  : 'latest';
```

### lerna publish

lerna 管理版本由于是一次可能发布多个仓库，所以无法使用上面提到的发布流程。针对 lerna，我设计的发布流程是监听到 push 动作后取最新的一条 commit，匹配是否包含 `chore(release): publish`。具体原理如下：

1. 判断 push 分支是否是主分支且提交信息包含 `chore(release): publish`
2. 因为是 lerna publish，所以需要使用 simple-git 这个库 clone 项目。
3. 由于 lerna publish [不支持 token](https://github.com/lerna/lerna/issues/2404)，我们采用将 `//registry.npmjs.org/:_authToken=${process.env.NPM_AUTH_TOKEN}` 写入 `.npmrc` 的方式完成带 token 的发布。
4. 最后，我们需要使用 `from-git` 的方式执行 `lerna publish`，`from-git` 的场景便是本地执行 `lerna version`，在 CI 中执行 `lerna publish`。

完整代码如下：

```js
app.on('push', async context => {
  if (!isTuya(context)) return;
  if (
    context.payload.ref.indexOf(context.payload.repository.default_branch) !==
      -1 &&
    context.payload.head_commit.message.indexOf('chore(release): publish') === 0
  ) {
    app.log('push event');
    execSh(`git --version`);
    const { repository: repo } = context.payload;
    const cloneTempDir = `${os.tmpdir()}/${repo.full_name}`;
    try {
      await git.clone(repo.clone_url, cloneTempDir);
      const { devDependencies } = require(`${cloneTempDir}/package.json`);
      if (devDependencies['lerna']) {
        app.log('lerna publishing');
        await execSh(
          `cd ${cloneTempDir} && echo //registry.npmjs.org/:_authToken=${process.env.NPM_AUTH_TOKEN} > .npmrc`,
        );
        await execSh(
          `cd ${cloneTempDir} && npm install && npm run build && ./node_modules/.bin/lerna publish from-git --yes --no-verify-access`,
        );
      }
      await execSh(`rm -rf ${cloneTempDir}`);
    } catch (error) {
      await execSh(`rm -rf ${cloneTempDir}`);
    }
  }
});
```

## Glitch 部署

如果你有自己的服务器，可以直接将机器人程序部署到自己的服务器。我这里使用官方推荐的 Glitch 服务部署。Glitch 可以免费托管 node 应用并且直接在浏览器中编辑他们。对于简单的应用完全够了。

1. 注册并在 [Glitch](https://glitch.com/) 新建项目，选择 **Import from GitHub**，弹窗写上应用 github 地址，或者使用 https://github.com/behaviorbot/new-issue-welcome 作为模板导入后再将自己的代码复制过来。
2. 打开 `.env` 文件使用以下内容替代：

```
APP_ID=<your app id>
WEBHOOK_SECRET=<your app secret>
PRIVATE_KEY_PATH=<your private_key>
NODE_ENV=production
NPM_AUTH_TOKEN=3c2c104e-9f1f-4fc5-903e-726610b75ce1
INPUT_TOKEN=
```

4. 将 glitch 链接设置为 GitHub App 的 webhook 地址即可，之后更新代码，glitch 会自动更新部署。

## 权限

Probot App 的初始权限在 `app.yml` 文件中，如果 App 已经创建了，又想要更新权限，可以在 https://github.com/settings/apps 中更新。我所用的权限配置请点击 [app.yml](https://glitch.com/edit/#!/tuya-robot?path=app.yml%3A119%3A0) 查看。

[probot]: https://probot.github.io/
[github apps]: https://docs.github.com/en/developers/apps/about-apps
[web 挂钩事件和有效负载]: https://docs.github.com/cn/developers/webhooks-and-çevents/webhooks/webhook-events-and-payloads
[marketplace-getting-started]: https://docs.github.com/en/marketplace/getting-started
[eslint-disable]: https://probot.github.io/apps/eslint-disable/
[dco]: https://probot.github.io/apps/dco/
[issue-check]: https://probot.github.io/apps/issue-check/
[pull]: https://probot.github.io/apps/pull/
[featured-apps]: https://probot.github.io/apps/
[github-probot-app-topics]: https://github.com/search?q=topic%3Aprobot-app&type=Repositories
[node-modules]: https://nodejs.org/api/modules.html
[probot-class]: https://probot.github.io/api/latest/classes/probot.html
[webhooks]: https://probot.github.io/docs/webhooks/
[github-api]: https://probot.github.io/docs/github-api/
[create-probot-app]: https://github.com/probot/create-probot-app
[npm-module]: https://docs.npmjs.com/files/package.json

> 本文首发于「[紫升的博客](https://blog.zisheng.pro/)」，同步于公众号「[紫升早茶馆](https://cdn.jsdelivr.net/gh/youngjuning/images/20210418112129.jpeg)」和「[掘金专栏](https://juejin.cn/user/325111174662855)」。
