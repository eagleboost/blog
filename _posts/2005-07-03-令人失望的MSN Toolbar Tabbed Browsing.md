---
layout:   post
title:  "Sigh，令人失望的MSN Toolbar Tabbed Browsing"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2005-07-03 23:16:00
author:   "eagleboost"
header-img: "img/post-bg-cave.jpg"
catalog: true
tags:
  - MSN Toolbar
  - Tabbed Browsing
  - Internet Explorer
  - UX
  - 用户体验
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2005年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/411923)


微软输在起跑线上不是一次两次了，不过这次在`MSN Toolbar`上输得比较难看，不仅输给了其他工具条，也输给了自己。

[MSN Toolbar](https://www.microsoft.com/en-us/download/details.aspx?id=32987)起先就不如[Google Toolbar](http://www.google.com/intl/zh-CN/toolbar/ie/index.html)好用，早先的版本工具条按钮甚至不支持`XP Theme`！让人很难相信是微软自己开发出来的。

`Tabbed Browsing`似乎已成了众望所归的浏览器功能之一，作为对Firefox的回击，IE7.0也确定要提供这个Feature，按照惯例，这应当是个非常值得期待的Goal，在“[the Microsoft Internet Explorer Weblog](http://blogs.msdn.com/ie/)”上也有不时会有文章提到当前的一些实现上的问题。

其实早就有第三方的工具（如`WebTools`）为IE提供多窗口浏览的功能了，除了在一些小问题的处理上有些问题外，使用还算顺手。`MSN Toolbar`的新版本也提供了这一功能，不过试用之下，我的感觉竟然是——还不如我自己写一个，甚至远远比不上`WebTools`。

我起先以为`MSN Toolbar`所提供的`Tabbed Browsing`，多少应该算作是IE多窗口浏览的一个“预览”吧，可能会不尽如人意，但以微软对于产品易用性上的要求，应该不会差到哪里去。然而这次的`MSN Toolbar`却实在是教人大跌眼镜。下面我列举一下`MSN Toolbar`设计上的几个问题：

1) 切换Tab的时候屏幕的闪烁直教人想把屏幕砸掉，并且原本最大化的窗口会变成大小接近全屏幕大小的“Normal”窗口（搞开发的朋友应该知道全屏的窗口大小和屏幕大小是不一样的，比如1024x768的屏幕，全屏窗口的大小是“(4,-4)-1028 ,772) 1032x776”）。

2) 自定义工具条按钮时按钮之间的分隔线（Separator）会有两条挨着的情况（这样的easy的问题也会出现？？）。

3) “Options”对话是非模态的（难道有其他考虑？）。
   
4) 最右边有按钮超出窗口显示范围时显示的“>>”（`Chevron`），其下拉的Tool Menu不是平板边缘（IE本身工具条的`Chevron`下拉菜单是平板的），而是一个看起来非常拙劣的`Thick Panel`。

上面列举的第一条就足以把`MSN Toolbar`排除第一流的行列，更不用说具备全部四个毛病了。以我的眼光来看，这根本还是一个最多刚刚进入beta测试的版本，不知道这次微软怎么想的，居然就发布了:(。