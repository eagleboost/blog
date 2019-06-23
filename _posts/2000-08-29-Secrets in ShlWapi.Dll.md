---
layout:   post
title:  "Secrets in ShlWapi.Dll"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2000-08-14
author:   "eagleboost"
header-img: "img/post-bg-tunnel-stairs.jpg"
tags:
  - ShlWapi
  - Delphi
  - Windows编程
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106248)，原文于2000年发布[阿甘的家](http://eagleboost.myrice.com/)。

我们编程时也许遇到过这样的问题：在一个Label或是Panel上显示路径，当路径过长显示不下的时候，希望像某些安装程序拷贝文件的时候那样把路径缩短，其中部分用省略号代替，比如：

```pascal
C:/Program Files/Borland/Delphi5/Source/Rtl/Win-->C:/Program Files/Borland/.../Win
```

自己编程实现并不难，不过不知什么原因，我一直没有动手做，忽然有一天，我看到了一篇文章，于是，一切问题迎刃而解，随之而来的，竟是意想不到的收获

那是一段Visual Basic的程序，不过，我第一眼就看到了一个函数的声明：PathCompactPath

```basic
Private Declare Function PathCompactPath
  Lib "shlwapi" Alias "PathCompactPathA"(_ ByVal hDC As Long, ByVal lpszPath As String, ByVal dx As Long) As Long
```

经过几番修改，在Delphi中试验通过了，果然能够做到压缩路径的效果，但我更感兴趣的是，`ShlWapi.dll`中是不是还有不少可以用的好东东呢？打开MSDN，敲入“`ShlWapi`”一搜索，果然出现一堆（注意，是“一堆”）以“Path”开头的函数，欣喜之情，不在话下。于是我便一个个查看其功能，发现我们需要的关于路径的几乎所有功能都有相应的的函数可以调用，比如：

```pascal
PathAddBackslash、PathRemoveBackslash：//在路径后面添加或去除“/”；
PathIsDirectory、PathIsHTMLFile、PathIsPrefix、PathIsRoot、PathIsURL……
```

等等，随便试了几个，可以用，接着我又琢磨如何找出其中全部的函数声明，我知道很多动态链接库在MSDN上都有相应的头文件，这次也许不会例外。果然又被我猜中！ShlWapi.H确实存在。接下来的工作就比较烦了，花了些时间，以“查找、替换”大法为辅助，我把其中关于路径操作的函数声明做成了ShlWapi.pas（ShlWapi.H中包括几部分的函数声明：字符串、路径、注册表、注册表流、调色板，还有一个很有用的函数DllGetVirsion，完整的声明可以在这里下载）。

再来说说`PathComact`（或者叫`PathEllipsis`），`PathCompactPath`函数需要设备的HDC做参数，使用起来可能会麻烦一点，所以还有另外一个函数`PathCompactPathEx`，参数与设备无关，不过关于字符和显示宽度的换算也有些地方需要推敲，我做了一个简单的`EllipsisPanel`控件，可以作为例子。

同样的功能，也可以用`DrawText`函数来完成，参数说明在MSDN或者Delphi的Windows SDK Help中找能找到，功能比`PathCompactPath`要强，调用的时候可以选择是在路径中间省略还是在路径尾部省略（类似于资源管理器的标题栏不能显示完整路径名的时候做的处理）DFS的`dfsEllipsisPanel`就是用它做的。 