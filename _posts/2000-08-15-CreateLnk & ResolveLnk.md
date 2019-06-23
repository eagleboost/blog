---
layout:   post
title:  "CreateLnk & ResolveLnk"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2000-08-15
author:   "eagleboost"
header-img: "img/post-bg-sunset-ave.jpg"
tags:
  - CreateLnk
  - ResolveLnk
  - 快捷方式
  - Delphi
  - Windows编程
  - UX
  - 用户体验
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106246)，原文于2000年发布[阿甘的家](http://eagleboost.myrice.com/)。

本文介绍的两个函数，其原型声明如下，具体实现可以在源代码中找到，我只简要介绍一下用法。 

```pascal
function ResolveLnk(Wnd : HWnd; lpszLinkFile : PChar; lpszPath, lpszArgs, lpszWorkDir, lpszIconPath, lpszDescription : PChar; piIcon, piShowCmd, piSpecial : PInteger) : HResult;
function CreateLnk(Wnd : HWnd; const SrcFile, DstPath, WorkDir, Description : String) : HResult;
```
`ResolveLnk`接收lpszLinkFile作为快捷方式的源文件名，解析后的目标文件名保存在lpszPath中，其余参数只需形式上定义一个变量传入即可。

```pascal
  Var
    lpszLinkFile, lpszPath, lpszArgs, lpszWorkDir, lpszIconPath, lpszDescription: array[0..MAX_PATH] of Char;
    piIcon, piShowCmd, piSpecial: PInteger;
    
    ResolveLnk(0, lpszLinkFile, lpszPath, lpszArgs, lpszWorkDir, lpszIconPath, lpszDescription, piIcon, piShowCmd, piSpecial);
```

`CreateLnk`比较简单，SrcFile, DstPath分别是源文件名和希望建立的快捷方式文件名，Description是对该快捷方式的描述，举例如下：

```pascal
CreateLnk(Handle,'c:/app.exe','c:/LinkToApp.lnk','c:/','ShortCut to app.exe');
```

WorkDir在Windows帮助中是这样说的：指定包含原始项目或一些相关文件的文件夹。有时，程序需要使用其他位置的文件。这时需要您指定这些文件所在的文件夹，以便程序能够找到它们。

建立快捷方式还可以使用下面的过程： 

```pascal
procedure CreateShortCut(sName: string; dPath: integer; dName: widestring);
var
  tmpObject : IUnknown;
  tmpSLink : IShellLink;
  tmpPFile : IPersistFile;
  PIDL : PItemIDList;
  StartupDirectory : array[0..MAX_PATH] of Char;
  StartupFilename : String;
  LinkFilename : WideString;
begin
  coInitialize(nil);
  StartupFilename := sName;
  tmpObject := CreateComObject(CLSID_ShellLink);//创建建立快捷方式的外壳扩展
  tmpSLink := tmpObject as IShellLink; //取得接口
  tmpPFile := tmpObject as IPersistFile;//用来储存*.lnk文件的接口 
  tmpSLink.SetPath(pChar(StartupFilename));//设定.exe所在路径
  tmpSLink.SetWorkingDirectory(pChar(ExtractFilePath(StartupFilename)));　//设定工作目录
  SHGetSpecialFolderLocation(0,dPath,PIDL);//获得dPath的Itemidlist
  SHGetPathFromIDList(PIDL,StartupDirectory); //获得dPath路径
  LinkFilename := StartupDirectory + dName; 
  tmpPFile.Save(pWChar(LinkFilename),FALSE);//保存*.lnk文件 
  CoUninitialize
end;
```

应用举例，把快捷方式建立到“SendTo”目录：

```pascal
CreateShortCut('c:/app.exe', CSIDL_SENDTO, 'c:/LinkToApp.lnk');
```