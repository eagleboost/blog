---
layout:     post
title:      "Internet Explorer 编程简述(十二)正确地设置和转移焦点"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2006-05-29 23:52:00
author:     "eagleboost"
header-img: "img/post-bg-cage.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - 焦点
    - Focus
    - 加速键
    - Accelerator
    - OLEIVERB_UIACTIVATE
    - IHTMLWindow2
    - IHTMLDocument4
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2006年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/762468)

### 1. 概述

对于`99%`有`UI`的`Windows`应用程序来说，键盘操作都是不可或缺而又容易被人们遗忘的一环。如果对`Windows`组件作一次逐个的测试，我们会发现`Microsoft`提供的任何一个`Windows`组件都通过键盘实现完全的控制（“计算器”比较特殊，它是一个按钮很多且每个按钮都不能获得焦点的程序，但在帮助文档中我们仍然可以找到为每个按钮设置的快捷键），这对于一个专业的`Windows`应用程序或软件来说非常重要。换句话说，就算没有鼠标用户也不应该束手无策，用户应该可以通过键盘操作完成其希望的功能。焦点的转移无疑是键盘操作的一个重要方面，在浏览器编程中尤其如此。

### 2. 焦点的基本概念

一般说来，在`Windows`中用户通过键盘转移焦点（`Focus`）有两个方法：第一，对于输入框附近有标签提示的情况，按住`Alt`+某个预设的字母（`Accelerator`，加速键）将焦点快速转移到输入框。如下图所示，按下“`Alt+D`”，焦点应转移到地址输入框；按下“`Alt+G`”，焦点应转移到搜索框（本文对此不做讨论）。第二，按住`Tab`键，焦点转移到由应用程序控制的下一个可获得焦点的窗口；按下``Shift+Tab``，焦点转移到上一个可获得焦点的窗口。如下图所示，如果地址输入框是当前获得焦点的窗口，则按下Tab时，焦点应转移到搜索框，再按下``Shift+Tab``，焦点应回到地址输入框。
 

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/ReBarFocus.jpg)

 
焦点的设置和转移对于用户体验（`Experience`）来说是细微体贴而又重要的设计，但不幸的是不少`Windows`应用程序都或多或少犯了一些错误：

1) 完全没有加速键。
   
这在国产信息系统中尤为常见。设计较差的信息系统常常会出现一个窗口拥有数十个输入框的情况，如果为每个编辑框都提供一个加速键的话，问题就出来了。字母键只有`26`个，就算把数字键也用上，也难免不能满足要求，所以很多信息系统干脆就不要加速键。

2) 摆设用的加速键。

一些应用软件甚至不懂得加速键的意义，只知道依样画葫地在输入框的旁边用标签说明加速键，但仅此而已，用户根本无法通过`Alt`+加速键转移焦点到输入框。

3) 错误地（或不能）转移焦点

对于基于对话框的应用程序来说，常犯的错误是用户按下`Tab`键时，焦点出乎用户意料地在输入框之间乱窜。而在上图这样的例子中，常犯的错误则是不能通过`Tab`转移焦点，或者按`Tab`能转移焦点但按``Shift+Tab``不能朝反方向转移焦点。

4) 对嵌入的`ActiveX`控件缺乏处理

