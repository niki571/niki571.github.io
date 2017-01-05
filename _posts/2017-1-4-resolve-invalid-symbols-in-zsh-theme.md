---
layout: post
title:  "怎样解决agnoster.zsh-theme字体不显示问题"
date:   2017-01-04 10:23:30 +0800
categories: [Tech]
excerpt:
tags:
  - CN
  - Tools
---

### 发现

这几天逛GitHub的时候突然发现一款很妖艳的皮肤 [wild-cherry](https://github.com/mashaal/wild-cherry) 见下图，感觉很炫酷！有Zsh, iTerm, Sublime, Atom版，甚至还有Slack!立刻决定给iTerm2换上。

![wild-cherry](../../assets/images/wild_cherry.gif)

但是项目里只有iTerm2的颜色包，要达到图片里的样式和emojis效果，还需要配置Zsh theme。

![iterm-warning](../../assets/images/iterm_warning.png)

### 错误

而如何正确配置Zsh theme呢？

按照文档里面写的找到[Powerline-patched font](https://github.com/powerline/fonts)，clone到本地运行`./install.sh`安装字体后，在iTerms里面运行`echo "\ue0b0 \u00b1 \ue0a0 \u27a6 \u2718 \u26a1 \u2699"`, 正常的结果应该是第一张图，如果出现第二张图中无法显示的字符就有问题了。

![characters](../../assets/images/characters.png)

![wrong-characters](../../assets/images/wrong_characters.png)

### 解决

我查了很多资料，终于找到解决方法！

`iTerm2`=>`Preferences...`=>`Profiles`=>`Text`=>`Change Font`

按下图勾选:

![characters](../../assets/images/iterm_setting.png)

最后一定要记得取消勾选下图：

![characters](../../assets/images/uncheck_option.png)

### 完成

撒花！！！

*因为agnoster这个theme更有名一点更具有广泛认知度，但也有同样的问题，所以本文取名agnoster而不是wild-cherry。*