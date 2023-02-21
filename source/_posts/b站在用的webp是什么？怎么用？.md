---
title: b站在用的webp是什么？怎么用？
date: 2022-08-21 16:13:15
tags: 图片
categories: 前端
---

打开控制台，`ctrl + c`看一下 b 站图片的后缀/格式，发现都是`.webp`后缀了。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/c49ccde1164d4b0ca5e617b3350b5010_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

# 背景介绍

webp 是谷歌 2010 年整出来的, 是 jpeg, png 经过 webp 压缩算法后得到的, 它的压缩算法有 1. 无损 2. 有损 两种，无损肯定是质量牛一点比较贴近 jpeg, png 原来的画质, 有损讲体积压缩得狠一点，关于具体压缩的体积比，可以看一下来自知乎的回答 WebP 相对于 PNG、JPG 有什么优势？, 这里就上一中其中的图：

<!-- more -->

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/fd808a0acdc449fe868798934d015aa1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

注：本篇文章不会讨论 webp 算法的实现过程, 侧重于 webp 概念介绍以及使用

总而言之，webp 可以优化我们请求图片时的速度，图片越小下载越快，算是页面性能优化的一个点，所以 webp 能用来干什么？

# webp 的用处

用于页面性能优化，优化页面图片加载速度，减小图片体积, 能保证压缩后的图片相对于原文件仍有较高的质量

# webp 适用场景

页面需要渲染大量图片的时候，就有必要考虑一下 webp 了，我随机查看了几个内容型网站，京东, 哔哩哔哩, 淘宝, 掘金, 知乎回答部分，其中

1. 京东, 哔哩哔哩, 淘宝首页均采用了 webp 格式图片, 掘金文章内容和广告图片用的是 awebp(难道是 article webp?)
2. 知乎回答内容里面并没有使用 webp 而是 jpg, 可能是默认一般人写回答的时候不会使用图片吧？

# webp 缺点

webp 并不是完美的, 它的缺点是它的兼容性和解码速度以及图片的保存上需要备份原图以备兼容处理, webp 解码速度比 jpg 慢 1.5 倍, 不过解码速度的优化上，应该交给浏览器，备份保存也不会讲，交给后端，兼容性的处理我们就能上手了, webp 兼容性的处理需要两个核心点

1. 检测用户浏览器是否支持 webp
2. 丝滑般的切换 - webp to jpg - 降级处理

# 检测是否支持

## Modernizr

这个`Modernizr`的原理就是支持的就在`HTML`上加上表示支持的类名，不支持的就加不支持的类名，`Modernizr`需要下载 npm 自行打包，我暂时没有发现到相关的 CDN 像`jsdeliver`上面都没有， Modernizr github 链接 ，在 github 上通过相应的配置文件自己挑选并生成对应的 js 文件，模块化算是给它玩明白了

PS: Modernizr github 上的 build 教程不是很人性化，不想看的可以直接按照我下面这段 js 操作

```javascript
// npm install modernizr
// 在当前文件目录下生成 modernizr.js
var modernizr = require("modernizr");
var fs = require("fs");

modernizr.build(
  {
    "feature-detects": [
      "img/webp-alpha",
      "img/webp-animation",
      "img/webp-lossless",
      "img/webp",
    ],
  },
  function (result) {
    fs.writeFile("modernizr.js", result, (err) => {
      if (err) throw err;
    });
  }
);
```

让我们小试一下，在这里先感谢一下老朋友 IE 为兼容性处理对比上做出的贡献, 参考下面的示例代码

```javascript
<!DOCTYPE html>
<html>
  <body>
    <div></div>
  </body>
  <script src="./modernizr.js"></script>
  <style>
    div {
      height: 380px;
      width: 400px;
    }
    .no-webp div {
      background-image: url("./img/img.png");
    }

    .webp div {
      background-image: url("./img/img.png");
    }

  </style>
</html>
```

其效果如下

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/d41bb85d15bb40f681a40a5994a506a1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/6a3189181ede4920876aff409420db23_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

注意看右边样式和`style`里面的显示的图片链接后缀，IE 是`.jpg`而 Chrome 的是`.webp`说明我们的`Modernizr`生效了，而`Modernizr`的作用方式也就是通过在`HTML`上添加`class`类名并搭配自定义的 CSS 代码实现`webp`和`jpg`切换处理，不支持类名就是`no-webp`，支持就是`webp`