对于嵌入的`ActiveX`控件，尤其是`WebBrowser`控件来说，焦点的处理就更为麻烦了（这本是基于`WebBrowser`的浏览器编程的难题之一）。常见的浏览器要么不处理常规窗口与`WebBrowser`控件之间的焦点传递（[Maxthon](https://www.maxthon.cn/)、`Gosurf`只支持在输入框之间传递焦点）；要么处理不完整，焦点一旦从某个输入框转移到`WebBrowser`控件就再也回不来（如[GreenBrowser](https://en.wikipedia.org/wiki/GreenBrowser)）；更有的根本就不处理任何焦点的传递（如[世界之窗浏览器](http://www.theworld.cn/)）。
 
按照本系列文章的惯例，本文讨论的目的将是提供一个完整(未必完美)的解决方案:

一，焦点在嵌入`ReBar`的各个输入框之间传递

二，焦点在普通`Windows`窗口(输入框)与`WebBrowser`控件之间传递。

### 3. 设定目标

下图说明了我们希望实现的正常的焦点转移行为：
+ 从工具条上的任何一个输入框出发，按`Tab`将焦点转移到下一个输入框，按`Shift+Tab`将焦点转移到上一个输入框

+ 如果焦点所在输入框是工具条上的最后一个输入框，按`Ta`b将焦点转移到`WebBrowser`控件当前的活动`Html Element`(上一次获得焦点的`Element`)

+ 如果焦点所在输入框是工具条上的第一个输入框，按`Shift+Tab`将焦点转移到`WebBrowser`控件当前活动`Html Element`

+ 对于上面两种情况，若`WebBrowser`控件没有当前活动的可获得焦点`Html Element`，则焦点应从输入框转移到`WebBrowser`控件的第一个或最后一个可获得焦点的`Html Element`

+ 如果焦点当前位于`WebBrowser`控件中，按`Tab`将焦点转移到下一个`Html Element`，按`Shift+Tab`将焦点转移到上一个`Html Element`

+ 如果焦点当前位于`WebBrowser`控件中，且当前的活动`Html Element`是最后一个可获得焦点的`Html Element`，按Tab将焦点转移到工具条的第一个输入框

+ 如果焦点当前位于`WebBrowser`控件中，且当前的活动`Html Element`是第一个可获得焦点的`Html Element`，按`Shift+Tab`将焦点转移到工具条的最后输入框

以下图为例，“`Google`大全”为`WebBrowser`当前获得焦点的`Html Element`，举例如下：

+ 例1：假设当前焦点位于地址输入框，按下`Tab`键不松开，焦点转移的顺序应是：“地址栏”，“搜索栏”，“`Google`大全”……“将`Google`设为首页”，“地址栏”，“搜索栏”，“个性化主页”，“搜索记录”……

+ 例2：假设当前焦点位于地址输入框，且`WebBrowser`控件没有活动的获得焦点的`Html Element`，按下`Tab`键不松开，焦点转移的顺序应是：“地址栏”，“搜索栏”，“个性化主页”，“搜索记录”……“将`Google`设为首页”，“地址栏”，……

+ 例3：假设当前焦点位于“搜索记录”，按下`Shift+Tab`键不松开，焦点转移的顺序应是：“搜索记录”，“个性化主页”，“搜索栏”，“地址栏”，“将`Google`设为首页”……“搜索记录”……

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/AllFocus.jpg)


### 4. 工具条输入框之间的焦点转移

为实现统一的处理，我们从`CDialogBar`派生一个`CDialogBar`Ex类，由该类处理`Tab/Shift Tab`按键，而输入框(如`EditBox`，`ComboBox`等)则放在`CDialogBar`Ex的派生类(如`CUrlAddressBar`、`CSearchBar`等)中，这样输入框就可以专注于其它的功能。示例代码如下：

