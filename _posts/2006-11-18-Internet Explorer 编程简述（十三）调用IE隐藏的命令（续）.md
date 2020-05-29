---
layout:     post
title:      "Internet Explorer 编程简述（十三）调用IE隐藏的命令（续）"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2006-11-18 14:24:00
author:     "eagleboost"
header-img: "img/post-bg-gloden-bridge.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - CGID_ShellDocView
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2006年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/1394208)

### 1. 概述

在本系列五《[调用IE隐藏的命令](https://eagleboost.com/2004/09/16/Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E4%BA%94-%E8%B0%83%E7%94%A8IE%E9%9A%90%E8%97%8F%E7%9A%84%E5%91%BD%E4%BB%A4-%E4%B8%AD%E6%96%87%E7%89%88/)》中我们曾经从`MSDN`的一篇文章给出的`ShowContextMenu`范例入手，深入`shdoclc.dll`找到了藏于其中的浏览器上下文菜单资源，并以`SendMessage`发送`WM_COMMAND`消息到"`Internet Explorer_Server`"窗口以及其父窗口"`Shell DocObject View`"的方法完美实现了对“添加到收藏夹”对话框，“导入/导出向导”对话框等的调用，《[自定义浏览器上下文菜单](https://eagleboost.com/2004/09/19/0-Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E5%85%AD-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%B5%8F%E8%A7%88%E5%99%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E8%8F%9C%E5%8D%95/)》和《[完美的“编码”菜单](https://eagleboost.com/2004/09/19/1-Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E4%B8%83-%E5%AE%8C%E7%BE%8E%E7%9A%84-%E7%BC%96%E7%A0%81-%E8%8F%9C%E5%8D%95/)》也运用了同样的技术。

这次，我们还是从`ShowContextMenu`范例入手，再次挖掘`IE`隐藏的命令——`CGID_ShellDocView`的命令。

### 2. 原理

《[完美的“编码”菜单](https://eagleboost.com/2004/09/19/1-Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E4%B8%83-%E5%AE%8C%E7%BE%8E%E7%9A%84-%E7%BC%96%E7%A0%81-%E8%8F%9C%E5%8D%95/)》一文所用的技术，其关键在于从浏览器的文档接口查询得到`IOleCommandTarget`，进而调用`CGID_ShellDocView`命令组的命令`CmdID_GetMimeSubMenu(27)`实现将`IE`内置的编码菜单“拿来”使用。好了，既然`CGID_ShellDocView`是一个`Command Group`，那么`CmdID_GetMimeSubMenu`当然不会是惟一的一个，那为什么不尝试一下将其它可用的命令找出来呢。

### 3. 找命令

先为我们的任务定一个目标：找到那些传入简单`Variant`参数就能得到可观察的效果或输出参数的命令`ID`。像`CmdID_GetMimeSubMenu(27)`这类输出参数需要转换为一个`HMENU`才有意义的命令，如果没有`MSDN`的文档，我们只怕打破脑袋也想不出其中的含义，所以这类命令不在目标范围之内。
下面的函数简单地实现了对`CGID_ShellDocView`命令组命令的调用。

```c++
HRESULT ExecShellDocViewCommand(LPDISPATCH lpDocDisp, UINT nCmdID)
{
  HRESULT hr = S_FALSE;
  IOleCommandTarget *pct;
  if ( lpDocDisp && SUCCEEDED(lpDocDisp->QueryInterface(IID_IOleCommandTarget, (void **)&pct)))
  {
    CComVariant vtIn;
    vtIn.vt = VT_EMPTY;
    CComVariant vtOut;
    hr = pct->Exec(&CGID_ShellDocView, nCmdID, OLECMDEXECOPT_DONTPROMPTUSER, &vtIn, &vtOut);
    pct->Release();
  }
  return hr;
}
```

### 4. 命令列表

总算功夫没有完全白费，经过多轮测试（费时费力不一定讨好的工作）之后找到了如下命令:

1) 编码菜单和文字大小菜单

```c++
#define SHDVID_SHOWMIMECSETMENU    1
#define SHDVID_SHOWFONTSIZEMENU    50
```

想不到吧，传入命令`ID = 1`，编码菜单居然弹出来了！但有些问题，由于`vtIn.vt = VT_EMPTY`，所以菜单弹出点在屏幕左上角`(0, 0)`，不过我们倒发挥发挥“猜”的功夫，怎么传入一个`POINT`呢？先试试把一个`POINT`的指针强制转换一下传进去，不行；再试试`Win32`程序设计的风格，两个坐标一个放高位，一个放低位拼成一个长整数，再以`VT_I4`传入，这次成功了。所以我们写出下面的函数：

`nCmdID`当然就是上面两个值，分别表示显示编码菜单和文字大小菜单。

```c++
HRESULT ShowShellDocViewMenu(LPDISPATCH lpDocDisp, POINT pt, UINT nCmdID)
{
  HRESULT hr = S_FALSE;
  IOleCommandTarget *pct;
  if ( lpDocDisp && SUCCEEDED(lpDocDisp->QueryInterface(IID_IOleCommandTarget, (void **)&pct)))
  {
    try
    {
      CComVariant vtIn;
      vtIn.vt = VT_I4;
      vtIn.lVal = MAKELONG(pt.x, pt.y);
      CComVariant vtOut;
      hr = pct->Exec(&CGID_ShellDocView, nCmdID, OLECMDEXECOPT_DONTPROMPTUSER, &vtIn, &vtOut);
    }
    catch (...) {
    }
    pct->Release();
  }
  return hr;
}

HRESULT ShowMimeSetMenu(LPDISPATCH lpDocDisp, POINT pt)
{
  return ShowShellDocViewMenu(lpDocDisp, pt, SHDVID_SHOWMIMECSETMENU);
}

HRESULT ShowFontSizeMenu(LPDISPATCH lpDocDisp, POINT pt)
{
  return ShowShellDocViewMenu(lpDocDisp, pt, SHDVID_SHOWFONTSIZEMENU);
}
```

这样一来，菜单的`Enable/Disable`状态不需要我们来维护，命令也不需要我们来转发，系列七《[完美的“编码”菜单](https://eagleboost.com/2004/09/19/1-Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E4%B8%83-%E5%AE%8C%E7%BE%8E%E7%9A%84-%E7%BC%96%E7%A0%81-%E8%8F%9C%E5%8D%95/)》这篇文章也许可以删掉了，因为文中的代码可以简化为下面的样子，更改文字大小也只需要一行代码就能实现。

```c++
void CMainFrame::OnDropDown( NMHDR* pNotifyStruct, LRESULT* pResult )
{
  NMTOOLBAR* pNMToolBar = ( NMTOOLBAR* )pNotifyStruct;

  CMenu menu;
  CMenu* pPopup = 0;

  CRect rc;
  ::SendMessage(pNMToolBar->hdr.hwndFrom, TB_GETRECT, pNMToolBar->iItem, (LPARAM)&rc);
  rc.top = rc.bottom;
  ::ClientToScreen( pNMToolBar->hdr.hwndFrom, &rc.TopLeft() );
  Cpoint pt(rc.left, rc.top);

  LPDISPATCH lpDispatch = GetHtmlDocument();//获得文档指针

  switch ( pNMToolBar->iItem )
  {
    case ID_VIEW_ENCODE://“编码”按钮
      ShowMimeSetMenu(lpDispatch, pt);
      break;
    case ID_VIEW_FONTSIZE://“文字大小”按钮
      ShowFontSizeMenu(lpDispatch, pt);
      break;
  }

  *pResult = TBDDRET_DEFAULT;
}
```

2)  Location URL

```c++
#define SHDVID_GETLOCATIONURL    20
```

调用这个命令获得的返回参数是`BSTR`类型，其值是网页的`Location URL`

3) 文档当前的代码页codepage