Modernizr 检测到浏览器后会给 HTML 加上类名, 然后我们就可以丝滑般的切换了，不过我个人认为并不是很实用, 依赖于 CSS 和 HTML 的切换太过繁琐，往往后端都是传递动态的链接给前端，今天明天不一样的链接，url 随时会换，这种方案此时就不起作用了，而且依赖于 Modernizr 的检测，万一不能用 JS 或者因为 First Plain 导致 FOUC 样式闪烁导致背景图左右横跳反而会得不偿失

注：大多数同类文章中，仙人指路去 Modernizr 下载文件，但是现在它的官网已经失效了，应该去 npm 下载自己构建

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/79e29ba0937e4e3390ed732ea6500552_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

### 适用场景

静态图片, 如主页背景图等一些不易变换的图片

## HTML5 - `<picture>`

`<picture>`是 HTML5 中新增的一个标签，它的原理是利用浏览器是否支持图片格式（type），因此在 HTML5 可以帮你自动选是`webp`还是`jpg`, `png`等其他格式，缺点是不能用在`background-image`属性上，这种情况可以和`Modernizr`折中一下

```javascript
<picture>

  <source srcset="./img/image.webp" type="image/webp" />
  <source srcset="./img/image.jpg" type="image/jpeg" />
  <!-- 默认值 -->
  <img src="./img/image.jpg" alt="img" />
</picture>
```

这种方式直接在 HTML 上是看不了到底是渲染哪个图片了的，但是不要紧，小细节，我们请求了图片，那么在 network 上一定有相应的请求发送，Chrome 效果如下

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/80b053619ef14921bad5cbd7a80ceb2a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

IE 的比较麻烦，Element 上看不到，Network 上也看不到，但是我最终还是通过将图片另存为这种方式知道了 IE 最后渲染得到图片格式

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/ca1f80e8744542e19b573c925d70b740_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

## 通过请求判断

知名摆烂选手 IE 发送图片请求时, 请求标头属性中 Accept 是这样的

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/eca4eeacaa6b4987a88502dcc232cd11_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

Chrome 请求标头属性中 Accept 是这样的

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/fd808a0acdc449fe868798934d015aa1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

后端看到这个还不分分钟钟拿捏？需要返回什么，要和前端怎么做商量好就行了

# 降级处理

有京东选手登场

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/605f1cdb454a4ec0a543f5cd75cca21a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

看到没有，图片的后缀是 .jpg.webp, 前面已经讲过了检测是否支持的方法，当我们知道能不能用之后，就可以对链接字符串操作了，支持你就改成 jpg 不支持不改

# 最佳方案？

还是来自京东的选手啊，这是他们文章里面给出的方案, 可以通过谷歌开发的 PageSpeed 去自动转换，省去保存备份和转换的麻烦, 它最终的效果就是源码你可以写带 jpg 的, nginx 开启 PageSpeed 配置后, 会对你的原生的 HTML 做出一点点改变

```javascript
<!-- 源码 -->
<body>
  <img src="./574ceeb8N73b24dc2.jpg" />
  <img src="./6597241290470949609.png" />
</body>
<!-- 支持 webp - 用户实际接收到的 -->
<body>
  <img src="x574ceeb8N73b24dc2.jpg.pagespeed.ic.YcCPjxQL4t.webp" />
  <img src="x6597241290470949609.png.pagespeed.ic.6c5y5LYYUu.webp" />
</body>
<!-- 不支持 webp - 用户实际接收到的 -->
<body>
  <img src="x574ceeb8N73b24dc2.jpg.pagespeed.ic.3TXX_PUg99.jpg" />
  <img src="x6597241290470949609.png.pagespeed.ic.rrgw7vPMd6.png" />
</body>
```

又是一个提示幸福感的小技巧啊

# 总结

是否需要考虑 webp 的图片方案，取决于你的页面，如果文字占比比较重的比如知乎，你上 webp 会不会是对你一种身心摧残，因为你可能弄半天看不到页面加载的效果，如果你全是图片而且页面加载还慢的一批，可以考虑一下 webp