```c++
BOOL CDialogBarEx::PreTranslateMessage(MSG* pMsg)
{
  if ( ( pMsg->message==WM_KEYDOWN ) )
  {
    if ( (pMsg->wParam == VK_TAB) )
    {
      //由MainFrame处理如何转移焦点，按下Shift表示焦点应转移到上一个窗口
      g_pMainFrame->SetFocusToNextControl( GetKeyState(VK_SHIFT) >= 0 );
      return TRUE;
    }
  }
  ......
  return CDialogBar::PreTranslateMessage(pMsg);
}
 
void CMainFrame::SetFocusToNextControl(bool bNext)
{
  //m_wnd`ReBar`是一个CReBarEx，可从C`ReBar`派生
  if ( !m_wndReBar.SetFocusToNextControl(bNext) )
  {
    //如果CReBarEx在其子窗口中找不到下(上)一个可以设置焦点的窗口，则把焦点转移到`WebBrowser`
    CChildFrame *pChildFrame = (CChildFrame *)MDIGetActive();
    if ( pChildFrame && pChildFrame->GetActiveView() )
    {
      pChildFrame->GetActiveView()->SetFocus();
    }
  }
}
 
bool CReBarEx::SetFocusToNextControl(bool bNext)
{
  return bNext ? FocusNextControl() : FocusPrevControl();
}
 
bool CReBarEx::FocusNextControl()
{
  REBARBANDINFO rbbi;
  rbbi.cbSize = sizeof( rbbi );
  rbbi.fMask = RBBIM_CHILD;
   
  //先找到当前获得焦点的Band
  UINT nBand;
  for ( nBand = 0; nBand < m_rbCtrl.GetBandCount(); nBand++ )
  {
    VERIFY( m_rbCtrl.GetBandInfo(nBand, &rbbi) );
    if ( ::IsChild(rbbi.hwndChild, ::GetFocus()) )
    {
      break;
    }
  }
   
  //如果运行到这里，必定能够找到当前获得焦点的Band
  ASSERT(nBand < m_rbCtrl.GetBandCount());
   
  for ( nBand = nBand + 1; nBand < m_rbCtrl.GetBandCount(); nBand++ )
  {
    VERIFY( m_rbCtrl.GetBandInfo(nBand, &rbbi) );
    ::SetFocus(rbbi.hwndChild);
    if ( ::IsChild(rbbi.hwndChild, ::GetFocus()) )
    {
      //成功找到并设置焦点到下一个窗口
      return true;
    }
  }
  //当前获得焦点的窗口已经是ReBarEx中最后一个可获得焦点的窗口
  return false;
}
 
bool CReBarEx::FocusPrevControl()
{
  //实现与FocusNextControl类似，此处略去
}
 
void CReBarEx::OnSetFocus(CWnd* pOldWnd)
{
  //如果此时Shift为按下的状态，表示焦点可能是从`WebBrowser`的第一个活动`Html Element`转过来，
  //则将焦点转移到最后一个输入框，否则转移到第一个输入框
  //SetFocusToLastControl与SetFocusToFirstControl的实现相当简单，略去
  return GetKeyState(VK_SHIFT) < 0 ? SetFocusToLastControl() : SetFocusToFirstControl();
}
```

### 5. 焦点从`WebBrowser`转移到工具条输入框

