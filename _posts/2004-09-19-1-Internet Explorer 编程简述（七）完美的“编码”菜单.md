---
layout:     post
title:      "Internet Explorer 编程简述（七）完美的“编码”菜单"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-19 22:02:00
author:     "eagleboost"
header-img: "img/post-bg-lights-on-trees.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - Custom Context Menu
    - 编码菜单
    - Encoding Menu
    - SHDVID_GETMIMECSETMENU
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/109886)

### 1. 概述

Internet Explorer有实在太多没有公布的东西。上一篇文章《Internet Explorer 编程简述（六）自定义浏览器上下文菜单》提到的获取“编码”菜单的方法就是利用了浏览器的上层窗口“Shell DocObject View”的未公布的命令ID。本文将要介绍的是如何用这个ID把“编码”菜单放到我们自己的菜单中来（如工具条上的“编码”按钮的下拉菜单）。

```c++
#define SHDVID_GETMIMECSETMENU 27
......
CComPtr spCT;
hr = pcmdTarget->QueryInterface(IID_IOleCommandTarget, (void**)&spCT);
......
// Get the language submenu
hr = spCT->Exec(&CGID_ShellDocView, SHDVID_GETMIMECSETMENU, 0, NULL, &var);
```

### 2. 原理

上面指向IOleCommandTarget接口的智能指针spCT是从IDocHostUIHandler::ShowContextMenu的参数pcmdTarget得到的，它其实也可以从HTML文档接口得到，这就是实现的关键。

### 3. 实现

下面的代码演示了如何将“编码”菜单放置到我们自己的编码菜单上去。

```c++
void CMainFrame::OnDropDown( NMHDR* pNotifyStruct, LRESULT* pResult )
{
  const UINT CmdID_GetMimeSubMenu = 27;
  // Command ID for getting the Encoding submenu
 
  NMTOOLBAR* pNMToolBar = ( NMTOOLBAR* )pNotifyStruct;
  CMenu menu;
  CMenu* pPopup = 0;
  CMyHtmlView *pView = NULL;
  m_bIsEncodMenuPopup = false;//标志变量，用以在WM_INITMENUPOPUP消息处理函数中检查“编码”菜单
  switch ( pNMToolBar->iItem )
  {
  ......  
  case ID_VIEW_ENCODE://按下“编码”按钮
  {
    m_bIsEncodMenuPopup = true;
    VERIFY( menu.LoadMenu( IDR_ENCODE ) );//IDR_ENCODE是预置的“编码”菜单资源，内含任意一项占位用的菜单
    CMyHtmlView = GetActiveMyHtmlView();//检查当前是否存在活动的浏览器视图窗口
    if ( pView != NULL )
    {
      LPDISPATCH lpDispatch =pView->GetHtmlDocument();//获得文档指针
      if ( lpDispatch != NULL )
      {
        // Get an IDispatch pointer for the IOleCommandTarget interface.
        IOleCommandTarget * pCmdTarget = NULL;
        HRESULT hr = lpDispatch->QueryInterface(IID_IOleCommandTarget, (void**)&pCmdTarget);
        if ( SUCCEEDED( hr ) )
        {
          VARIANT varEncSubMenu;
          ::VariantInit( &varEncSubMenu );
          hr = pCmdTarget->Exec( &::CGID_ShellDocView, CmdID_GetMimeSubMenu, OLECMDEXECOPT_DODEFAULT, NULL, &varEncSubMenu );
          if ( SUCCEEDED( hr ) )
          {
            // 添加“编码”菜单
            MENUITEMINFO miiEncoding;
            ::memset( &miiEncoding, 0, sizeof(MENUITEMINFO) );
 
            miiEncoding.cbSize = sizeof(MENUITEMINFO);
            miiEncoding.fMask = MIIM_SUBMENU;
            miiEncoding.hSubMenu = reinterpret_cast< HMENU > (varEncSubMenu.byref);
            menu.SetMenuItemInfo(0, &miiEncoding, TRUE);//丢掉设计时占位用的菜单，替换为“编码”菜单
           }
        }
      }
    }
    pPopup = menu.GetSubMenu( 0 );
    break;
  }
  ......
  }
  
  if ( pPopup != 0 )
  {
    CRect rc;
    ::SendMessage( pNMToolBar->hdr.hwndFrom, TB_GETRECT, pNMToolBar->iItem, ( LPARAM )&rc );
    rc.top = rc.bottom;
    ::ClientToScreen( pNMToolBar->hdr.hwndFrom, &rc.TopLeft() );
    long lResult = pPopup->TrackPopupMenu( TPM_LEFTALIGN | TPM_LEFTBUTTON | TPM_RETURNCMD, rc.left, rc.top, this );
    m_bIsEncodMenuPopup = false;
    if ( pNMToolBar->iItem == ID_VIEW_ENCODE )
    {
      //其余的事教给浏览器去做，参考《Internet Explorer 编程简述（五）调用IE隐藏的命令（中文版）》
       CFindIEWnd FindIEWnd( pView->m_wndBrowser.m_hWnd, "Internet Explorer_Server");
      ::SendMessage( FindIEWnd.m_hWnd, WM_COMMAND, MAKEWPARAM(LOWORD(lResult), 0x0), 0 );
    }
    else
    {
      SendMessage( WM_COMMAND, MAKEWPARAM(LOWORD(lResult), 0x0), 0 );
    }
  }
  *pResult = TBDDRET_DEFAULT;
}

void CMainFrame::OnInitMenuPopup(CMenu* pPopupMenu, UINT nIndex, BOOL bSysMenu)
{
  CMDIFrameWndEx::OnInitMenuPopup(pPopupMenu, nIndex, bSysMenu);
  if ( m_bIsEncodMenuPopup )
  {
    //默认情况下“编码”的所有菜单项都是Disabled的，在此修改其状态为Enabled
    for ( UINT i=0; i

GetMenuItemCount(); i++ )
    {
      pPopupMenu->EnableMenuItem( pPopupMenu->GetMenuItemID( i ), MF_ENABLED | MF_BYCOMMAND );
    }
  }
}
```

这样一来，原本只在浏览器上下文菜单中出现的“编码”菜单就出现在了我们自己的工具条按钮下拉菜单上，无需更多的处理，菜单状态的改变，编码的设置等，一切都教给浏览器自己去完成了。

### 参考资料

+ [《Internet Explorer 编程简述（六）自定义浏览器上下文菜单》](https://eagleboost.com/2004/09/19/0_Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E5%85%AD-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%B5%8F%E8%A7%88%E5%99%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E8%8F%9C%E5%8D%95/)