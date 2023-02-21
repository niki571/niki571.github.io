---
title: 基于 timestamp 和 nonce 的防止重放攻击方案
date: 2022-04-07 11:09:39
tags: [网络, 爬虫]
categories: 爬虫
---

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/1610c97ae68f2c5f_tplv-t2oaga2asx-zoom-crop-mark_3024_3024_3024_1702.awebp)

以前总是通过 timestamp 来防止重放攻击，但是这样并不能保证每次请求都是一次性的。今天看到了一篇文章介绍的通过 nonce（Number used once）来保证一次有效，感觉两者结合一下，就能达到一个非常好的效果了。

<!-- more -->

> 重放攻击是计算机世界黑客常用的攻击方式之一，所谓重放攻击就是攻击者发送一个目的主机已接收过的包，来达到欺骗系统的目的，主要用于身份认证过程。

首先要明确一个事情，重放攻击是二次请求，黑客通过抓包获取到了请求的 HTTP 报文，然后黑客自己编写了一个类似的 HTTP 请求，发送给服务器。也就是说服务器处理了两个请求，先处理了正常的 HTTP 请求，然后又处理了黑客发送的篡改过的 HTTP 请求。

# 基于 timestamp 的方案

每次 HTTP 请求，都需要加上 timestamp 参数，然后把 timestamp 和其他参数一起进行数字签名。因为一次正常的 HTTP 请求，从发出到达服务器一般都不会超过 60s，所以服务器收到 HTTP 请求之后，首先判断时间戳参数与当前时间相比较，是否超过了 60s，如果超过了则认为是非法的请求。

假如黑客通过抓包得到了我们的请求 url： koastal.site/index/Info?…

其中

```php
    $sign=md5($uid.$token.$stime);
// 服务器通过 uid 从数据库中可读出 token
```

一般情况下，黑客从抓包重放请求耗时远远超过了 60s，所以此时请求中的 stime 参数已经失效了。 如果黑客修改 stime 参数为当前的时间戳，则 sign 参数对应的数字签名就会失效，因为黑客不知道 token 值，没有办法生成新的数字签名。

但这种方式的漏洞也是显而易见的，如果在 60s 之后进行重放攻击，那就没办法了，所以这种方式不能保证请求仅一次有效。

# 基于 nonce 的方案

nonce 的意思是仅一次有效的随机字符串，要求每次请求时，该参数要保证不同，所以该参数一般与时间戳有关，我们这里为了方便起见，直接使用时间戳的 16 进制，实际使用时可以加上客户端的 ip 地址，mac 地址等信息做个哈希之后，作为 nonce 参数。

我们将每次请求的 nonce 参数存储到一个“集合”中，可以 json 格式存储到数据库或缓存中。 每次处理 HTTP 请求时，首先判断该请求的 nonce 参数是否在该“集合”中，如果存在则认为是非法请求。

假如黑客通过抓包得到了我们的请求 url： koastal.site/index/Info?…

其中

```php
$sign=md5($uid.$token.$nonce);
// 服务器通过 uid 从数据库中可读出 token
```

nonce 参数在首次请求时，已经被存储到了服务器上的“集合”中，再次发送请求会被识别并拒绝。 nonce 参数作为数字签名的一部分，是无法篡改的，因为黑客不清楚 token，所以不能生成新的 sign。

这种方式也有很大的问题，那就是存储 nonce 参数的“集合”会越来越大，验证 nonce 是否存在“集合”中的耗时会越来越长。我们不能让 nonce“集合”无限大，所以需要定期清理该“集合”，但是一旦该“集合”被清理，我们就无法验证被清理了的 nonce 参数了。也就是说，假设该“集合”平均 1 天清理一次的话，我们抓取到的该 url，虽然当时无法进行重放攻击，但是我们还是可以每隔一天进行一次重放攻击的。而且存储 24 小时内，所有请求的“nonce”参数，也是一笔不小的开销。

# 基于 timestamp 和 nonce 的方案

那我们如果同时使用 timestamp 和 nonce 参数呢？ nonce 的一次性可以解决 timestamp 参数 60s 的问题，timestamp 可以解决 nonce 参数“集合”越来越大的问题。

我们在 timestamp 方案的基础上，加上 nonce 参数，因为 timstamp 参数对于超过 60s 的请求，都认为非法请求，所以我们只需要存储 60s 的 nonce 参数的“集合”即可。

假如黑客通过抓包得到了我们的请求 url： koastal.site/index/Info?…

其中

```php
$sign=md5($uid.$token.$stime.$nonce);
// 服务器通过 uid 从数据库中可读出 token
```

如果在 60s 内，重放该 HTTP 请求，因为 nonce 参数已经在首次请求的时候被记录在服务器的 nonce 参数“集合”中，所以会被判断为非法请求。超过 60s 之后，stime 参数就会失效，此时因为黑客不清楚 token 的值，所以无法重新生成签名。

综上，我们认为一次正常的 HTTP 请求发送不会超过 60s，在 60s 之内的重放攻击可以由 nonce 参数保证，超过 60s 的重放攻击可以由 stime 参数保证。

因为 nonce 参数只会在 60s 之内起作用，所以只需要保存 60s 之内的 nonce 参数即可。

我们并不一定要每个 60s 去清理该 nonce 参数的集合，只需要在新的 nonce 到来时，判断 nonce 集合最后一次修改时间，超过 60s 的话，就清空该集合，存放新的 nonce 参数集合。其实 nonce 参数集合可以存放的时间更久一些，但是最少是 60s。

验证流程

```php
//判断stime参数是否有效
if( $now - $stime > 60){
    die("请求超时");
}
//判断nonce参数是否在“集合”已存在
if( in_array($nonce,$nonceArray) ){
    die("请求仅一次有效");
}
//验证数字签名
if ( $sign != md5($uid.$token.$stime.$nonce) ){
    die("数字签名验证失败");
}
//判断是否需要清理nonce集合
if( $now - $nonceArray->lastModifyTime > 60 ){
    $nonceArray = null;
}
//记录本次请求的nonce参数
$nonceArray.push($nonce);


//开始处理合法的请求
```
