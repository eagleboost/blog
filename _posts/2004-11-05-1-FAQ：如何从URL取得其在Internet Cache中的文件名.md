---
layout:     post
title:      "FAQ：如何从URL取得其在Internet Cache中的文件名"
subtitle:   "——谨以怀念写邮件回答网友关于Internet Explorer编程问题的青春岁月"
date:       2004-11-05 01:00:00
author:     "eagleboost"
header-img: "img/post-bg-globe.jpg"
catalog: true
tags:
    - Internet Explorer编程FAQ
    - Internet Explorer编程
    - GetUrlCacheEntryInfo
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/167900)

### 问

张硕，你好，   

我目前对`IE`编程感兴趣，在网上看到了你的文章，觉得很不错。   

我用了很长时间的`MyIE`（现在叫`Maxthon`），它里面有一个功能不错，就是按住`Ctrl`键然后拖动一个图片，就可以把这个图片保存到一个默认的目录下（在设置中设）。我刚开始以为它只是把图片再下载一次，但是我拔网线后再`Ctrl`+拖鼠标，还是能够保存图片。   

我打算自己写一写，但是`IOleCommandTarget::Exec()`和`IWebBrowser2::ExecWB()`都会弹出另存为对话框。   

请问你，有什么高招能够直接把图片从`Cache`中拷出来吗？   
非常感谢！ 

2004-11-05

### 答

在`WinInet`库中`Microsoft`提供了一系列的`API`函数来操作`Internet Cache`，所以你的要求很容易满足。下面的例子给出了根据`url`取得其在`Internet`临时目录中文件名的方法。得到鼠标拖动的图片的`url`比较简单，此处不再赘述。

```c++
DWORD dwEntrySize=0;
LPINTERNET_CACHE_ENTRY_INFO lpCacheEntry;
char strTemp[80];
DWORD dwTemp;

//假设lpszUrl是图片的url
if (!GetUrlCacheEntryInfo(lpszUrl, NULL, &dwEntrySize))
{
  if (GetLastError() == ERROR_INSUFFICIENT_BUFFER)
  {
    return FALSE;
  }

  lpCacheEntry = (LPINTERNET_CACHE_ENTRY_INFO)new char[dwEntrySize];
}

if (!GetUrlCacheEntryInfo(lpszUrl, lpCacheEntry, &dwEntrySize))
{
  return FALSE;
}

//lpCacheEntry->lpszLocalFileName即是lpszUrl在缓存中的文件名
return TRUE;
```