```c++
#define SHDVID_GETCODEPAGE    23
```

调用这个命令可以得到当前网页的编码类型，比如`936`（简体中文），`1200`（`Unicode`）等。

| Code           | Value(Codepage) | Alphabet                         |
|----------------|-----------------|----------------------------------|
| DIN_66003      | 20106           | IA5(German)                      |
| NS_4551-1      | 20108           | IA5(Norwegian)                   |
| SEN_850200_B   | 20107           | IA5(Swedish)                     |
| _autodetect    | 50932           | Japanese(AutoSelect)             |
| _autodetect_kr | 50949           | Korean(AutoSelect)               |
| big5           | 950             | ChineseTraditional(Big5)         |
| csISO2022JP    | 50221           | Japanese(JIS-Allow1byteKana)     |
| euc-kr         | 51949           | Korean(EUC)                      |
| gb2312         | 936             | ChineseSimplified(GB2312)        |
| hz-gb-2312     | 52936           | ChineseSimplified(HZ)            |
| ibm852         | 852             | CentralEuropean(DOS)             |
| ibm866         | 866             | CyrillicAlphabet(DOS)            |
| irv            | 20105           | IA5(IRV)                         |
| iso-2022-jp    | 50220           | Japanese(JIS)                    |
| iso-2022-jp    | 50222           | Japanese(JIS-Allow1byteKana)     |
| iso-2022-kr    | 50225           | Korean(ISO)                      |
| iso-8859-1     | 1252            | WesternAlphabet                  |
| iso-8859-1     | 28591           | WesternAlphabet(ISO)             |
| iso-8859-2     | 28592           | CentralEuropeanAlphabet(ISO)     |
| iso-8859-3     | 28593           | Latin3Alphabet(ISO)              |
| iso-8859-4     | 28594           | BalticAlphabet(ISO)              |
| iso-8859-5     | 28595           | CyrillicAlphabet(ISO)            |
| iso-8859-6     | 28596           | ArabicAlphabet(ISO)              |
| iso-8859-7     | 28597           | GreekAlphabet(ISO)               |
| iso-8859-8     | 28598           | HebrewAlphabet(ISO)              |
| koi8-r         | 20866           | CyrillicAlphabet(KOI8-R)         |
| ks_c_5601      | 949             | Korean                           |
| shift-jis      | 932             | Japanese(Shift-JIS)              |
| unicode        | 1200            | UniversalAlphabet                |
| unicodeFEFF    | 1201            | UniversalAlphabet(Big-Endian)    |
| utf-7          | 65000           | UniversalAlphabet(UTF-7)         |
| utf-8          | 65001           | UniversalAlphabet(UTF-8)         |
| windows-1250   | 1250            | CentralEuropeanAlphabet(Windows) |
| windows-1251   | 1251            | CyrillicAlphabet(Windows)        |
| windows-1252   | 1252            | WesternAlphabet(Windows)         |
| windows-1253   | 1253            | GreekAlphabet(Windows)           |
| windows-1254   | 1254            | TurkishAlphabet                  |
| windows-1255   | 1255            | HebrewAlphabet(Windows)          |
| windows-1256   | 1256            | ArabicAlphabet(Windows)          |
| windows-1257   | 1257            | BalticAlphabet(Windows)          |
| windows-1258   | 1258            | VietnameseAlphabet(Windows)      |
| windows-874    | 874             | Thai(Windows)                    |
| x-euc          | 51932           | Japanese(EUC)                    |
| x-user-defined | 50000           | UserDefined                      |



