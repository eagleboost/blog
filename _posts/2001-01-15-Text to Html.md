---
layout:   post
title:  "Text to Html"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2000-08-14
author:   "eagleboost"
header-img: "img/post-bg-food.jpg"
tags:
  - ShlWapi
  - Delphi
  - Windows编程
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106253)，原文于2001年发布[阿甘的家](http://eagleboost.myrice.com/)。

相信大家看到过Html<->Text转换的软件，自己编过这类转换软件的朋友可能也不少，工作中会也有可能会遇到。`Html To Text`无需多说，我在[《TWebBrowser编程简述》](https://eagleboost.com/2001/02/07/TWebBrowser%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0/)一文中已有提及，我自己就过一个`HtmlToText`的软件，自我感觉界面还算不错（也许近期会发布）。方法很简单，使用来自`IE & Delphi`的`TIEParser`或者`UILess`单元，以IE作为HTML解码引擎，取得`IHTMLDocument2`接口，之后`Body.OuterText`中装的就是网页中的文本了，此处不再赘述。

Text To Html呢？看来很简单，把文本放入一个`TStringList`，写几个函数，一行行地加上标记，遇到特殊字符转化一下就OK了。真的是这样吗？看了下面的例子就知道要做好并非如此简单。

一个网上查询系统，有些文本文件每天都更新，需要原封不动地显示到IE的浏览器窗口上，由于不知道网页中对Tab如何处理，我就用8个空格来代替它，这样，对齐的时候麻烦就来了。因为文本文件来自不同的人员和不同的编辑器。有的人喜欢用Tab对齐，有的人喜欢用空格对齐，更有的一会儿用Tab一会儿用空格，要知道不同的编辑器对Tab的定义可能不同，有的是8个空格的位置，有的又只有5个，还有的（像Delphi）可以自定义。总之用我的方法显示出来是无法对齐的。

怎么办？我想了各种方法，工作量都很大，而且不一定能起到很好的效果。把文本放在一个`TextArea`里面倒是没有问题，但显示出来显得很不规范。于是当时的解决办法是——放弃。

很多天以后，我才又再次了解到了自己的浅薄，其实`HTML`语法规则指定时早已考虑到了这种情况，所以提供了一对标记“\<PRE\>”和“\<\/PRE\>”，在此标记包围中的文本均为格式化文本，空格和回车都会保留。于是Text To Html就变得很简单了，需要考虑的只是“<”符号，两步就搞定：

1) 使用`StringReplace`函数将需要转换的文本中所有的“<”符号替换为“\&lt;”，避免文本中出现以“<>”包围的HTML标记文字的时候显示出错。`StringReplace`函数声明如下：

```pascal
type 
  TReplaceFlags = set of (rfReplaceAll, rfIgnoreCase);
function StringReplace(const S, OldPattern, NewPattern: string;Flags: TReplaceFlags): string;

2) 在文本首尾分别加上“<PRE>”和“</PRE>”即可（不要忘了一个空的网页需要的最基本元素）。 

试一试就知道，非常简单。