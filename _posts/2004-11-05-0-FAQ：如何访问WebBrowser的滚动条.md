---
layout:     post
title:      "FAQ：如何访问WebBrowser的滚动条"
subtitle:   "——谨以怀念写邮件回答网友关于Internet Explorer编程问题的青春岁月"
date:       2004-11-05 00:48:00
author:     "eagleboost"
header-img: "img/post-bg-leaves.jpg"
catalog: true
tags:
    - Internet Explorer编程FAQ
    - Internet Explorer编程
    - IHTMLElement2
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/167884)

### 问

我使用`webbrowser`控件，但是想用自己的滚动条，但不知如何得到`webbrowser`中滚动条的长度，怎么办？谢谢！！

2004-10-24

### 答

抱歉拖了很久才回复你的问题。

`WebBrowser`的滚动条不是一般的`Windows`滚动条，用`GetScrollPos`或`GetScrollInfo`等`API`是不能访问的。下面的代码演示了在`VC`中如何通过`HTML`接口来访问浏览器的滚动条。

```c++
  HRESULT hr;    
  IDispatch *pDisp = GetHtmlDocument();    
  ASSERT(pDisp); //if NULL, we failed        

  // 获得Html文档指针    
  IHTMLDocument2 *pDocument = NULL;    
  hr = pDisp->QueryInterface( IID_IHTMLDocument2, (void**)&pDocument);    
  ASSERT(SUCCEEDED(hr));    
  ASSERT(pDocument);    

  IHTMLElement *pBody = NULL;    
  hr = pDocument->get_body( &pBody);    
  ASSERT(SUCCEEDED(hr));    
  ASSERT(pBody);    

  // 从body获得IHTMLElement2接口指针，用以访问滚动条    
  IHTMLElement2 *pElement = NULL;    
  hr = pBody->QueryInterface(IID_IHTMLElement2, (void**)&pElement);    
  ASSERT(SUCCEEDED(hr));    
  ASSERT(pElement);    

  // 向下滚动100个像素    
  pElement->put_scrollTop( 100);         

  // 获得文档真正的高度，不是可见区域的高度    
  long scroll_height;     
  pElement->get_scrollHeight( &scroll_height);    

  // 获得文档真正的宽度，不是可见区域的宽度    
  long scroll_width;     
  pElement->get_scrollWidth( &scroll_width);    

  // 获得滚动条位置，从顶端开始    
  long scroll_top;    
  pElement->get_scrollTop( &scroll_top);
```