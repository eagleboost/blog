---
layout:     post
title:      "Internet Explorer 编程简述（一）WebBrowser还是WebBrowser_V1"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-03
author:     "eagleboost"
header-img: "img/post-bg-globe.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - WebBrowser
    - WebBrowser_V1
    - NewWindow
    - NewWindow2
    - NewWindow3
    - INewWindowManager
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/95964)

你的机器上总是存在着“两”个WebBrowser，一个叫WebBrowser，另一个叫WebBrowser_V1，其CLASSID如下：

```c++
  CLASS_WebBrowser: TGUID = '{8856F961-340A-11D0-A96B-00C04FD705A2}';  
  CLASS_WebBrowser_V1: TGUID = '{EAB22AC3-30C1-11CF-A7EB-0000C05BAE0B}';
```

它们分别对应的接口是`IWebBrowser2`和`IWebBrowser`。问题是我们该用哪一个呢？按照微软的推荐，应该尽量使用前者，因为后者是为兼容`Internet Explorer 3.x`而保留的（尽管它能够响应来自Internet Explorer 3.x、4.x、5.x、6.x的事件），相应的`IWebBrowser`和`IWebBrowserApp`接口也应抛弃。
由于`Internet Explorer 3.x`年代久远，导致`WebBrowser_V1`提供的事件少得可怜，但值得一提的是它提供的两个事件`OnNewWindow`和`OnFrameBeforeNavigate`有着与`OnBeforeNavigate`几乎相同的参数：

```c++
OnBeforeNavigate(  
  BSTR URL,   
  long Flags,   
  BSTR TargetFrameName,   
  VARIANT* PostData,   
  BSTR Headers,   
  BOOL FAR* Cancel)

OnNewWindow(  
  BSTR URL,   
  long Flags,   
  BSTR TargetFrameName,   
  VARIANT* PostData,   
  BSTR Headers,   
  BOOL FAR* Processed)
  
OnFrameBeforeNavigate(  
  BSTR URL,   
  long Flags,   
  BSTR TargetFrameName,   
  VARIANT* PostData,   
  BSTR Headers,   
  BOOL FAR* Cancel)
```

所以使用`WebBrowser_V1`使得我们的浏览器在有新窗口打开时能够轻易捕捉到其URL及相关的数据，如果将Processed设置为TRUE，则可取消新窗口的弹出。同样，处理Frame也比在WebBrowser中来得容易。

但`WebBrowser_V1`的致命弱点是它不支持高级接口，如[IDocHostUIHandler](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa753260(v%3Dvs.85))，即便我们实现了[IDocHostUIHandler](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa753260(v%3Dvs.85))接口，也不会被`WebBrowser_V1`调用。所以希望在自己的浏览器中实现XP的界面主题、扩展IE的DOM（Document Object Model）等高级控制的话，就肯定不能选择`WebBrowser_V1`了。

处理新窗口实在是很麻烦的一件事，不知道微软为什么在新版本的`OnNewWindow2`事件中去掉了URL这样的参数，而且`OnNewWindow2`事件不能完全捕捉到所有的新窗口打开。但如果安装了Windows XP SP2的话，好处又回来了。

Windows XP SP2对`Internet Explorer 6`作了升级，并且提供了一个新的事件`OnNewWindow3`，它在`OnNewWindow2`事件之前发生，也包含了让我们能够加以过滤处理的新窗口的URL等参数，再加上[INewWindowManager](https://docs.microsoft.com/en-us/windows/desktop/api/shobjidl_core/nn-shobjidl_core-inewwindowmanager)接口，就是实现Windows XP SP2中过滤广告窗口功能的基础。
