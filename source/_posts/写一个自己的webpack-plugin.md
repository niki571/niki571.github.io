---
title: 写一个自己的webpack plugin
date: 2021-06-30 19:19:04
tags: [webpack]
categories: 工程化
---

想必大家都用过 webpack，也或多或少了解它的原理，但是不知道大家有没有写过 Webpack 的插件呢？

今天我们就一起来写一个划分 Chunk 的 webpack 插件吧，写完后你会发现想怎么分 Chunk 都可以！

首先我们简单了解下 webpack 的原理：

<!-- more -->

# webpack 的原理

webpack 是一个打包工具（bundler），它打包的是什么呢？

模块。

那模块能再拆分么？

不能了，模块是 webpack 处理的基本单位了，只是对模块做一些打包。

那怎么对模块打包呢？

首先要找到所有的模块(Module)，从入口模块开始，分析依赖，构成一个图，叫做模块依赖图(ModuleGraph)。

然后模块要分成几个包，要有一种中间结构来保存这种划分，叫做 Chunk。把不同的模块放到不同的 Chunk 里就完成了分包。

但是 Chunk 只是一种中间结构，还要再变成可用的目标代码。通过代码模版把它打印成代码就可以了。

**这三步分别叫 make、seal、emit。**

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/8ccf085606c64a70972e21185c3d155a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

make 这一步就是构建模块依赖图 ModuleGraph 的，这个过程中会从入口模块（EntryPoint）开始递归解析依赖，对解析出的每个模块做处理，也就是调用注册的 loader。

然后 Seal 也就是封装的意思，把不同的 Module 分到不同的 Chunk 里。

这一步会先做基础的 Chunk 划分，比如入口模块 EntryPoint 肯定要单独放到 Chunk 里，动态引入的模块肯定也要单独放到 Chunk 里。

完成了基础的划分之后，可以再对这些 Chunk 做进一步的优化划分，比如根据 Chunk 大小等来划分。

分完之后，ModuleGrapqh 就变成了 ChunkGraph。

最后 emit 阶段就是通过模版打印代码了。

这三步合起来就是一次编译过程 Compilation。

编译过程由 webpack 的 Compiler 调用。

整个过程中还会暴露出很多扩展点，也就是留给插件的 hook，不同阶段的 hook 自然就可以拿到不同阶段的资源。

这些插件都是保存在对象上的：

比如 compiler 的 hook：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/4a32a8b438864db4a97f03cf8f132e09_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

compilation 的 hook：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/6c5da4601ba3427e997f2ef83613ea90_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

那插件里自然就是往不同对象的 hook 上添加回调函数：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/beefbb79669d4e8f8b361d3a1398ee9f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

而且 webpack 为了控制 hook 的执行顺序，封装了一个 tappable 的包。可以指定 hook 是同步、异步，并行、串行执行。

比如这几种 hook：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/194947f802734f549b2ce8b1984ab777_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

- SynHook 就是同步顺序执行。

- AsyncSeriesHook 就是异步串行执行。

- SyncBailHook 也是同步顺序执行，但是如果中间的 hook 返回 false 就会停止后续 hook 的执行，也就是可以熔断。

理解了 webpack 的编译流程，hook 的运行机制，接下来我们就写个插件来操作下 Chunk 吧。

# 操作 Chunk 的 webpack 插件

前面讲过，webpack 会对 Module 根据是否是入口模块、是否是异步引入的模块做基础的 Chunk 划分。

之后会进一步做优化的 Chunk 划分。

这些 chunk 相关的逻辑都是在 seal 那一步做的。

我们在源码里看到的也确实是这样：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/ebfd6ba1a91144b49a454fc70191b8e4_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

在 seal 里做了 ChunkGraph 的创建，然后调用 optimizeChunks 的 hook 对 Chunks 做处理。

这里为啥是个死循环呢？

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/b9ed952bb9c740a3857acaf01fc4573f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

记得上面说过一种 hook 类型叫 SyncBailHook 么？

也就是同步执行插件，但是可以插件可以返回 false 熔断后面插件的执行。

这里的 hook 就是同步熔断 hook：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/0f20e5f37bd04fbe8aeb5be9bdcad79f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

那我们就开始在这个 hook 里写一些逻辑吧：

```javascript
class ChunkTestPlugin {
  constructor(options) {
    this.options = options || {};
  }

  apply(compiler) {
    const options = this.options;

    compiler.hooks.thisCompilation.tap("ChunkTestPlugin", (compilation) => {
      compilation.hooks.optimizeChunks.tap("ChunkTestPlugin", (chunks) => {
        return true;
      });
    });
  }
}

module.exports = ChunkTestPlugin;
```

把 options 挂到 this 上。

然后注册一个 optimizeChunks 这个 hook 的回调。

为啥外面还要加一层 compiler 的 hook 呢？

因为你得在 compiler 刚开始编译的时候去注册 compilation 的 hook 呀！不然就晚了。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/e1a98a1d26eb4dc6aac12d15e5b482ed_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

