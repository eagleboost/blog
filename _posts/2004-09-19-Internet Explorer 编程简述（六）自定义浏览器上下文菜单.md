---
layout:     post
title:      "Internet Explorer 编程简述（六）自定义浏览器上下文菜单"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-19
author:     "eagleboost"
header-img: "img/post-bg-dream.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - Custom Context Menu
    - ShowContextMenu
    - IDocHostUIHandler
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/109873)

### 1. 概述

Internet Explorer提供了非常开发的接口，使开发人员不仅可以把其浏览器核心嵌入应用程序，还可以通过各种接口以实现更深层的控制。本文就将介绍对浏览器进行高级控制的话题之一——自定义上下文菜单。

### 2. 最简单的情况

自定义的IE及WebBrowser的上下文菜单，最简单的方式就是在注册表的HKEY_CURRENT_USER/Software/Microsoft/Internet Explorer/MenuExt下添加自定义的键值，步骤如下：

1) 添加一个新的键，其名称即为将来显示在上下文菜单中的菜单项名称，如：　HKEY_CURRENT_USER/Software/Microsoft/Internet Explorer/MenuExt/&Google Search
   
2) 将新增的键的默认值设置为一个包含脚本的网页的URL（或文件路径全名），该网页中的脚本将在用户点击上下文菜单中的“Google Search”后被浏览器执行。

3) 在新增的键下还可以新建一个二进制值Contexts，用以指定我们新增的菜单项针对特定的网页对象是否出现，其取值可以是如下值的组合（逻辑或）

| Context       | Value |
| --------      | :--- |
| Default       | 0x1  |
|Images         | 0x2  |
|Controls       | 0x4  |
|Tables         | 0x8  |
|Text selection | 0x10 |
|Anchor         | 0x20 |

4) 还可以建立一个DWORD类型的Flags项并将其值设置为0x01，这将使得前述脚本在一个模态窗口中执行，就好像是通过window.showModalDialog调用的，但不同的是在脚本中仍然可以访问window对象。
   
5) 实例脚本如下：

> 脚本缺失

通过修改注册表自定义菜单的方法适用于Internet Explorer和WebBrowser，也具有良好的扩展性。但我们如果希望执行的是不仅仅是脚本，二是自己的程序中代码，这种方法就不适用了。

### 3. 使用完全自定义的菜单

1) `IDocHostUIHandler`接口提供了一个`ShowContextMenu`方法，在需要显示上下文菜单之前，MSHTML引擎就会调用实现了`IDocHostUIHandler`接口的宿主程序的`ShowContextMenu`方法。

```c++
HRESULTIDocHostUIHandler::ShowContextMenu(
    DWORD dwID,
    POINT *ppt,
    IUnknown *pcmdtReserved,
    IDispatch *pdispReserved
);
```

dwID参数的意义与Contexts的组合类似；ppt为菜单的弹出点屏幕坐标；pcmdtReserved接口指向IOleCommandTarget接口，可用于检测网页对象的状态和执行命令等操作。pdispReserved在IE5以上版本中指向的是网页对象的IDispatch接口，用以区分不同对象，比如我们可以这样来获得网页对象的指针：

```c++
IHTMLElement *pElem;
HRESULT hr;
hr = pdispReserved->QueryInterface(IID_IHTMLElement, (void**)pElem);
if (SUCCEEDED (hr)) 
{
  BSTR bstr;    
  pElem->get_tagName(bstr);    
  USES_CONVERSION;    
  ATLTRACE("TagName:%s/n", OLE2T(bstr));    
  SysFreeString(bstr);    
  pElem->Release();
}
```

如果我们在该方法中返回S_OK，则告诉MSHTML我们将使用自己的菜单（界面），如果是S_FALSE，则弹出默认的菜单。

2) 实现

理清楚之后，实现起来非常简单，和弹出一般的菜单没什么两样，举例如下，显示主框架的“文件菜单”：

```c++
HRESULT CMyHtmlView::OnShowContextMenu(DWORD dwID, LPPOINT ppt, IUnknown * pcmdtReserved, IDispatch *)
{  
  // 载入主菜单  
  HMENU hMenuParent = ::LoadMenu( ::AfxGetInstanceHandle(), MAKEINTRESOURC(IDR_MAINFRAME );  
  if (hMenuParent)  
  {    
    HMENU hMenu = ::GetSubMenu( hMenuParent, 0 ); // 取得“文件”子菜单    
    if (hMenu)    
    {      
      // 显示菜单      
      TrackPopupMenuEx( hMenu, TPM_LEFTALIGN | TPM_TOPALIGN, ppt->x, ppt->y,       ::AfxGetMainWnd()->m_hWnd, NULL );    
    }    
    ::DestroyMenu( hMenuParent );  
  }  
  return S_OK;
}
```

### 4. 自定义标准上下文菜单

1) 原理

