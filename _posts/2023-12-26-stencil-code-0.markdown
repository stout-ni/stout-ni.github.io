---
layout: post
title: "Stencil 源码学习（一）：项目结构与编译"
date: 2023-12-26 18:08:39 +0800
categories: 前端 Stencil
---

## 前言

> [Stencil](https://stenciljs.com/) is a library for building reusable, scalable component libraries. Generate small, blazing fast Web Components that run everywhere.

Stencil 是 Ionic 团队开源的库，以上是摘自 Stencil 官网的介绍。距离这个库开源也已经很多年了，但是网上没有看到什么讲解 Stencil 实现原理的文章，正好最近工作有遇到做跨框架组件库的内容，在技术选型上就决定用了 Stencil，正好也趁此机会深入学习一下这个库的源码和原理。第一篇文章就先从 Stencil 项目的整体开始。

## 项目概览

```sh
git clone https://github.com/ionic-team/stencil
```

一般来说从零开始上手一个仓库源码，通常我们最重要的第一件事就是看这个项目的入口。打开 `package.json` 文件，除了 npm 包名、版本等基本信息外，通过 main、module、types 我们可以得知项目的主入口在 `src/internal/stencil-core`，里面分别导出了 CommonJS 模块、ES 模块和 TypeScript 类型声明，用于在不同的模块系统和开发环境中正确加载和使用包。

但是通过 `src/internal/stencil-core/index.d.ts` 可以看到所有导出的接口都应该在 `internal` 目录下，但是在仓库中却找不到这个目录。其实稍一猜就能猜到这是打包之后产物的目录，只是在 `.gitignore` 中隐藏了。之所以顺便讲一讲 Stencil.js 的打包和编译流程，是因为这个项目乍一看整个仓库只有一个 `package.json`，但其实是一个 Monorepo，只不过区别在于相比普通的 Monorepo，它并非是按照一个 workspace 就是一个 npm 包的样子进行分包。因此要想了解具体代码的分布还是需要看一下它的打包规则才知道核心代码具体放在哪里。

## 项目编译

我们还是了解一下 Stencil 整个项目的编译流程，查看了 `package.json` 的 build scripts 之后可以看到 Stencil 的项目编译方式：先通过 tsc 编译 TypeScript 类型声明，通过 Rollup 编译 CommonJS 模块和 ES 模块，编译脚本主入口是 `scripts/build/build.js` 的 `createBuild` 方法。整体的打包和编译流程主要如下：

1. 清空所有打包目标路径文件夹
2. 打包 `src/sys/node/bundles` 下的文件（主要是导出一些 Node.js 包），如果没有缓存则通过 Webpack 打包并写入缓存
3. 接着按 npm 包的维度进行打包，最后返回一个 `RollupOptions` 数组

从 `scrips` 目录下还可以看到有一个 `scripts/esbuild` 目录，因此 Stencil 的代码仓库里其实包含了 Rollup、Webpack 和 Esbuild 三种打包工具。但看了一下代码，最主要的 internal 包还没有接入 Esbuild，因此我的猜测是 Stencil 会准备将 Webpack 编译的部分全部迁移到 Esbuild 上，抑或是两者并存的形式。

## 项目结构

下面可以大致看一下 `src` 下的源码结构，并讲解一下部分目录的主要用途。

```sh
├── app-data
├── app-globals
├── cli
├── client
├── compiler
├── declarations
├── dev-server
├── hydrate
├── internal
├── mock-doc
├── runtime
├── screenshot
├── sys
├── testing
└── utils
```

1. `cli`: 这个目录是和 Stencil 的命令行工具相关的代码
2. `client`: 这个目录主要存放和客户端、浏览器相关的代码
3. `compiler`: 这个目录下主要是 Stencil 的编译器实现代码，用于将一个 Stencil 项目编译成符合规范的 Web Components
4. `declarations`: 这个目录下主要是一些类型声明文件
5. `hydrate`: 这个目录下主要是和 [Hydrate App](https://stenciljs.com/docs/hydrate-app#hydrate-app) 相关的实现代码
6. `runtime`: 这个目录下主要是和运行时相关的代码，包括 Stencil 的 VDOM 实现等
