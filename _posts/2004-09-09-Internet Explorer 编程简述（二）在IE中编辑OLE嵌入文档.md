---
layout:     post
title:      "Internet Explorer 编程简述（二）在IE中编辑OLE嵌入文档"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2004-09-09
author:     "eagleboost"
header-img: "img/post-bg-sky-night.jpg"
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - OLE嵌入
    - In-Place Activating
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/99250)

除了打开Internet上的网页，Internet Explorer还能够浏览本地文件夹及文件。如果浏览的是PDF文档或Office文档，有时候你会发现当调用Navigate("xxx.doc")的时候，Adobe Reader/Acrobat或Office等Document Servers会在IE中嵌入自己的一个实例以打开相应的文件，当然有时候也会在独立的Acrobat或Office窗口中打开文件。 

在Adobe Reader/Acrobat的属性设置窗口中，我们可以找到“`Display PDF in browser`”的选项，如果勾上，则Navigate("xxx.pdf")将会以嵌入的方式在IE中浏览PDF文件，否则在独立的Adobe Reader/Acrobat窗口中浏览。但在Office的“选项”对话框中我们找不到这样的设置。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/Adobe_Reader_Preference.jpg)

**问题**：如何在自己的浏览器中控制Office这类Ole Servers的打开方式？

**答案**：修改文件夹选项，或修改注册表。

+ 方法1、如下所示，从控制面板中打开“文件夹”选项，在“文件类型”属性页上找到相应的文件后缀名，如“DOC”，点击“高级”按钮，在弹出的“编辑文件类型”对话框中有“在同一窗口中浏览”的选项，如果勾上，则以嵌入IE的方式打开文档，否则在独立窗口中打开。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/Open_File_In_Same_Window_1.jpg)
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/Open_File_In_Same_Window_2.jpg)

+ 方法2、直接修改注册表。 
在下面的键值下，保存了各种文件类型的注册信息，以Office文档为例，与文档相关键值如下。

```
HKEY_LOCAL_MACHINE/SOFTWARE/Classes
```
 
|文档类型|                             键值|
| --------                      | :-----  |
|Microsoft Excel 7.0 worksheet       | Excel.Sheet.5|
|Microsoft Excel 97 worksheet        | Excel.Sheet.8|
|Microsoft Excel 2000 worksheet      | Excel.Sheet.8|
|Microsoft Word 7.0 document         | Word.Document.6|
|Microsoft Word 97 document          | Word.Document.8|
|Microsoft Word 2000 document        | Word.Document.8|
|Microsoft Project 98 project        | MSProject.Project.8|
|Microsoft PowerPoint 2000 document  | PowerPoint.Show.8|
 
如果我们要修改Word文档的打开方式，，则在下面的键值下新建一个名为“`BrowserFlags`”，类型为“`REG_DWORD`”的子键值，如果设置其值为“8”，则在独立的窗口中打开Word文档，否则在嵌入IE的Word窗口中打开文档。

```
HKEY_LOCAL_MACHINE/SOFTWARE/Classes/Word.Document.8
```

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/cathyeagle/Office_File_BrowserFlags.jpg)

>注：Microsoft Excel 7.0 worksheet稍有不同，应设置BrowserFlags的值为“9”方可在独立的窗口中打开文档。


参考资料：
+ MSDN：259970：[In-Place Activating Document Servers in Internet Explorer](http://support.microsoft.com/default.aspx?scid=kb;en-us;259970)
+ MSDN：162059：[How to configure Internet Explorer to open Office documents in the appropriate Office program instead of in Internet Explorer](http://support.microsoft.com/default.aspx?scid=kb;en-us;162059)
