---
title: Vue2 与 Vue3 的区别
date: 2022-08-20 19:02:07
tags: [Vue]
categories: Vue
---

# 前言

Vue 内部根据功能可以被分为三个大的模块：`响应性 reactivite`、`运行时 runtime`、`编辑器 compiler`，以及一些小的功能点。

那么要说 `vue2` 与 `vue3` 的区别，我们需要从这三个方面加小的功能点进行说起。

# 响应性 reactivite

`vue2` 的响应性主要依赖 `Object.defineProperty` 进行实现，但是 `Object.defineProperty` 只能监听`指定对象`的`指定属性`的 `getter` 行为和 `setter` 行为，那么这样在某些情况下就会出现问题。

<!-- more -->

什么问题呢？

比如说：我们在 data 中声明了一个对象 person ，但是在后期为 person 增加了新的属性，那么这个新的属性就会失去响应性。

想要解决这个问题其实也非常的简单，可以通过 `Vue.$set` 方法来增加指定对象指定属性的响应性。但是这样的一种方式，在 `Vue` 的自动响应性机制中是不合理。

所以在 `Vue3` 中，Vue 引入了`反射`和`代理`的概念，所谓`反射`指的是 `Reflect`，所谓`代理`指的是 `Proxy`。

我们可以利用 `Proxy` 直接代理一个普通对象，得到一个 `proxy` 实例的代理对象。在 `vue3` 中，这个过程通过 `reactive` 这个方法进行实现。

但是 `proxy` 只能实现代理复杂数据类型，所以 `vue` 额外提供了 `ref` 方法，用来处理简单数据类型的响应性。

`ref` 本质上并没有进行数据的监听，而是构建了一个 `RefImpl` 的类，通过 `set` 和 `get` 标记了 `value` 函数，以此来进行的实现。所以 `ref` 必须要通过 `.value` 进行触发，之所以要这么做本质是调用 `value` 方法。

# 运行时 runtime

所谓的运行时，大多数时候指的是 `renderer 渲染器`，渲染器本质上是一个对象。内部主要三个方法 `render`、`hydrate`、`createApp` ，其中 render 主要处理渲染逻辑，`hydrate` 主要处理服务端渲染逻辑，而 `createApp` 就是创建 `vue 实例`的方法。

这里咱们主要来说 `render` 渲染函数，`vue3` 中为了保证宿主环境与渲染逻辑的分离，把所有与宿主环境相关的逻辑进行了抽离，通过接口的形式进行传递。这样做的目的其实是为了解绑宿主环境与渲染逻辑，以保证 vue 在非浏览器端的宿主环境下可以正常渲染。

# 编辑器 compiler

vue 中的 `compiler` 其实是一个 `DSL`（特定领域下专用语言编辑器） ，其目的是为了把 `template 模板` 编译成 `render 函数`。 逻辑主要是分成了三大步：`parse`、`transform` 和 `generate`。

其中 `parse` 的作用是为了把 `template` 转化为 `AST（抽象语法树）`，`transform` 可以把 `AST（抽象语法树）` 转化为 `JavaScript AST`，最后由 `generate` 把 `JavaScript AST` 通过转化为 `render 函数`。转化的过程中会涉及到一些稍微复杂的概念，比如 有限自动状态机 这个就不再这里展开说了。

# 除此之外，还有一些其他的变化

比如 `vue3` 新增的 `composition API`。 `composition API` 在 `vue3.0` 和 `vue3.2` 中会有一些不同的呈现，比如说：最初的 `composition API` 以 `setup 函数`作为入口函数，`setup 函数`必须返回两种类型的值：第一是对象，第二是函数。

当 `setup 函数`返回`对象`时，对象中的数据或方法可以在 `template` 中被使用。当 `setup 函数`返回`函数`时，函数会被作为 `render 函数`。

但是这种 `setup 函数`的形式并不好，因为所有的逻辑都集中在 `setup 函数`中，很容易出现一个巨大的 `setup 函数`，我们把它叫做巨石（屎山）函数。

所以 `vue 3.2` 的时候，新增了一个 `script setup` 的语法糖，尝试解决这个问题。目前来看 `script setup` 的呈现还是非常不错的。

除此之外还有一些小的变化，比如 `Fragment`、`Teleport`、`Suspense` 等等，这些就不去说了...
