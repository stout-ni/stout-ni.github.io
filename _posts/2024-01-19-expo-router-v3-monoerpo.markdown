---
layout: post
title: "在 monorepo 中使用 Expo Router V3"
date: 2024-01-19 07:04:41 +0800
categories: Expo ReactNative
---

## 前言

最近在翻看 Expo 文档的时候发现官方更新了 SDK 50 和 Expo Router V3 版本，遂上手尝试了一下，但是发现了一系列的报错，发现是由于 Expo 默认不支持 Monorepo 引起的。其实解决方案很简单，但搜索了一下网上都没有什么结果，在这里简单分享一下。

## TL;DR

> 示例代码参考：[expo-router-monorepo-example](https://github.com/stout-ni/expo-router-monorepo-example)

参考[官网文档](https://docs.expo.dev/guides/monorepos/#create-our-first-app)一直操作到添加完`metro.config.js`文件。

接着如下添加`index.js`文件：

```js
import "expo-router/entry";
```

其余步骤依旧参照官方文档。完成后保存并重新运行即可。

## 排查思路

首先看一开始页面上的报错毫无头绪，到具体的 node_modules 找`@expo/cli`这个包也看不到`package.json`里导出了 src 这个源文件目录。

````plain
TypeError: ExpoResponse is not a constructor
    at getHtml (node_modules/@expo/cli/src/start/server/metro/createServerRouteMiddleware.ts:107:20)
    at runNextTicks (node:internal/process/task_queues:60:5)
    at processTimers (node:internal/timers:511:9)
    at handler (node_modules/@expo/server/src/index.ts:149:26)
    at node_modules/@expo/server/src/vendor/http.ts:36:24
    ```
````

其实页面上的报错只是一个障眼法，具体的错误可以在命令行里看到：

> Metro error: Unable to resolve module ./node_modules/expo-router/entry from apps/mobile/.:

结合稍微看两眼文档就知道这是由于 Expo 默认不兼容 Monorepo 引起的路径问题。但如果沿着 Expo 的官方文档一步步走下去，到了添加`index.js`这一步就有问题了，因为官方文档使用的不是 Expo Router 版本的项目，没有`App.[jt]sx`文件，只有一个`app`目录。

因为很明显可以知道`registerRootComponent`是一个用于指定根组件的函数，因此这个时候的思路就在于，怎么去找到 Expo Router 里的入口文件，也就是此时该从哪里去 import 这个目前不存在的 App。

然后我们就可以去看一看官网让我们替换`package.json`中原来的 main 入口，也就是`expo-router/entry`这个东西。直接去 node_modules 里就能找到这个文件。

```js
// `@expo/metro-runtime` MUST be the first import to ensure Fast Refresh works
// on web.
import "@expo/metro-runtime";

import { App } from "expo-router/build/qualified-entry";
import { renderRootComponent } from "expo-router/build/renderRootComponent";

// This file should only import and register the root. No components or exports
// should be added here.
renderRootComponent(App);
```

可以看到这里的逻辑只有一个，就是调用`renderRootComponent`这个方法，它的参数同样是一个 App 入口。那么问题就真相大白了：对于非 Monorepo 的情况来说，入口文件路径是固定的，也就是这里引入的 App。但是在 Monorepo 里，由于依赖都安装在一个 node_modules 中，项目的 node_modules 中找不到这个入口，于是就只能我们通过手动添加一个入口的方式去注册这个根组件。

那么最简单的解决方式就是直接在`index.js`里手动引入一下这个`expo-router/entry`，让所有的路径解析都让 Monorepo 的包管理工具（yarn 或者 pnpm）去解决，我们则不需要关心。
