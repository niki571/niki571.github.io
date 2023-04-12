---
title: 一文详解小程序授权、登录、session_key 和 unionId
date: 2021-09-15 11:01:37
tags: [小程序]
categories: 前端
---

微信应用的一个很大的优势就在于使用过程中是不需要进行注册和显式登录的，大部分问题基本上可以一键解决。但是在授权、登录和获取用户信息的过程中都发生了哪些事情，今天我们就来讨论一下。这篇文章主要分析以下几个问题：

1. 授权和登录的意义
2. session_key 的作用
3. unionId 的作用，有哪些获取途径
4. 在应用中如何保存用户登录态

<!-- more -->

# 授权和登录的意义

首先必须要明白，授权和登录实际上是两个操作。

## 授权

那授权的作用是啥呢？从小程序官方文档中我们可以看到授权操作只需通过`wx.authorize()` 接口便可以完成，以下是文档中对授权操作的描述：

提前向用户发起授权请求。调用后会立刻弹窗询问用户是否同意授权小程序使用某项功能或获取用户的某些数据，但不会实际调用对应接口。如果用户之前已经同意授权，则不会出现弹窗，直接返回成功。

也就是说，授权过程实际上只是在小程序前端获得了操作部分 wx 接口的访问许可，**这个过程实际上是不会与开发者服务器发生任何关系的**。那这些访问许可包含哪些内容呢？再来看微信官方提供的 scope 列表：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/snq7zpqzno.png)

> 注：新版 api 已废弃 wx.authorize()

## 登录

所谓的登录就是要让开发者服务器知道当前的用户是谁？在传统的 web 应用中，我们必须要让用户输入账号和密码才能实现登录操作。但是在微信应用中，我们可以通过微信服务器来完成这个操作，获取到与当前用户对应的唯一标志`openId`，具体操作实现流程如下：

`wx.login()`用来做登录的方法，调用接口获取登录凭证，`code` 发送给后端用于置换 `session_key` 和 `openid` 等数据。每个用户相对于每个微信应用（公众号或者小程序）的 `openId` 是唯一的，也就是说一个用户相对于不同的微信应用会存在不同的 `openId`。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/nvm3jqw0x9.jpeg)

**这是小程序官方的一张登录流程图，现在就来解读一下这个流程**

- 前端 `wx.login()`获取 `code`，调用后端接口，将得到的 `code` 发送到后端
- 后端调用微信接口，用 `appid`+`appsecret`+`code` 发送过去，置换到 `session_key`+`openid`，以前是不能置换 `unionid` 的，但是现在在满足以下条件可以置换到 `unionid`
  1. 微信开放平台下存在同主体的 App、公众号、小程序
  2. 用户关注了某个相同主体公众号，或曾经在某个相同主体 App、公众号上进行过微信登录授权 同时满足以上两个条件就能拿到用户 `unionid`，这样一来，就能在 `wx.login()`准确识别出用户是谁
- 自定登录态与 `openid` 和 `session_key` 关联，实际就是生成一个与 `openid`，`session_key` 关联的 `token`，下发给前端
- 前端将后端下发的 `token` 存入缓存，在后面的接口请求中带上自定登录态

以上就是小程序的整个登录流程，可以看到其实并不是一定要 `wx.getUserInfo()`才能拿到用户的信息，在特定的条件下，通过 `wx.login()`的调用拿到 `unionId` 也能后端数据库里拿到用户信息。登录过程中涉及 `session_key` 和 `unionId`，于是又引出了下面的问题。

# session_key 的作用

那么，`session_key` 在登录的过程中或者登录完成后起什么作用呢？一起来看一下。

## wx.getUserInfo

首先来看一下 `wx.getUserInfo` 这个 api：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/t1fntajss4.png)

在设置 `withCredentials` 属性为 `true` 的情况下，这个 api 可以拿到 `encryptedData`，`iv` 等敏感信息，`encryptedData` 需要使用 `session_key` 进行解密，解密后可以拿到的数据如下：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/q4k5rnou5v.png)

也就是说，`session_key` 的作用之一是将小程序前端从微信服务器获取到的 `encryptedData` 解密出来，获取到 `openId` 和 `unionId` 等信息。

但是在 1.2 登录过程中可以看到开发者服务器是能够直接拿到用户的 openId 信息，而且 unionId 也是有其他获取途径，所以 session_key 在这里的作用看起来有点鸡肋。

## getPhoneNumber

`session_key` 更重要的作用大概体现在获取用户手机方面（可能还包含其他敏感信息获取 api）。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/k5hc5pefxl.png)

从文档中可以看到 `getPhoneNumber` 返回的用户数据是加密过的，只有使用 `session_key` 才能解密，而小程序前端没有 `session_key`，所以无法获取到用户的手机，只能传到开发者服务器进行处理。

# unionId 的作用，有哪些获取途径？

## UnionID 机制说明

如果公司拥有多个移动应用、网站应用、和公众帐号（包括小程序），可通过 unionid 来区分用户的唯一性，因为只要是同一个微信开放平台帐号下的移动应用、网站应用和公众帐号（包括小程序），用户的 `unionid` 是唯一的。换句话说，同一用户，对同一个微信开放平台下的不同应用，`unionid` 是相同的。

> Tip：`unionid` 用于识别同一主体下不同账号之间的用户。举例说明：就是公司有 A 订阅号，B 服务号，同一个人关注 A 和 B，会得到不同的 `OPENID`，但是会得到相同的 `unionid`。这样就可以识别到相同的用户，用于不同账号之间打通用户关系。

## UnionID 获取途径

必须有一个微信开放平台账号绑定了至少一个微信公众账号或者网站应用或者小程序，否则 `UnionID` 返回 null。绑定了开发者帐号的小程序，可以通过下面 3 种途径获取 `UnionID`。

- **方法一**：调用接口 wx.getUserInfo，从解密数据中获取 UnionID。注意本接口需要用户授权，请开发者妥善处理用户拒绝授权后的情况。

- **方法二**：如果开发者帐号下存在同主体的公众号，并且该用户已经关注了该公众号。开发者可以直接通过 wx.login 获取到该用户 UnionID，无须用户再次授权。

- **方法三**：如果开发者帐号下存在同主体的公众号或移动应用，并且该用户已经授权登录过该公众号或移动应用。开发者也可以直接通过 wx.login 获取到该用户 UnionID，无须用户再次授权。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/0xxx3wwfci.png)

# 在应用中如何保存用户登录态

保存用户登录态，一直以来都有两种解决方案：前端保存和后端保存。

## 后端保存

在 登录 ③ 中写 session 的时候可以直接设定过期时间，定期通知小程序前端重新进行登录（`wx.login`）。

## 前端保存

因为 `session_key` 存在时效性问题（毕竟是用来查看敏感信息），而小程序前端可以通过 `wx.checkSession()` 来检查 `session_key` 是否过期。所以可以通过这个来作为保存用户登录态的机制，这也是小程序文档中推荐的方法：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/q4i5au2srx.png)
