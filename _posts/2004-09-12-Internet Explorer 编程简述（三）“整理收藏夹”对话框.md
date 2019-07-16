---
layout:     post
title:      "Internet Explorer 编程简述（三）“整理收藏夹”对话框"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-1214:37:00
author:     "eagleboost"
header-img: "img/post-bg-flash-car.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - 添加到收藏夹
    - 整理收藏夹
    - DoAddToFavDlg
    - DoOrganizeFavDlg
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/102007)

关于Internet Explorer的收藏夹，比较常见的两个问题就是调用“`整理收藏夹`”对话框和“`添加到收藏夹`”对话框。调用的方法有多种，但其中还是有些值得讨论的地方。

### 1. 整理收藏夹

调用“整理收藏夹”对话框（如下），基本上来说都用的是同一个方法，即调用“`shdocvw.dll`”中的“`DoOrganizeFavDlg`”函数，把父窗口句柄和收藏夹路径作为参数传入即可。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/OrganizeFavDlg.gif)

### 2. 代码

代码实例如下所示，值得注意的是对“shdocvw.dll”的处理，为避免重复调用，应该先检查其是否已经在内存中。

```c++
void CMyHtmlView::OnFavOrganizefav()
{  
  typedef UINT (CALLBACK* LPFNORGFAV)(HWND, LPTSTR);

  bool bResult = false;
  bool bShouldUnload = false;

  HMODULE hMod = ::GetModuleHandle( _T("shdocvw.dll") );
  if (hMod == NULL)//如果"shdocvw.dll"尚未载入则载入之  
  {    
    hMod = ::LoadLibrary( _T("shdocvw.dll") );
    if (hMod == NULL)    
    {      
      MessageBox( _T("The dynamic link library ShDocVw.DLL cannot be found."), _T("Error"), MB_OK | MB_ICONSTOP );      
      return;    
    }
    bShouldUnload = true; //原先不在内存，所以用完也应该释放掉
  }

  LPFNORGFAV lpfnDoOrganizeFavDlg = (LPFNORGFAV)::GetProcAddress(hMod, "DoOrganizeFavDlg");
  if (lpfnDoOrganizeFavDlg == NULL)
  {      
    MessageBox( _T("The entry point DoOrganizeFavDlg cannot be found/n"), _T("in the dynamic link library ShDocVw.DLL."), _T("Error"), MB_OK | MB_ICONSTOP );

    if (bShouldUnload)
    {
      ::FreeLibrary(hMod);
    }  
    return;    
  }

  TCHAR szPath [ MAX_PATH ];    
  HRESULT hr;
  hr = ::SHGetSpecialFolderPath(m_hWnd, szPath, CSIDL_FAVORITES, TRUE);
  if (FAILED(hr))    
  {      
    MessageBox( _T("The path of the Favorites folder cannot be found."), _T("Error"), MB_OK | MB_ICONSTOP );

    if (bShouldUnload)
    {
      ::FreeLibrary(hMod);
    }  
    return;    
  }

  bResult = (*lpfnDoOrganizeFavDlg)(m_hWnd, szPath) ? true : false;

  if (bShouldUnload)
  {
    ::FreeLibrary(hMod);
  }  
}

```

### 3. 讨论

实际上，从“`DoOrganizeFavDlg`”函数的原型声明我们可以看到，由于需要一个路径，所以“`整理收藏夹`”对话框其实不仅可以用来整理收藏夹，还可以整理磁盘上的目录。而且所谓的整理也不过是提供了一个对话框使用户用起来比较方便而已，和直接在资源管理器中整理没有实质性的差别。因此调用“`整理收藏夹`”对话框的方法从IE4.0开始就没有变过，除了对话框的布局有所改变。
 
```c++
typedef UINT (CALLBACK* LPFNORGFAV)(HWND, LPTSTR);
```

IE 4.0的“整理收藏夹”对话框
 
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/OrganizeFavDlg_IE_4.gif)

IE 4.0的“整理收藏夹”对话框（原先的设计）
 
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/OrganizeFavDlg_IE_4_Old.gif)


“添加到收藏夹”就不同了，“DoAddToFavDlg”函数不再像“DoOrganizeFavDlg”函数一样对所有IE的版本都适用。
 
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/AddToFavDlg.gif)
 
### 参考资料：
+ MSDN: [Adding Internet Explorer Favorites to Your Application](http://www.microsoft.com/mind/0798/favorites.asp)
