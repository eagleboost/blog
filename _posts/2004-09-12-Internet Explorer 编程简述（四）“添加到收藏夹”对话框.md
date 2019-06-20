---
layout:     post
title:      "Internet Explorer 编程简述（四）“添加到收藏夹”对话框"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-12
author:     "eagleboost"
header-img: "img/post-bg-gloden-bridge.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - 添加到收藏夹
    - 模态窗口
    - IShellUIHelper
    - DoAddToFavDlg
    - DoOrganizeFavDlg
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/102355)

### 1.概述

调用“添加到收藏夹”对话框（如下）与调用“整理收藏夹”对话框有不同之处，前者所做的工作比后者要来得复杂。将链接添加到收藏夹除了将链接保存之外，还可能会有脱机访问的设置，从IE 4.0到IE 5.0，处理的方式也发生了一些变化。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/AddToFavDlg.gif)

### 2. IShellUIHelper接口

微软专门提供了一个接口[IShellUIHelper](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa753575(v%3Dvs.85))来实现对Windows Shell API一些功能的访问，将链接添加到收藏夹也是其中之一，就是下面的`AddFavorite`函数。

```c++
HRESULT IShellUIHelper::AddFavorite(BSTR URL, VARIANT *Title);
```

实例代码如下：

```c++
void CMyHtmlView::OnAddToFavorites()
{  
  IShellUIHelper* pShellUIHelper;  
  HRESULT hr = CoCreateInstance(CLSID_ShellUIHelper, NULL, CLSCTX_INPROC_SERVER, IID_IShellUIHelper,(LPVOID*)&pShellUIHelper);

  if (SUCCEEDED(hr))
  {    
    _variant_t vtTitle(GetTitle().AllocSysString());    
    CString strURL = m_webBrowser.GetLocationURL();

    pShellUIHelper->AddFavorite(strURL.AllocSysString(), &vtTitle);pShellUIHelper->Release();  
  }
}
```

我们注意到这里的“`AddFavorite`”函数并没有像“`DoOrganizeFavDlg`”那样需要一个父窗口句柄。这也导致与在IE中打开不同，通过[IShellUIHelper](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa753575(v%3Dvs.85))接口显示出来的“添加到收藏夹”对话框是“非模态”的，有一个独立于我们应用程序的任务栏按钮，这使我们的浏览器显得非常不专业（我是个追求完美的人，这也是我的浏览器迟迟不能发布的原因之一）。于是我们很自然地想到“`shdocvw.dll`”中除了“`DoOrganizeFavDlg`”外，应该还有一个类似的函数，可以传入一个父窗口句柄用以显示模态窗口，也许就像这样：

```c++
typedef UINT (CALLBACK* LPFNADDFAV)(HWND, LPTSTR, LPTSTR);
```

事实上，这样的函数确实存在于“`shdocvw.dll`”中，那就是“`DoAddToFavDlg`”。


### 3. DoAddToFavDlg函数

“`DoAddToFavDlg`”函数也是“`shdocvw.dll`”暴露出来的函数之一，其原型如下：

```c++
typedef BOOL (CALLBACK* LPFNADDFAV)(HWND, TCHAR*, UINT, TCHAR*, UINT,LPITEMIDLIST);
```

第一个参数正是我们想要的父窗口句柄，第二和第四个参数分别是初始目录（一般来说就是收藏夹目录）和要添加的链接的名字（比如网页的Title），第三和第五个参数分别是第二和第四两个缓冲区的长度，而最后一个参数则是指向与第二个参数目录相关的item identifier list的指针(`PIDL`)。但最奇怪的是这里并没有像“`AddFavorite`”函数一样的链接URL，那链接是怎样添加的呢？答案是“手动创建”。

