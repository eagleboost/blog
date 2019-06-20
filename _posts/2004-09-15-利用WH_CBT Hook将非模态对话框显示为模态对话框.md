---
layout:     post
title:      "利用WH_CBT Hook将非模态对话框显示为模态对话框"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-13
author:     "eagleboost"
header-img: "img/post-bg-cage.jpg"
catalog: true
tags:
    - Windows 编程
    - 非模态
    - 模态
    - Hook
    - WH_CBT
    - CBTProc
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106004)

### 1. 意图

有时候我们希望将非模态窗口显示为模态窗口。比如在IE的“文件”菜单下选择“打印”，弹出的“打印”对话框就是非模态的（也许我们不太清楚Microsoft的设计意图，一般来说这里的“打印”对话框应该是模态的）。这种情况下如何将“打印”对话框显示为模态的呢？（这个对话框对我们来说是Black Box）

### 2. 简单实现

简单地说，模态窗口显示时，其父窗口是被Disable的，所以模态窗口才呈现“模态”，所以只要在显示我们非模态窗口前将父窗口Disable即可实现，如下：

```c++
AfxGetMainWnd()->EnableWindow(FALSE);//将主窗口Disable，显示出的非模态窗口就变成模态的了
ShowModelessWindow();
```
问题在于非模态窗口显示之后是立即返回的，那我们将父窗口Enable的代码放在哪里呢？笨办法是用时钟，不断地检测显示出来的非模态窗口是否已经关闭，若关闭则将父窗口Enable。

当然，还要更好的办法。

### 3. WH_CBT Hook

+ 首先定义两个变量，此处为全局静态变量。
 
```c++
static HHOOK g_hHook = NULL;
static HWND g_hWndDialog = NULL;//用以保存窗口句柄
```

+ 再添加一个函数CbtProc，由于是回调函数，注意要声明为static。

```c++
static LRESULT CALLBACK CbtProc(int nCode, WPARAM wParam, LPARAM lParam);
```

+ 挂钩

假设下面是我们的某个浏览器中调用“打印”对话框的函数

```c++
void CMyHtmlView::OnFilePrint()
{
  AfxGetMainWnd()->EnableWindow(FALSE);
  g_hWndDialog = 0; //可能多次调用，需要重置保存窗口句柄的变量
  g_hHook = SetWindowsHookEx(WH_CBT, CbtProc, NULL, GetCurrentThreadId());
  if (!g_hHook)
  {
    AfxGetMainWnd()->EnableWindow(TRUE);
    return;
  }
  ///此处调用“打印”对话框
}
 
LRESULT CALLBACK CMyHtmlView::CbtProc(int nCode, WPARAM wParam, LPARAM lParam) 
{  
  switch (nCode)
  {
    case HCBT_CREATEWND:
    {
      HWND hWnd = (HWND)wParam;
      LPCBT_CREATEWND pcbt = (LPCBT_CREATEWND)lParam;
      LPCREATESTRUCT pcs = pcbt->lpcs;
      if ((DWORD)pcs->lpszClass == 0x00008002)//#32770，“打印”对话框类名
      {
        if ( g_hWndDialog == 0 )
        {
          g_hWndDialog = hWnd; // 只保存一次保存“打印”窗口的句柄
        }
      }
      break;
    }
    
    case HCBT_DESTROYWND:
    {
      HWND hwnd = (HWND)wParam;
      if (hwnd == g_hWndDialog)
      {
        AfxGetMainWnd()->EnableWindow(TRUE);//恢复窗口状态
        UnhookWindowsHookEx(g_hHook);//去除挂钩
      }
      break;
    }
  }
  return CallNextHookEx(g_hHook, nCode, wParam, lParam); 
} 
```

很简单吧，更重要的是这种方法确实有效。
 
### 参考资料：
+ MSDN：[CBTProc](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms644977(v%3Dvs.85))