---
layout:     post
title:      "Internet Explorer 编程简述（五）调用IE隐藏的命令（中文版）"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-16
author:     "eagleboost"
header-img: "img/post-bg-cave.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - Add To Favorite
    - Import/Export Wizard
    - Shell DocObject View
    - Internet Explorer_Server
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106013)

### 1. 概述

概述除了“`整理收藏夹`”和“`添加到收藏夹`”对话框外，还有其它一些对话框是我们希望直接通过`WebBrowser`调用的，比如“导入/导出”对话框，用一般的方法很难调用。`IShellUIHelper`尽管提供了`ImportExportFavorites`方法，但结果只是显示一个选择文件的对话框，且只能导入/导出收藏夹而不能对Cookies操作。

### 2. 契机

MSDN中有一篇叫“[WebBrowser Customization](https://msdn.microsoft.com/en-us/ie/aa770041(v=vs.94))”的文章，其中介绍了通过`IDocHostUIHandler.ShowContextMenu`方法自定义`WebBrowser`上下文菜单的方法。其原理是从“`shdoclc.dll`”的资源中创建菜单，作一些修改之后用`TrackPopupMenu`函数（注意在标志中包含TPM_RETURNCMD）将菜单弹出，然后把返回的Command ID发送给“``Internet Explorer_Server``”窗口进行处理。

```c++
// 显示菜单
int iSelection = ::TrackPopupMenu(hMenu, 
  TPM_LEFTALIGN | TPM_RIGHTBUTTON | TPM_RETURNCMD,  
  ppt->x,  
  ppt->y,  
  0,  
  hwnd,  
  (RECT*)NULL);

// 发送Command ID到外壳窗口
LRESULT lr = ::SendMessage(hwnd, WM_COMMAND, iSelection, NULL);
```
好，如果找到所有上下文菜单的Command ID，不就可以随时调用了？确实是这样的。

### 3. 实现

用eXeScope之类应用程序资源探索器打开“`shdoclc.dll`”便可以在菜单资源下找到上下文菜单的设计，如下图：

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/eXeScope.jpg)

我们要做的，就是将这些ID发送到“`Internet Explorer_Server`”窗口进行处理。问题是`WebBrowser`其实是一个OLE容器，我们使用的CHtmlView又是更外层的封装，他们的m_hWnd成员变量并不是IE窗口的句柄，如何找到我们需要的句柄呢？请看下面的图：

根据图中显示的从属关系，顺藤摸瓜，最内层的窗口“`Internet Explorer_Server`”的句柄就是我们需要的东西。为了简化问题，我这里使用了来自MSDN Magazine资深专栏撰稿人Paul Dilascia的CFindWnd类，非常好用。

```c++
////////////////////////////////////////////////////////////////
// MSDN Magazine -- August 2003
// If this code works, it was written by Paul DiLascia.
// If not, I don't know who wrote it.
// Compiles with Visual Studio .NET on Windows XP. Tab size=3.
//
// ---
// This class encapsulates the process of finding a window with a given class name
// as a descendant of a given window. To use it, instantiate like so:
//
// CFindWnd fw(hwndParent,classname);
//
// fw.m_hWnd will be the HWND of the desired window, if found.
//
class CFindWnd 
{
  private:  
    //////////////////  
    // This private function is used with EnumChildWindows to find the child  
    // with a given class name. Returns FALSE if found (to stop enumerating).  
    //  
    static BOOL CALLBACK FindChildClassHwnd(HWND hwndParent, LPARAM lParam) 
    {    
      CFindWnd *pfw = (CFindWnd*)lParam;    
      HWND hwnd = FindWindowEx(hwndParent, NULL, pfw->m_classname, NULL);    
      if (hwnd) 
      {      
        pfw->m_hWnd = hwnd; // found: save it      
        return FALSE; // stop enumerating    
      }    
      EnumChildWindows(hwndParent, FindChildClassHwnd, lParam); // recurse    
      return TRUE; // keep looking
    }
  
  public:  
    LPCSTR m_classname; // class name to look for  
    HWND m_hWnd; // HWND if found  
    // ctor does the work--just instantiate and go  
    CFindWnd(HWND hwndParent, LPCSTR classname) : m_hWnd(NULL), m_classname(classname)  
    {
      FindChildClassHwnd(hwndParent, (LPARAM)this);  
    }
};
```