可以看到 thisCompilation 是在 newCompilation 这个方法调用的。

而 newCompilation 是在 make、seal、emit 的流程开始之前调用的：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/53c92bf2dfef43269dcd49335facc0b8_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

也就是说在 thisCompilation 的 Compiler hook 里注册的 Compilation hook 就可以在这次编译过程中生效。

有的同学说，那还有另一个 hook 是干啥的呢？

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/5bda3d740df34f7983c610e8186008eb_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

这俩 hook 唯一的区别是当有 child compiler 的时候，compilation 的 hook 会生效，而 thisCompilation 不会。

而我们是想在这个 hook 里注册 Compilation 的 hook 的，全局只需要执行一次就行，所以用 thisCompilation 的 Compiler hook。

我们在项目里用一下：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/067134578b834475bf54e66508bb9ef2_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

这个项目有三个入口模块：

pageA：

```javascript
require(["./common"], function (common) {
  common(require("./a"));
});
```

pageB：

```javascript
require(["./common"], function (common) {
  common(require("./b"));
});
```

```javascript
pageC：
require(["./a"], function(a) {
console.log(a + require("./b"));
});
```

这三个模块里都通过 requrie() 或者 import() 的 webpack api 来动态引入了一些模块。

动态引入的模块分别是：

a：

```javascript
module.exports = "a";
```

b：

```javascript
module.exports = "b";
```

common：

```javascript
module.exports = function (msg) {
  console.log(msg);
};

function hello() {
  return (
    "guangguangguangguangguangguanggua" +
    "ngguangguangguangguangguangguangguangguangg" +
    "uangguangguangguangguangguangguangguangguangg" +
    "uangguangguangguangguangguangguangguangguang" +
    "guangguangguangguangguangguangguangguangguangguang" +
    "guangguangguangguangguangguangguangguangguangguangguang"
  );
}
```

webpack 会从入口模块开始构建 ModuleGraph，然后划分 Chunk，构成 ChunkGraph。

大家觉得这几个模块会分几个 Chunk 呢？

6 个。

因为入口模块要用单独的 Chunk，而且异步引入的模块也是单独的 Chunk。

打个断点看一下就知道了：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/306f246a63c24e939eb7a189eebf4d54_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

确实，到了 optimizeChunks 这一步，拿到的是 6 个 Chunk：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/51d8fe660ffb4d62aa25a96d6ea73529_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

分别是 3 个入口，以及每个入口用到的异步模块。

在这个 optimizeHook 的插件里，我们就可以自己做一些 Chunk 拆分了。

chunkGroup 有一个 integrateChunks 的 api，把后面的 chunk 合并到前面的 chunk 里：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/f0d7718de2ca41c5817bb76d1ca53e84_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

我们调用 integrateChunks 进行 chunk 合并，然后把被合并的那个 chunk 删掉即可。

那怎么找到 a 和 b 两个 chunk 呢？

两层循环，分别找到两个不想等的 chunk 进行合并即可：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/aee11acc2daa43b2a9e641abce7e263f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

我们只取第一组 chunk 进行合并，合并完如果还有就返回 true，继续进行下次合并。

合并完之后记得 return false，因为外面是一个 while 循环，不 return false，就一直死循环。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/041e0dfa32b649d09227f31e3890dec0_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

先试一下现在的效果：

不引入插件的时候是这样的：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/9ada46f3928542759836248ac161aeee_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

3 个入口 chunk，3 组入口 chunk 的异步引入的模块。所以产生了 6 个文件。

入口 chunk 对应的文件里引入异步模块的方法变成了 webpack runtime 的 \_webpack_require.e

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/ab3f1e4ed00141f189d7defab9093a01_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

而它引入的异步 chunk 里就如前面分析的，包含了这个模块的所有异步依赖：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/21cd6aaf4f29490ca5acb9d9159f83f0_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/21cd6aaf4f29490ca5acb9d9159f83f0_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/1ddfb5bc17454510acc4f665ab21231b_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

分别是 a + common，b + common，a + b，也就是每个入口模块依赖的所有异步模块。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/d24f06a5529141b291728e39650069e8_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/328a4751ba06450d85f162bc04dfb200_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/6f7070fc4a0f4991916d6ab93f18c1ba_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

那优化之后呢？

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/41265d4cf9c24153bc305f1ce982be06_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

都放到一个 chunk 里了：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/4a883077c36e44d3ac0fd6a19f43bfb4_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

这倒是符合我们写的逻辑，因为两两合并，最后剩下的肯定只有一个。

但这样显然不大好，因为每个页面是独立的，应该分开，但是异步的 chunk 倒是可以合并。

所以我们优化一下：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/988934861746430b8351d3cd5cca9cf4_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

调用 chunk 的 isInitial 方法就可以判断是否是入口的 chunk，是的话就跳过。

这样就只合并了异步 chunk。

效果是这样的：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/e914f1668cd043058f5320072a2443c4_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