第二个参数在函数调用返回后会包含用户在“添加到收藏夹”对话框中选择或创建的完整链接路径名（如“`X:/XXX/mylink.url`”），我们就根据这个路径和网页的URL来创建链接，代码如下（为简化，此处省去检查"`shdocvw.dll`"是否已在内存中的代码，参见《Internet Explorer 编程简述（三）“整理收藏夹”对话框）：

 
```c++
void CMyHtmlView::OnFavAddtofav()
{  
  typedef BOOL (CALLBACK* LPFNADDFAV)(HWND, TCHAR*, UINT, TCHAR*, UINT,LPITEMIDLIST);
  HMODULE hMod = (HMODULE)LoadLibrary("shdocvw.dll");  

  if (hMod)  
  {    
    LPFNADDFAV lpfnDoAddToFavDlg = (LPFNADDFAV)GetProcAddress( hMod, "DoAddToFavDlg");   if (lpfnDoAddToFavDlg)    
    {      
      TCHAR szPath[MAX_PATH];      
      LPITEMIDLIST pidlFavorites;

      if (SHGetSpecialFolderPath(NULL, szPath, CSIDL_FAVORITES, TRUE) && (SUCCEEDED(SHGetSpecialFolderLocation(NULL, CSIDL_FAVORITES, &pidlFavorites))))  
      {        
        TCHAR szTitle[MAX_PATH];        
        strcpy(szTitle, GetLocationName());
        TCHAR szURL[MAX_PATH];        
        strcpy(szURL, GetLocationURL());
        BOOL bOK = lpfnDoAddToFavDlg(m_hWnd, szPath, sizeof(szPath)/sizeof(szPath[0]), szTitle, sizeof(szTitle)/sizeof(szTitle[0]), pidlFavorites); 
        CoTaskMemFree(pidlFavorites);
        if (bOK) 
        {
          CreateInternetShortcut( szURL, szPath, "");  //创建Internet快捷方式      
        }
      }    
    }    
    FreeLibrary(hMod);  
  }  
  return;
}
```

实现`CreateInternetShortcut`函数创建Internet快捷方式，可以用读写INI文件的方法，但更好的则是利用[IUniformResourceLocator](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/dd565673(v%3Dvs.85))接口。

```c++
HRESULT CMyHtmlView::CreateInternetShortcut(LPCSTR pszURL, LPCSTR pszURLfilename, LPCSTR szDescription, LPCTSTR szIconFile, int nIndex)
{  
  HRESULT hres;

  CoInitialize(NULL); 

  IUniformResourceLocator *pHook;

  hres = CoCreateInstance (CLSID_InternetShortcut, NULL, CLSCTX_INPROC_SERVER, IID_IUniformResourceLocator, (void **)&pHook);  
  
  if (SUCCEEDED (hres))  
  {    
    IPersistFile *ppf;    
    IShellLink *psl;
    // Query IShellLink for the IPersistFile interface 
    hres = pHook->QueryInterface (IID_IPersistFile, (void **)&ppf);  
    hres = pHook->QueryInterface (IID_IShellLink, (void **)&psl);

    if (SUCCEEDED (hres))  
    {
      // buffer for Unicode string
      WORD wsz [MAX_PATH];
      // Set the path to the shortcut target.  
      pHook->SetURL(pszURL,0);
      hres = psl->SetIconLocation(szIconFile, nIndex);

      if (SUCCEEDED (hres))  
      {    
        // Set the description of the shortcut.    
        hres = psl->SetDescription(szDescription);
        if (SUCCEEDED (hres))  
        {    
          // Ensure that the string consists of ANSI characters.    
          MultiByteToWideChar (CP_ACP, 0, pszURLfilename, -1, wsz, MAX_PATH);
          // Save the shortcut via the IPersistFile::Save member function.  
          hres = ppf->Save (wsz, TRUE);  
        }
      }
      // Release the pointer to IPersistFile.  
      ppf->Release ();  
      psl->Release ();  
    }
    // Release the pointer to IShellLink.  
    pHook->Release ();
  }  
  return hres;
}
```
 
好，上面的方法虽然麻烦一点，但总算解决了“模态窗口”的问题，使得我们的程序不至于让用户鄙视。但是问题又来了，我们发现“允许脱机使用”是Disabled的，那“自定义”也就无从谈起了，尽管90%的人都没有使用过IE提供的脱机浏览。

难道我们的希望要破灭吗？我们一方面想像调用“`AddFavorite`”函数一样的不必手动创建链接，一方面又要模态显示窗口，就像IE那样，还能自定义脱机浏览。

### 4. 脚本方式

许多网页上都会有一个按钮或链接“添加本页到收藏夹”，实际上通过下面的脚本显示模态的“添加到收藏夹”对话框将网页加入到收藏夹。

```javascript
window.external.AddFavorite(location.href, document.title);
```