再写一个函数`InvokeIEServerCommand`，调用就很方便了，[《Internet Explorer 编程简述（四）“添加到收藏夹”对话框》](https://eagleboost.com/2004/09/12/Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E5%9B%9B-%E6%B7%BB%E5%8A%A0%E5%88%B0%E6%94%B6%E8%97%8F%E5%A4%B9-%E5%AF%B9%E8%AF%9D%E6%A1%86/)中最后给出的方法就是从这里来的。

```c++
void CMyHtmlView::InvokeIEServerCommand(int nID)
{  
  CFindWnd FindIEWnd(m_wndBrowser.m_hWnd, "`Internet Explorer_Server`");  
  ::SendMessage(FindIEWnd.m_hWnd, WM_COMMAND, MAKEWPARAM(LOWORD(nID), 0x0), 0);
}

void CMyHtmlView::OnFavAddtofav()
{  
  InvokeIEServerCommand(ID_IE_CONTEXTMENU_ADDFAV);//调用“添加到收藏夹”对话框
}
```

### 4. Command IDs

对所有的Command ID逐一尝试后我们发现：
1. 不是所有的Command ID都可以用上面的方法调用
2. 不是所有的Command ID都是由“`Internet Explorer_Server`”窗口处理
3. 有一些Command ID是由上一级窗口“`Shell DocObject View`”处理。

所以我们还需要写一个函数。

```c++
void CMyHtmlView::InvokeShellDocObjCommand(int nID)
{  
  CFindWnd FindIEWnd(m_wndBrowser.m_hWnd, "Shell DocObject View");  
  ::SendMessage(FindIEWnd.m_hWnd, WM_COMMAND, MAKEWPARAM(LOWORD(nID), 0x0), 0);
}
```

调用文章开头提到的“`导入/导出`”对话框可以这样来做：

```c++
void CDemoView::OnImportExport()
{
  InvokeShellDocObjCommand(ID_IE_FILE_IMPORTEXPORT);//调用“导入/导出”对话框
}
```

由"`Internet Explorer_Server`"窗口处理的Command ID:

```c++
#define ID_IE_CONTEXTMENU_ADDFAV 2261
#define ID_IE_CONTEXTMENU_VIEWSOURCE 2139
#define ID_IE_CONTEXTMENU_REFRESH 6042
```

由"`Shell DocObject View`"窗口处理的Command ID:

```c++
#define ID_IE_FILE_SAVEAS 258
#define ID_IE_FILE_PAGESETUP 259
#define ID_IE_FILE_PRINT 260
#define ID_IE_FILE_NEWWINDOW 275
#define ID_IE_FILE_PRINTPREVIEW 277
#define ID_IE_FILE_NEWMAIL 279
#define ID_IE_FILE_SENDDESKTOPSHORTCUT 284
#define ID_IE_HELP_ABOUTIE 336
#define ID_IE_HELP_HELPINDEX 337
#define ID_IE_HELP_WEBTUTORIAL 338
#define ID_IE_HELP_FREESTUFF 341
#define ID_IE_HELP_PRODUCTUPDATE 342
#define ID_IE_HELP_FAQ 343
#define ID_IE_HELP_ONLINESUPPORT 344
#define ID_IE_HELP_FEEDBACK 345
#define ID_IE_HELP_BESTPAGE 346
#define ID_IE_HELP_SEARCHWEB 347
#define ID_IE_HELP_MSHOME 348
#define ID_IE_HELP_VISITINTERNET 349
#define ID_IE_HELP_STARTPAGE 350
#define ID_IE_FILE_IMPORTEXPORT 374
#define ID_IE_FILE_ADDTRUST 376
#define ID_IE_FILE_ADDLOCAL 377
#define ID_IE_FILE_NEWPUBLISHINFO 387
#define ID_IE_FILE_NEWCORRESPONDENT 390
#define ID_IE_FILE_NEWCALL 395
#define ID_IE_HELP_NETSCAPEUSER 351
#define ID_IE_HELP_ENHANCEDSECURITY 375
```

### 5. Refresh

熟悉`TEmbeddedWB`的读者可能注意到了ID_IE_CONTEXTMENU_REFRESH(`6042`)这个ID，在TEmbeddedWB中给出了一个当网页刷新时触发的`OnRefresh`事件，其中的关键代码如下：

```pascal
if Assigned(FOnRefresh) and ((nCmdID = 6041 {F5}) or (nCmdID = 6042 {ContextMenu}) or (nCmdID = 2300)) then
begin  
  FCancel := False;  
  FOnRefresh(self, nCmdID, FCancel);  
  if FCancel then 
    Result := S_OK;
end;
```

其中的`6402`就是我们这里的`ID_IE_CONTEXTMENU_REFRESH`，2300是内置的刷新命令，那`6041`呢。见下图，还是“`shdoclc.dll`”，`6041`原来是IE“`查看`”菜单下“`刷新`”菜单的命令ID。实际开发中我们发现直接调用`WebBrowser`的Refresh命令有时候会导致一些错误，可以用这里的方法替换一下。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/eXeScope_View_Menu.jpg)

### 6. 需要注意的问题

1. 用InvokeIEServerCommand(`ID_IE_CONTEXTMENU_ADDFAV`)调用“添加到收藏夹”对话框时需要注意的是，IE接收到`ID_IE_CONTEXTMENU_ADDFAV`命令时是对网页中`当前被选中的链接`来执行“添加到收藏夹”操作的，如果没有选中的链接，才是将当前网页添加到收藏夹。
2. 新建IE窗口。这是浏览器编程中的难题之一，即从当前窗口新建一个`Internet Explorer`窗口，完全复制当前页的内容（包括“前进”、“后退”的状态），这可以通过InvokeShellDocObjCommand(`ID_IE_FILE_NEWWINDOW`)来实现。
3. 显示IE的版本信息。调用InvokeShellDocObjCommand(`ID_IE_HELP_ABOUTIE`)。
4. InvokeShellDocObjCommand(`ID_IE_FILE_PRINT`)调出的“打印”对话框是非模态的（我们不太清楚Microsoft的设计意图，我认为“打印”对话框应该是模态的），显示模态窗口的方法请参加我的另一篇文章[《利用`WH_CBT Hook`将非模态对话框显示为模态对话框》](https://eagleboost.com/2004/09/13/%E5%88%A9%E7%94%A8WH_CBT-Hook%E5%B0%86%E9%9D%9E%E6%A8%A1%E6%80%81%E5%AF%B9%E8%AF%9D%E6%A1%86%E6%98%BE%E7%A4%BA%E4%B8%BA%E6%A8%A1%E6%80%81%E5%AF%B9%E8%AF%9D%E6%A1%86/)

### 参考资料：
+ MSDN：[WebBrowser Customization](https://msdn.microsoft.com/en-us/ie/aa770041(v=vs.94))