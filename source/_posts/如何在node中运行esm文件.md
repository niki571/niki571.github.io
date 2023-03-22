---
title: 如何在 node 中运行 esm 模块
date: 2022-03-22 16:49:15
tags: [Node]
categories: Node
---

我们都知道前端有两种常见的模块化规范，一种是 ES6 模块 简称 ESM，一种是 Node.js 专用的 CommonJS 模块 简称 CJS，两者是不兼容的。

我最近在写 node 项目，整个项目是用 @nest/cli 起的，文件模块默认用的是 ESM。为了快捷验证我的爬虫，想直接在 node REPL 环境中单独运行，但总是报错。

<!-- more -->

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/%E6%88%AA%E5%B1%8F2023-03-22%20%E4%B8%8B%E5%8D%885.36.13.png)

研究了下，有 2 种方便快捷的方法可以在 node REPL 中跑 ESM。

# 第一种

直接将文件后缀从`.ts`改为`.mjs`：

```shell
node crawl.mjs
```

# 第二种

环境中分别安装 typescript 和 ts-node：

```shell
npm install -g typescript
npm install -g ts-node
```

修改`tsconfig.json`：

```json
"module": "ESNext", /* Specify what module code is generated. */
"moduleResolution": "node", /* Specify how TypeScript looks up a file from a given module specifier. */
```

修改`package.json`：

```json
 "type": "module",
```

运行文件：

```shell
ts-node --esm crawl.ts
```