这里的external对象是`WebBrowser`内置的COM自动化对象，以实现对文档对象模型（DOM）的扩展（我们也可以通过IDocHostUIHandler实现自己的扩展）.查阅MSDN可以得知external对象的的方法与IShellUIHelper接口提供的方法是一样的。我们有理由相信，IShellUIHelper提供了对`WebBrowser`内置的external对象的访问，如果在适当的地方创建IShellUIHelper接口的实例，也许调用“AddFavorite”函数显示出来的就是模态对话框了。问题是我们还没有找到这样的地方。

从上面的脚本，我们很自然地又想到另一个方法。如果能够让网页来执行上面的脚本，岂不是问题就解决了？说做就做，如下：

```c++
void CMyHtmlView::OnFavAddtofav()
{  
  CString strUrl = GetLocationURL();  
  CString strTitle = GetLocationName();  
  CString strjs = "javascript:window.external.AddFavorite('" + strUrl + "'," + "'" + strTitle + "');";  
  ExecScript(strjs);
}

void CMIEView::ExecScript(CString strjs)
{  
  CComQIPtr pHTMLDoc = (IHTMLDocument2*)GetHtmlDocument();  
  if ( pHTMLDoc != NULL )  
  {    
    CComQIPtr pHTMLWnd;    
    pHTMLDoc->get_parentWindow( &pHTMLWnd );    
    if ( pHTMLWnd != NULL )    
    {      
      CComBSTR bstrjs = strjs.AllocSysString();
      CComBSTR bstrlan = SysAllocString(L"javascript");
      VARIANT varRet;
      pHTMLWnd->execScript(bstrjs, bstrlan, &varRet);
    }
  }
}
```

先从`CHtmlView`获得文档的父窗口window对象的指针，再调用其方法execScript来执行脚本（事实上可以执行任意的脚本）。试验发现，这个方法非常有效，不仅窗口是模态的，而且不需要手动创建链接，更重要的是“允许脱机使用”和“自定义”按钮也可以用了。

### 5. 问题仍旧没有解决

执行脚本的方式看起来有效，可一旦我们的程序实现了[IDocHostUIHandler](https://docs.microsoft.com/en-us/cpp/atl/reference/idochostuihandlerdispatch-interface?view=vs-2019)接口对`WebBrowser`进行高级控制，就会发现一旦执行的脚本包含有对“external”对象的调用，就会出现“缺少对象”的脚本错误。原因是当`MSHTML`解析引擎（并非`WebBrowser`）检查到宿主实现了[IDocHostUIHandler](https://docs.microsoft.com/en-us/cpp/atl/reference/idochostuihandlerdispatch-interface?view=vs-2019)接口，就会调用其`GetExternal`方法以获得一个用以扩展DOM的自动化接口的引用。

```c++
HRESULT IDocHostUIHandler::GetExternal(IDispatch **ppDispatch）
```

但有时候我们并没有想要扩展DOM，同时我们还希望`WebBrowser`使用它自己的DOM扩展。糟糕的是`GetExternal`方法的文档中说这种情况下必须把ppDispatch设置为NULL，换句话说，`WebBrowser`连它内置的`external`对象也不用了，那我们的`window.external.AddFavorite`就变得无处为家了。
我曾多方尝试将`WebBrowser`内置的external对象找出来，虽然都没有成功，但是解决问题的方法却被我找到了。

### 6. 完美的方案

`WebBrowser`内置的`external`对象我们虽然找不到，但它肯定存在，我们只要想办法让`WebBrowser`自己完成对其调用即可。实现非常简单，找到`WebBrowser`中包含的“`Internet Explorer_Server`”窗口的句柄，发一个消息就完成了。下面的代码中假设m_hWndIE就是“`Internet Explorer_Server`”窗口的句柄。

```c++
#define ID_IE_ID_ADDFAV 2261::SendMessage( m_hWndIE, WM_COMMAND, MAKEWPARAM(LOWORD(ID_IE_ID_ADDFAV), 0x0), 0 );
```

试一试成果，是不是和在Internet Explorer中选择“添加到收藏夹”的效果一模一样。

至于为什么这样做，后续文章再说。


### 参考资料：
+ MSDN: [Adding Internet Explorer Favorites to Your Application](http://www.microsoft.com/mind/0798/favorites.asp)
+ MSDN: [IShellUIHelper Interface](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa753575(v%3Dvs.85))
+ MSDN: external Object
+ MSDN: [IDocHostUIHandler Interface](https://docs.microsoft.com/en-us/cpp/atl/reference/idochostuihandlerdispatch-interface?view=vs-2019)
