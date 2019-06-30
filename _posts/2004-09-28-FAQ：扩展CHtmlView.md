---
layout:     post
title:      "FAQ：扩展CHtmlView"
subtitle:   "——谨以怀念写邮件回答网友关于Internet Explorer编程问题的青春岁月"
date:       2004-09-28
author:     "eagleboost"
header-img: "img/post-bg-color-feather.jpg"
catalog: true
tags:
    - Internet Explorer编程FAQ
    - Internet Explorer编程
    - CHtmlView
    - IDocHostShowUI
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/119190)

### 问

我想在`CHtmlView`中提供`IDocHostShowUI`接口，但不知道该如何提供此接口。查了很多资料，好象必须同时实现`IOleDocumentSite`和`IOleClientSite`接口，这就必须要重载`CHtmlView::CreateControlSite()`，我就没有办法使用`CHtmlView`中默认的`ControlSite`实现了。请问我该怎么做才能方便地实现这个接口？

2004-09-27

### 答

扩展WebBrowser在VC6和以后版本（VC7.0/VC7.1）中有所不同。    

VC6中`CHtmlView`的实现非常简单，只是调用`CreateControl`创建了一个`WebBrowser Control`，没有额外的高级控制。扩展（比如实现`IDocHostUIHandler`）只有自己实现一个自定义的`Control Site`来完成，并且需要一个`1`COccManager`1`来管理（MSDN的知识库中有范例，在我的Blog上也会讲到）。    

VC7.x的MFC实现有所不同，为在窗口中嵌入`ActiveX Control`提供了更好的支持，如重载CWnd的`CreateControl`Site方法就可以实现自定义的`Ole Control Site`。VC7.x中的`CHtmlView`在此基础上已经提供了对`IDocHostUIHandler`的扩展，下面是`ChtmlView`的`CreateControlSite`方法，在其中创建了一个`CHtmlControlSite`。

```c++
BOOL CHtmlView::CreateControlSite(COleControlContainer* pContainer,    COleControlSite** ppSite, UINT /* nID */, REFCLSID /* clsid */)
{  
  ASSERT(ppSite != NULL);  
  *ppSite = new CHtmlControlSite(pContainer);  
  return TRUE;
}
```

因此，要实现`IDocHostShowUI`等扩展，需要从`COleControlSite`或`CHtmlControlSite`派生一个自定义的`Control Site`（实现的代码可以参照viewhtml.cpp中的`CHtmlControlSite`），然后在自己的HtmlView中照上面的例子覆盖`CHtmlView::CreateControlSite`即可，所不同的是不再需要手动给出`COccManager`。