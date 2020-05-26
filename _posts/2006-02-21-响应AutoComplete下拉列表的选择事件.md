---
layout:   post
title:  "响应AutoComplete下拉列表的选择事件"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2006-02-21 23:23:00 
author:   "eagleboost"
header-img: "img/post-bg-gloden-bridge.jpg"
catalog: true
tags:
  - SHAutoComplete
  - Delphi
  - Windows编程
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2006年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/605054)

## 1、`SHAutoComplete`简介

`Shlwapi.dll`是微软提供的一个轻量级外壳工具函数库（`Shell Lightweight Utility Functions`），它提供了一些比较常用的函数，用以处理调色板、路径（如《[Secrets in Shlwapi.dll](https://eagleboost.com/2000/08/14/Secrets-in-ShlWapi.Dll/)》中提到的`PathCompactPath`函数）、注册表、字符串等。从`5.0`版本（随`Internet Explorer 5`推出）开始，`Shlwapi.dll`还提供了一个函数`SHAutoComplete`，它使得编辑框控件（如`Edit`、`ComboBox`）具有被称为“自动完成”的功能，即当用户在编辑框中输入的时候自动弹出一个非激活的窗口，为用户输入提供建议。如我们在`Internet Explorer`的地址栏输入“`google`”，如果系统记录了以前输入过“`www.google.com`”，则地址栏下方会显示出建议的网址。

这样贴心的功能自然为`IE 5`赢得市场提供了帮助，而对开发人员来说，如果能够毫不费力地为自己的程序添加这样的功能则是再好不过。`SHAutoComplete`就是最直接而简单的选择。下面的代码演示了如何调用`SHAutoComplete`函数为某个`Edit`控件添加自动完成的功能：

```c++
typedef HRESULT (CALLBACK* LPFNDLLFUNC)(HWND ,DWORD);

HINSTANCE hIns = LoadLibrary(_T("Shlwapi.dll"));
if( hIns != NULL )
{
  LPFNDLLFUNC lpfnDllFunc = (LPFNDLLFUNC)GetProcAddress(hIns, "SHAutoComplete");
  if( lpfnDllFunc != NULL )
  {
    lpfnDllFunc(m_wndAddressBar.m_hWnd, SHACF_AUTOAPPEND_FORCE_ON | SHACF_AUTOSUGGEST_FORCE_ON | SHACF_URLALL);
  }
  FreeLibrary(hIns);
}
```

从微软的习惯来说，这种对用户来说极为有用的功能不会只提供这样一个简单的函数就完事。事实上，`SHAutoComplete`只完成系统默认实现的功能，开发人员如果需要自定义以提供更强功能的话则可通过实现`IAutoComplete`接口以及`IAutoComplete`2接口来完成（在`Windows XP`中还提供了`IAutoCompleteDropDown`接口用以控制上图中那个下拉列表窗口的状态）。

## 2、问题的提出

我们知道，当用户在`ComboBox`控件的下拉框中用鼠标点击某个列表项或在列表项上按回车键时，`ComboBox`的父窗口会通过`WM_COMMAND`消息接收到一个`CBN_SELENDOK`通知，从而可以知道用户选择了那一个列表项并进行处理，在`VC++`中类似这样：

```c++
BEGIN_MESSAGE_MAP(CMainFrame, CMDIFrameWnd)

ON_CBN_SELENDOK(ID_ADDRESSBOX, &CMainFrame::OnSelAddress)

......

END_MESSAGE_MAP()

void CMainFrame::OnSelAddress()
{
  CString str = m_wndAddressBar.GetLBAddress();
  ...... //处理用户的选择
}
```

自然而然地，对于`SHAutoComplete`提供的下拉框，我们也希望有这样的通知，但`MSDN`中似乎并没有相关的文档。

### 3、SPY++

我们马上想到用`Spy++`来跟踪消息。以`IE`的地址栏为例，当我们在`SHAutoComplete`的下拉框中点击（按回车键有同样的效果）“`http://www.google.com`” 这个列表项时，地址栏中的`Edit`的消息踪迹如下所示：

```c++
<00589> 001B0BF4 S WM_SETTEXT lpsz:0013D074 ("http://www.google.com")
......
<00606> 001B0BF4 R EM_SETSEL
<00607> 001B0BF4 S message:0xC2B6 [Registered:AC_ItemActivate] wParam:00000000 lParam:0013D494
<00608> 001B0BF4 R message:0xC2B6 [Registered:AC_ItemActivate] lResult:00000000
<00609> 001B0BF4 S WM_KEYDOWN nVirtKey:VK_RETURN cRepeat:0 ScanCode:00 fExtended:0 fAltDown:0 fRepeat:0 fUp:0
```

包含`Registered:AC_ItemActivate`的这一行立刻引起了我们的注意。这是一个用`RegisterWindowMessage`函数在运行时向系统注册的消息（参见《[Windows通知栏图标高级编程概述](https://eagleboost.com/2004/08/09/1-Windows%E9%80%9A%E7%9F%A5%E6%A0%8F%E5%9B%BE%E6%A0%87%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B%E6%A6%82%E8%BF%B0/)》和《[具有自动恢复功能的通知栏图标控件](https://eagleboost.com/2004/08/09/2-%E5%85%B7%E6%9C%89%E8%87%AA%E5%8A%A8%E6%81%A2%E5%A4%8D%E5%8A%9F%E8%83%BD%E7%9A%84%E9%80%9A%E7%9F%A5%E6%A0%8F%E5%9B%BE%E6%A0%87%E6%8E%A7%E4%BB%B6/)》），从其字面意思来看正是我们想要的。我们注意到`Edit`在接收到该消息之前还接收到了`WM_SETTEXT`消息，即此刻`Edit`中的文字已经被`SHAutoComplete`的后台工作设置为我们所选择的列表项了，因此对于该消息的`wParam`和`lParam`的意义我们也可以不去深究。

## 4、问题解决

下面是一个简单的解决方案:

```c++
// CACEditSubclassWnd

class CACEditSubclassWnd : public CWnd
{
  DECLARE_DYNAMIC(CACEditSubclassWnd)
  static const UINT m_nAcItemActivateMsg;  //用来保存向系统注册的消息
public:
  CACEditSubclassWnd(){};
  virtual ~CACEditSubclassWnd(){};

protected:
  LRESULT OnAcItemActivate(WPARAM wParam, LPARAM lParam);
  DECLARE_MESSAGE_MAP()
};

// CACEditSubclassWnd

// 向系统注册我们需要的消息
const UINT CACEditSubclassWnd::m_nAcItemActivateMsg = ::RegisterWindowMessage(_T("AC_ItemActivate")); 

IMPLEMENT_DYNAMIC(CACEditSubclassWnd, CWnd)

BEGIN_MESSAGE_MAP(CACEditSubclassWnd, CWnd)
  ON_REGISTERED_MESSAGE(CACEditSubclassWnd::m_nAcItemActivateMsg, &CACEditSubclassWnd::OnAcItemActivate)
END_MESSAGE_MAP()

// CACEditSubclassWnd message handlers

LRESULT CACEditSubclassWnd::OnAcItemActivate(WPARAM wParam, LPARAM lParam)
{
  AfxGetMainWnd()->SendMessage(`WM_COMMAND`, MAKEWPARAM(LOWORD(AC`CBN_SELENDOK`), 0x0), 0);
  //AC`CBN_SELENDOK`是我们自定义的通知

  return 0L;
}
```

假设`CUrlAddressCombo`是一个地址栏类，则为其声明一个`CACEditSubclassWnd`类型的成员，并在适当的位置（如`Init`成员函数中）子类化`Edit`控件。

```c++
class CUrlAddressCombo : public CComboBoxEx
{
private:
  CACEditSubclassWnd m_ACEditSubclassWnd;
  ......
}

void CUrlAddressCombo::Init(void)
{
  m_ACEditSubclassWnd.SubclassWindow( GetEditCtrl()->m_hWnd );
}
```

而在`CMainFrame`中可以这样实现：

```c++
BEGIN_MESSAGE_MAP(CMainFrame, CMDIFrameWnd)
  ON_COMMAND(ACCBN_SELENDOK, &CMainFrame::OnACSelEndOk)
  ......
END_MESSAGE_MAP()

void CMainFrame::OnACSelEndOk()
{
  CString strAddr;
  m_wndAddressBar.GetEditCtrl()->GetWindowText(strAddr);
  ...... //处理用户的选择
}
```

非常简单，不是吗？

## 5、参考文献

MSDN: [SHAutoComplete Function](https://docs.microsoft.com/en-us/windows/win32/api/shlwapi/nf-shlwapi-shautocomplete)