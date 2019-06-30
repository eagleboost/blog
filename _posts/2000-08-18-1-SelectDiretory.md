---
layout:   post
title:  "SelectDiretory"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2000-08-18 00:00:01
author:   "eagleboost"
header-img: "img/post-bg-corner-21.jpg"
tags:
  - SelectDiretory
  - Delphi
  - Windows编程
  - UX
  - 用户体验
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106234)，原文于2000年发布[阿甘的家](http://eagleboost.myrice.com/)。

Delphi里有个函数`SelectDiretory`，重载了两种形式：

```pascal
function SelectDirectory(const Caption: string; const Root: WideString; out Directory: string): Boolean; overload; 
function SelectDirectory(var Directory: string; Options: TSelectDirOpts; HelpCtx: Longint): Boolean; overload; 
```

按第一种方式可以调用Win32的标准选择目录对话框，第二种方式弹出的则是Delphi自定义风格的对话框。我们编程常用的是第一种，但我在使用中发现，用该函数不能初始化对话框的起始目录，如右图：希望对话框弹出时就定位到某个目录，是办不到的。

我从来是单干，自然很久都没有找到答案，直到有一天终于注册上了“[大富翁](http://http://www.delphibbs.com)”（其实我很久以前就知道大富翁论坛了，只是一直注册不了），我提出的问题就是“如何指定`SelectDirectory`的起始目录”。问题很快得到了解答，答案是由cAkk提供的，如下： 

给那个窗口发消息可以设置路径：

```pascal
SendMessage(
  Hwnd,
  BFFM_SETSELECTION, 
  Ord(TRUE), 
  Longint(PChar(Path))
  ); 
```
关键是如何得到该窗口的句柄？

Borland在写`SelectDirectory`函数时省略了`BrowseInfo`的lpfn属性，这个属性指向一个CallBack函数,可以实现你的程序和该对话框窗口的通讯.该Callback函数声明为： 

```pascal
int BrowseCallbackProc(
  HWND hwnd,
  UINT uMsg,
  LPARAM lParam,
  LPARAM lpData
  );
```

其中,HWND参数就是传递过来的该对话框的句柄,得到这个句柄,你就可以 用我前面说的`SendMessage`设置路径了。 

还有一点,你应该在`BrowseCallbackProc`函数里判断当接受到`BFFM_INITIALIZED`消息时设置路径,也就是说：uMsg:=BFFM_INITIALIZED的时候。

具体实现如下，需要注意的几点是：

1. 不能再用`SelectDirectory`函数（要不就修改它的源代码），需要直接调用API函数`ShBrowseForFolder`。
2. 要把shlobj和AcriveX两个单元包含进去。 

```pascal
unit 
  Unit1; 
interface 
uses
  shlobj,ActiveX;

var
  Form1: TForm1; 
  Path: string; //起始路径

implementation 
{$R *.DFM} 

function BrowseCallbackProc(hwnd: HWND;uMsg: UINT;lParam: Cardinal;lpData: Cardinal): integer; stdcall; 
begin 
  if uMsg=BFFM_INITIALIZED then 
    result :=SendMessage(Hwnd,BFFM_SETSELECTION,Ord(TRUE),Longint(PChar(Path)))
  else
    result :=1 
end; 

function SelDir(const Caption: string; const Root: WideString; out Directory: string): Boolean; 
var
　WindowList: Pointer; 
　BrowseInfo: TBrowseInfo; 
  Buffer: PChar; 
  RootItemIDList, ItemIDList: PItemIDList; 
  ShellMalloc: IMalloc; 
  IDesktopFolder: IShellFolder; 
  Eaten, Flags: LongWord; 
begin 
  Result := False; 
  Directory := ''; 
  FillChar(BrowseInfo, SizeOf(BrowseInfo), 0); 
  if (ShGetMalloc(ShellMalloc) = S_OK) and (ShellMalloc <> nil) then 
  begin 
    Buffer := ShellMalloc.Alloc(MAX_PATH); 
    try  RootItemIDList := nil;  
      if Root <> '' then 
      begin
        SHGetDesktopFolder(IDesktopFolder);
        IDesktopFolder.ParseDisplayName(Application.Handle, nil, POleStr(Root), Eaten, RootItemIDList, Flags);  
      end;  

      with BrowseInfo do 
      begin
        hwndOwner := Application.Handle;
        pidlRoot := RootItemIDList;
        pszDisplayName := Buffer;
        lpszTitle := PChar(Caption);
        ulFlags := BIF_RETURNONLYFSDIRS;
        lpfn :=@BrowseCallbackProc;
        lParam :=BFFM_INITIALIZED;  
      end;  
    
      WindowList := DisableTaskWindows(0);  
      try
        ItemIDList := ShBrowseForFolder(BrowseInfo);  
      finally
        EnableTaskWindows(WindowList);  
      end;
　
      Result := ItemIDList <> nil;  
      if Result then 
      begin  
        ShGetPathFromIDList(ItemIDList, Buffer);
        ShellMalloc.Free(ItemIDList);
  Directory := Buffer;
      end; 
    finally
      ShellMalloc.Free(Buffer); 
    end; 
  end; 
end; 

procedure TForm1.SpeedButton1Click(Sender: TObject); 
var 
　Path1: string; begin 
  Path :=Edit1.Text; 
  SelDir('SelectDirectory Sample','', Path1); 
  Edit1.Text :=Path1 
  end; 
end. 
```