处理浏览器的按键也曾是嵌入`WebBrowser`控件的编程难题之一，`Delphi`对`WebBrowser`的封装对按键的支持就存在很大问题。在《[Programming Internet Explorer](https://www.amazon.com/Programming-`Microsoft`-Internet-Explorer/dp/0735607818)》(原链接已失效)》中曾提到的方法是处理`MainFrame`的`PreTranslateMessage`，并在其中从`WebBrowser`的`Document`查询得到`IOleInPlaceActiveObject`接口，将按键交给`IOleInPlaceActiveObject`的`TranslateAccelerator`成员区处理。查询`MSDN`我们可以知道，`IOleInPlaceActiveObject`::`TranslateAccelerator`被调用时，`MSHTML`引擎会调用`IDocHostUIHandler`接口的`TranslateAccelerator`方法，从而给开发人员一个接口来处理按键。所以对于实现了`IDocHostUIHandler`接口的应用程序来说，按键处理就非常简单了。

```c++
//在此处理将焦点从WebBrowser中转移到ReBar上的输入框
HRESULT CMyView::OnTranslateAccelerator(LPMSG lpMsg,const GUID* pguidCmdGroup, DWORD nCmdID)
{
  if (lpMsg && lpMsg->message == WM_KEYDOWN && lpMsg->wParam == VK_TAB)
  {
    LPDISPATCH lpDispatch = GetHtmlDocument();
    CComQIPtr<IHTMLDocument2> pHTMLDoc = lpDispatch;
    if ( pHTMLDoc )
    {
      CComQIPtr<IHTMLElement> pElement;
      if ( SUCCEEDED(pHTMLDoc->get_activeElement(&pElement)) && !pElement )
      {
        //没有任何活动的Html Element，把焦点转移到ReBar
        g_pMainFrame->m_wndReBar.SetFocus();
        //通知`MSHTML`不要再继续处理按键
        return S_OK;
      }
    }
  }
  return S_FALSE;
}
```

### 6. 使WebBrowser获得焦点

使浏览器获得焦点也颇为讲究。我的一篇老文章《[TWebBrowser编程简述中](https://eagleboost.com/2001/02/07/TWebBrowser%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0/)》写到有好几种方法可以使`WebBrowser`获得焦点：`IOleObject::DoVerb(OLEIVERB_UIACTIVATE...)`，`IHTMLWindow2::focus()`，`IHTMLDocument4::focus()`。而实际上这几种方法是有区别的(内部实现我们并不清楚，也不关心)。

+ `IOleObject::DoVerb`能够将焦点设置到`WebBrowser`上一次失去焦点时获得焦点的`Html Element`上。缺点在于如果`WebBrowser`上次失去焦点时没有任何`Html Element`获得焦点，则DoVerb并不能保证焦点会转移到`WebBrowser`中。

+ `IHTMLWindow2::focus`不管三七二十一，将焦点转移到`WebBrowser`的开头`Html Element`。这显然不是我们想要的。

+ 测试的结果，`IHTMLDocument4::focus`似乎能够满足要求：能够记住`WebBrowser`上次失去焦点时获得焦点的`Html Element`；在`WebBrowser`上次失去焦点时没有任何`Html Element`获得焦点的情况下，能够焦点转移到开头的`Html Element`。但事实上并不理想，假如按住`Tab`键不松开，反复调用`IHTMLDocument4::focus`多次之后，我们会发现焦点再也到不到`WebBrowser`中了。
 
有没有完美解决的办法呢？答案当然是`Positive`的，如下：

```c++
void CMyView::OnSetFocus(CWnd* pOldWnd)
{
  LPDISPATCH lpDisp = GetHtmlDocument();
  CComQIPtr<IHTMLDocument2, &IID_IHTMLDocument2> pHTMLDoc(lpDisp);
  if ( pHTMLDoc )
  {
    CComQIPtr<IHTMLElement> pElement;
    if ( SUCCEEDED(pHTMLDoc->get_activeElement(&pElement)) && !pElement )
    {
      //没有任何活动元素，把焦点转移到WebBrowser的开头
      CComQIPtr<IHTMLWindow2> pHTMLWnd;
      if( SUCCEEDED(pHTMLDoc->get_parentWindow( &pHTMLWnd )) && pHTMLWnd )
      {
        pHTMLWnd->focus();
        return;
      }
    }
  }
   
  //有活动的元素(上一次的焦点)，直接将焦点转移过去
  //CWnd::SetFocus()会调用IOleObject::DoVerb()正确地设置焦点
  m_wndBrowser.SetFocus();
}
```

### 7. 总结

至此，我们就算完整地实现了焦点在普通窗口和浏览器之间的传递，任何时候，按住`Tab`键不松开，焦点将会在所有可获得焦点的窗口之间循环传递；同样，按住`Shift-Tab`不松开，焦点会以反方向传递。而不会出现用户无法将焦点转移到浏览器窗口的情况，或者焦点无法从浏览器窗口转移到输入框的情况。当然，还有比较重要也比较抽象的一点，增强了用户体验，呵呵。

### 8. 参考资料

《[Programming Internet Explorer](https://www.amazon.com/Programming-`Microsoft`-Internet-Explorer/dp/0735607818)》(原链接已失效)