更多的时候我们希望能在浏览器原来菜单的基础上作一些修改，如删掉“查看源文件”，添加自己的菜单项，等等，而不是完全不要原始的菜单，怎么办呢？借助MSDN提供的例子，我们来看看：

```c++
HRESULT CBrowserHost::ShowContextMenu(DWORD dwID, POINT *ppt,IUnknown *pcmdTarget,IDispatch *pdispObject) 
{  
  #define IDR_BROWSE_CONTEXT_MENU 24641  
  #define IDR_FORM_CONTEXT_MENU 24640  
  #define SHDVID_GETMIMECSETMENU 27  
  #define SHDVID_ADDMENUEXTENSIONS 53  
  
  HRESULT hr;  
  HINSTANCE hinstSHDOCLC;  
  HWND hwnd;  HMENU hMenu;  
  CComPtr spCT;  
  CComPtr spWnd;  
  MENUITEMINFO mii = {0};  
  CComVariant var, var1, var2;  
  
  hr = pcmdTarget->QueryInterface(IID_IOleCommandTarget, (void**)&spCT);  
  hr = pcmdTarget->QueryInterface(IID_IOleWindow, (void**)&spWnd);  
  hr = spWnd->GetWindow(&hwnd);//取得浏览器窗口句柄  
  hinstSHDOCLC = LoadLibrary(TEXT("SHDOCLC.DLL"));  
  if (hinstSHDOCLC == NULL)  
  {    
    // Error loading module -- fail as securely as possible    
    return;  
  }  
  hMenu = LoadMenu(hinstSHDOCLC, MAKEINTRESOURCE(IDR_BROWSE_CONTEXT_MENU));  
  hMenu = GetSubMenu(hMenu, dwID);  
  // Get the language submenu  
  hr = spCT->Exec(&CGID_ShellDocView, SHDVID_GETMIMECSETMENU, 0, NULL, &var);  
  mii.cbSize = sizeof(mii);  
  mii.fMask = MIIM_SUBMENU;  
  mii.hSubMenu = (HMENU) 
  var.byref;  
  // Add language submenu to Encoding context item  
  SetMenuItemInfo(hMenu, IDM_LANGUAGE, FALSE, &mii);  
  // Insert Shortcut Menu Extensions from registry  
  V_VT(&var1) = VT_INT_PTR;  
  V_BYREF(&var1) = hMenu;  
  V_VT(&var2) = VT_I4;  
  V_I4(&var2) = dwID;  
  hr = spCT->Exec(&CGID_ShellDocView, SHDVID_ADDMENUEXTENSIONS, 0, &var1, &var2);  
  // Remove View Source  
  DeleteMenu(hMenu, IDM_VIEWSOURCE, MF_BYCOMMAND);//删除“查看源文件”菜单项  
  // Show shortcut menu  
  // TPM_RETURNCMD表示返回用户选择的菜单命令ID
  int iSelection = ::TrackPopupMenu(hMenu, TPM_LEFTALIGN | TPM_RIGHTBUTTON | TPM_RETURNCMD, ppt->x, ppt->y, 0, hwnd, (RECT*)NULL);  
  // Send selected shortcut menu item command to shell  
  LRESULT lr = ::SendMessage(hwnd, WM_COMMAND, iSelection, NULL);//发送到Internet Explorer_Server进行内部处理。  
  FreeLibrary(hinstSHDOCLC);  
  return S_OK;
}
```

从上面的例子我们看出，基本的方法就是根据“`shdoclc.dll`”文件中的菜单资源建立菜单，再通过来自pcmdTarget的`IOlcCommandTarget`接口获得“编码”菜单以及HKEY_CURRENT_USER/Software/Microsoft/Internet Explorer/MenuExt下的定义扩展菜单，然后以TPM_RETURNCMD标志调用`TrackPopupMenu`或`TrackPopupMenuEx`弹出菜单，并将返回的菜单命令ID教给浏览器窗口进行处理。这种方法可以调用许多通过浏览器无法直接调用的命令和对话框（参阅：[《Internet Explorer 编程简述（五）调用IE隐藏的命令》](https://eagleboost.com/2004/09/16/Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E4%BA%94-%E8%B0%83%E7%94%A8IE%E9%9A%90%E8%97%8F%E7%9A%84%E5%91%BD%E4%BB%A4-%E4%B8%AD%E6%96%87%E7%89%88/)）。

所以，我们只需要在弹出菜单之前做一些自定义操作即可达到修改默认菜单的目的，如上面代码中就用删除了“查看源文件”菜单项。

2) 问题

如果我们不仅仅是删除默认的菜单项或是修改了默认的菜单项，还添加了自己的菜单项，会出现什么情况呢？由于使用了类似于MFC中UpdateUI的机制，遇到不认识的CommandID，浏览器就会将其状态设置为Disabled，所以我们自己添加的菜单是无法被选择的。

可以想到，如果把菜单状态设置为Enabled，并通过`TPM_RETURNCMD`标志调用`TrackPopupMenu`或`TrackPopupMenuEx`，再把返回的命令ID教给合适的窗口（如主框架窗口）去处理不就行了。关键点就在于如何把菜单状态设置为Enabled。