3 个入口 chunk 的依赖也变成这个 chunk 了：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/bf0c3297c5c24996b2756f78664fdaf1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

那如果我要根据 chunk 大小来优化呢？

那就可以判断下 a、b 的 chunk 的大小和合并之后的 chunk 大小，如果合并之后比合并前小很多，就合并。

当然，不同的 chunk 合并效果是不一样的，我们要把所有的合并效果下来：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/3b47d20186bd45bb8e35ea74c72fe585_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

通过 chunkGraph.getChunkSize 的 api 拿到 chunk 大小，通过 chunkGroup.getIntegratedChunkSize 的 api 拿到合并后的 chunk 大小。

记录下合并的两个 chunk 合并的收益。

做个排序，把合并收益最大的两个 chunk 合并。

返回 true 来继续循环进行合并，直到收益小于 1.5，那就 return false 停止合并。

当然，这个 1.5 也可以通过 options 传进来。

效果是这样的：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/1a3dcfb02340400580fe0f11a732c424_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

两个异步 chunk 分别为：

a + b + common：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/203925c262ca420dbb66507f8b0a32b5_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

a + b：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/398838d882264afda087bc94f3aa8231_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

也就是说只把之前的 a + common 和 b + common 合并了，因为 common 模块比较大，所以合并之后的收益是挺大的。

这样就完成了 chunk 拆分的优化。

有的同学说，我平时也不用自己写插件来拆分 chunk 呀，webpack 不是提供了 SplitChunksPlugin 的插件么，还变成内置的了，配置下 optimization.splitChunks 就行。

没错，webpack 默认提供了拆分 chunk 的插件。

那这个插件是怎么实现的呢？

没错，SplitChunkPlugin 的实现原理就是我们刚才说的这些，注册了 optimizeChunks 的 hook，在里面做了 chunk 拆分：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/3e1509975afb4169bb2331c7d7c84f7f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

它可以根据配置来拆分 chunk，但是终究是有局限性的。

如果某种 chunk 拆分方式它不支持呢？

我们就可以写插件自己拆分了，会自己拆分 chunk 之后，还不是想怎么分就怎么分么！

我们写的这个 webpack 插件的全部代码如下：

```javascript
class ChunkTestPlugin {
  constructor(options) {
    this.options = options || {};
  }

  apply(compiler) {
    const options = this.options;
    const minSizeReduce = options.minSizeReduce || 1.5;

    compiler.hooks.compilation.tap("ChunkTestPlugin", (compilation) => {
      compilation.hooks.optimizeChunks.tap("ChunkTestPlugin", (chunks) => {
        const chunkGraph = compilation.chunkGraph;

        let combinations = [];
        for (const a of chunks) {
          if (a.canBeInitial()) continue;
          for (const b of chunks) {
            if (b.canBeInitial()) continue;
            if (b === a) break;

            const aSize = chunkGraph.getChunkSize(b, {
              chunkOverhead: 0,
            });
            const bSize = chunkGraph.getChunkSize(a, {
              chunkOverhead: 0,
            });
            const abSize = chunkGraph.getIntegratedChunksSize(b, a, {
              chunkOverhead: 0,
            });
            const improvement = (aSize + bSize) / abSize;

            combinations.push({
              a,
              b,
              improvement,
            });
          }
        }

        combinations.sort((a, b) => {
          return b.improvement - a.improvement;
        });

        const pair = combinations[0];

        if (!pair) return;
        if (pair.improvement < minSizeReduce) return;

        chunkGraph.integrateChunks(pair.b, pair.a);
        compilation.chunks.delete(pair.a);
        return true;
      });
    });
  }
}

module.exports = ChunkTestPlugin;
```

# 总结

webpack 的处理单位是模块，它的编译流程分为 make、seal、emit：

- make：对入口模块分析依赖，构建 ModuleGraph，对每个模块调用 loader 处理。
- seal：合并 Module 为 Chunk，合并之后 ModuleGraph 会变为 ChunkGraph。
- emit：对每个 Chunk 通过模版打印成代码后输出

这个编译流程中有很多 hook，通过 tappable 的 api 组织，可以控制回调的同步、异步、串行、并行执行。

我们今天写的 Chunk 拆分插件，就是一个 SyncBailHook，同步熔断的串行 hook 类型，也就是前面回调返回 false 会终止后面的回调执行。

首先在 compiler 的 thisCompilation 的 hook 里来注册 compilation 的 optimizeChunks 的 hook。

在 optimizeChunks 的 hook 里可以拿到所有的 chunk，调用 chunkGraph 的 api 可以进行合并。

我们排除掉了入口 chunk，然后把剩下的 chunk 根据大小进行合并，达到了优化 chunk 的目的。

webpack 内置了 SplitChunksPlugin，但是毕竟有局限性，当不满足需求的时候就可以自己写插件来划分 chunk 了。

自己来控制 Chunk 划分，想怎么分就怎么分！