4)  证书对话框

```c++
#define SHDVID_SSLSTATUS    33
```

在浏览https的网站时，浏览器的状态栏应显示“锁”图标，表示该页面使用了`xxx`位的`SSL`加密，双击该图标即可显示该站点的证书信息。这个命令`ID`就用来显示站点的证书对话框，当然对于没有加密的页面，调用的结果会显示“该类文档没有安全证书”。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/DlgZone.jpg)


5) 安全区域设置对话框

```c++
#define SHDVID_ZONESTATUS    35
```

与上面类似，当浏览器在不同的`Zone`之间切换时，状态栏应显示图标和文字以表示当前站点所处的`Security Zone`，双击改栏则显示安全区域属性页，用户可修改安全区域的设置。该命令`ID`可显示此对话框。


![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/DlgCertificate.jpg)

6) 管理加载项对话框

```c++
#define SHDVID_MANAGEADDONS    78
```

该命令ID显示“管理加载项”对话框


![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/DlgAddons.jpg)


7) “阻止下载文件”信息栏

```c++
#define SHDVID_INFOBAND_DOWNLOADFILE    81
```

在浏览器窗口上方显示“阻止下载文件”的信息栏，也许会有什么用……


![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/IEDLRestricted.jpg)

8) “低权限模式”信息栏

```c++
#define SHDVID_INFOBAND_PROTECTEDMODE    108
```

显示信息栏，表示Internet Explorer以低权限模式运行，也许会有什么用……

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/IEProtected.jpg)

9) `IE7.0`的“不完全”平滑缩放

```c++
#define SHDVID_ZOOM    113
```

调用该命令在`IE7.0`中使得网页按`125%`的缩放比例显示，其它的比例我没有试验成功。`IE7.0`的缩放比`IE6`有了很大改进，新的插值算法使得图片放大时平滑得多，至于如何调用`IE7.0`的缩放功能，此处按下不表，且待下回分解。

### 参考资料

MSDN: 《[WebBrowser Customization (Part 2)](http://search.msdn.microsoft.com/search/Redirect.aspx?title=WebBrowser+Customization+(Part+2)+&url=http://msdn.microsoft.com/workshop/browser/hosting/wbcustompart2.asp)》(原链接已失效)