3) 实现
   
解决的办法是截获`WM_INITMENUPOPUP`消息，在菜单创建以后，尚未显示之前修改菜单项状态，那浏览器就没有办法了。截获`WM_INITMENUPOPUP`消息则可使用子类化（subclass）的技术，前面通过IOleWindow接口我们得到了浏览器窗口的句柄hwnd，则可以这样做：

```c++
HMENU g_hPubMenu = NULL;
WNDPROC g_lpPrevWndProc = NULL;

LRESULT CALLBACK CustomMenuWndProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{  
  if (uMsg == WM_INITMENUPOPUP)  
  {    
    if (wParam == (WPARAM) g_hPubMenu)    
    {      
      ::EnableMenuItem( 自定义的菜单命令ID, MF_ENABLED | MF_BYCOMMAND );      ::CheckMenuItem( 自定义的菜单命令ID, MF_BYCOMMAND);      
      return 0;    
    }  
  }
  return CallWindowProc(g_lpPrevWndProc, hwnd, uMsg, wParam, lParam);
}

HRESULT CMyHtmlView::OnShowContextMenu(DWORD dwID, LPPOINT ppt, LPUNKNOWN pcmdtReserved, LPDISPATCH pdispReserved)
{
  //浏览器菜单句柄保存在g_hPubMenu中
  //......
  // subclass浏览器窗口
  g_lpPrevWndProc = (WNDPROC)::SetWindowLong(hwnd, GWL_WNDPROC, (LONG)CustomMenuWndProc);
  //m_SubclassWnd.SubclassWindow( hwnd ); //MFC中用此方法更简便

  // Show shortcut menu
  int iSelection = ::TrackPopupMenu(hSubMenu, TPM_LEFTALIGN | TPM_RIGHTBUTTON | TPM_RETURNCMD, ppt->x, ppt->y, 0, hwnd, (RECT*)NULL);
  // Unsubclass浏览器窗口
  ::SetWindowLong(hwnd, GWL_WNDPROC, (LONG)g_lpPrevWndProc);g_lpPrevWndProc = NULL;
  //m_SubclassWnd.UnsubclassWindow();

  if (iSelection == 自定义的菜单命令ID )
  { 
    ::SendMessage( ::AfxGetMainWnd()->m_hWnd, WM_COMMAND, MAKEWPARAM(LOWORD(lResult), 0x0), 0 );
  }
  else
  {
    LRESULT lr = ::SendMessage(hwnd, WM_COMMAND, iSelection, NULL);
  }
  ......
}
```

在MFC中则更为方便，从CWnd继承一个窗口类，假设为`CWebBrowserSubclassWnd`，为CMyHtmlView添加一个`CWebBrowserSubclassWnd`类型的成员变量m_SubclassWnd，在子类化和去除子类化时调用m_SubclassWnd.SubclassWindow( hwnd )和m_SubclassWnd.UnsubclassWindow()即可。相应的`WM_INITMENUPOPUP`消息处理函数如下所示：

```c++
void CWebBrowserSubclassWnd::OnInitMenuPopup(CMenu* pPopupMenu, UINT nIndex, BOOL bSysMenu) 
{  
  CWnd::OnInitMenuPopup(pPopupMenu, nIndex, bSysMenu);
  pPopupMenu->EnableMenuItem( 自定义的菜单命令ID, MF_ENABLED | MF_BYCOMMAND );  pPopupMenu->CheckMenuItem( 自定义的菜单命令ID, MF_BYCOMMAND);
}
```

下面的图片显示了将“文字大小”菜单项添加到“编码”菜单项的下面的效果。

### 5. 新的问题

看完上面的代码，我们又自然地想到浏览器编程中的另一个问题，那就是“编码”菜单。我们指定，手动建立一个“编码”菜单是比较麻烦的事，而且很难做到与浏览器上下文菜单上的“编码”菜单一样的效果。何不使用上述的方法让浏览器自己建立“编码”菜单和处理相应的命令呢？

具体实现请看下一篇文章《Internet Explorer 编程简述（七）完美的“编码”菜单》

### 参考资料

+ MSDN: [Adding Entries to the Standard Context Menu](http://msdn.microsoft.com/library/default.aspurl=/workshop/browser/ext/tutorials/context.asp)
+ MSDN: [How To Adding to the Standard Context Menus of the WebBrowser Control](http://support.microsoft.com/default.aspx?scid=kb;en-us;177241)
+ MSDN: [IDocHostUIHandler::ShowContextMenu Method](http://msdn.microsoft.com/library/default.aspurl=/workshop/browser/hosting/reference/ifaces/idochostuihandler/showcontextmenu.asp)
+ BeginThread.com: [Custom WebBrowser Context Menus](http://www.beginthread.com/Article/Ehsan/Custom%20WebBrowser%20Context%20Menus/)