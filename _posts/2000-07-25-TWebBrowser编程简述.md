---
layout:     post
title:      "TWebBrowser编程简述"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:       2001-02-07 
author:     "eagleboost"
header-img: "img/post-bg-desktop.jpg"
catalog: true
tags:
    - TWebBrowser
    - Internet Explorer编程
    - 阿甘的家
    - Delphi
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/93984)，重新排版但对原文未作修改。原始文章发布于2007年在myrice.com托管的网站[阿甘的家](http://eagleboost.myrice.com/)，现已无法访问

## 引言

这篇文章最先发表于2000年07月25日，最后一次修改是在2001年02月07日。这里再次贴出的目的，一是作为前一阶段的完结，所以文章中的错误都不作修改；二是希望作为一个新的起点。我准备整理一下至今所积累的浏览器编程的知识，比较完整地写出来，与网友共勉。

## TWebBrowser编程简述

**摘要**：`Delphi` 3开始有了`TWebBrowser`构件，不过那时是以`ActiveX`控件的形式出现的，而且需要自己引入，在其后的4.0和5.0中，它就在封装好`shdocvw.dll`之后作为Internet构件组之一出现在构件面板上了。常常听到有人骂`Delphi`的帮助做得极差，这次的`TWebBrowser`又是Microsoft的东东，自然不会好到哪里去，虽说MSDN上什么都有，可是内容太过庞杂，如果没有入口点更是件烦人的事，查找起来给人的感觉大概可以用一句话来形容：非常复杂、复杂非常。

这里有平时我自己用`TWebBrowser`做程序的一些心得和上网收集到的部分例子和资料，整理了一下，希望能给有兴趣用`TWebBrowser`编程的朋友带来些帮助。


### 1. 初始化和终止化（Initialization & Finalization）

大家在执行`TWebBrowser`的某个方法以进行期望的操作，如`ExecWB`等的时候可能都碰到过“试图激活未注册的丢失目标”或“OLE对象未注册”等错误，或者并没有出错但是得不到希望的结果，比如不能将选中的网页内容复制到剪贴板等。以前用它编程的时候，我发现`ExecWB`有时侯起作用但有时侯又不行，在`Delphi`生成的缺省工程主窗口上加入`TWebBrowser`，运行时并不会出现“OLE对象未注册”的错误。同样是一个偶然的机会，我才知道OLE对象需要初始化和终止化（懂得的东东实在太少了）。

我用我的前一篇文章《`Delphi`程序窗口动画&正常排列平铺的解决》所说的方法编程，运行时出了上面所说的错误，我便猜想应该有OleInitialize之类的语句，于是，找到并加上了下面几句话，终于搞定！究其原因，我想大概是由于`TWebBrowser`是一个嵌入的OLE对象而不算是用`Delphi`编写的VCL吧。

```pascal
initialization
  OleInitialize(nil);
finalization
  try
    OleUninitialize;
  except
  end;
```
这几句话放在主窗口所有语句之后，“end.”之前。

### 2. EmptyParam

在`Delphi` 5中`TWebBrowser`的Navigate方法被多次重载：

```pascal
procedure Navigate(const URL: WideString); overload;
procedure Navigate(const URL: WideString; var Flags: OleVariant); overload; 
procedure Navigate(const URL: WideString; var Flags: OleVariant; var TargetFrameName: OleVariant); overload; 
procedure Navigate(const URL: WideString; var Flags: OleVariant; var TargetFrameName: OleVariant; var PostData: OleVariant); overload; 
procedure Navigate(const URL: WideString; var Flags: OleVariant; var TargetFrameName: OleVariant; var PostData: OleVariant; var Headers: OleVariant); overload;
```

而在实际应用中，使用后几种方法调用时，由于我们很少用到后面几个参数，但函数声明又要求是变量参数，一般的做法如下：

```pascal
var
  t:OleVariant;
begin
  webbrowser.Navigate(edit1.text,t,t,t,t);
end;
```
需要定义变量t（还有很多地方要用到它），很麻烦。其实我们可以用EmptyParam来代替（EmptyParam是一个公用的Variant空变量，不要对它赋值），只需一句话就可以了：

```pascal
webbrowser.Navigate(edit1.text,EmptyParam,EmptyParam,EmptyParam,EmptyParam);
```

虽然长一点，但比每次都定义变量方便得多。当然，也可以使用第一种方式。

```pascal
webbrowser.Navigate(edit1.text)
```

### 3. 命令操作

常用的命令操作用`ExecWB`方法即可完成，`ExecWB`同样多次被重载：

```pascal
procedure ExecWB(cmdID: OLECMDID; cmdexecopt: OLECMDEXECOPT); overload;
procedure ExecWB(cmdID: OLECMDID; cmdexecopt: OLECMDEXECOPT; var pvaIn: OleVariant); overload;
procedure ExecWB(cmdID: rOLECMDID; cmdexecopt: OLECMDEXECOPT; var pvaIn: OleVariant; var pvaOut: OleVariant); overload;
```
+ **打开**
   
   弹出“打开Internet地址”对话框，CommandID为OLECMDID_OPEN（若浏览器版本为IE5.0，则此命令不可用）。
   
+ **另存为**：
   
   调用“另存为”对话框。

```pascal
    ExecWB(OLECMDID_SAVEAS,OLECMDEXECOPT_DODEFAULT, EmptyParam, EmptyParam);
```
+ **打印、打印预览和页面设置**
   
   调用“打印”、“打印预览”和“页面设置”对话框（IE5.5及以上版本才支持打印预览，故实现应该检查此命令是否可用）。

```pascal
    ExecWB(OLECMDID_PRINT, OLECMDEXECOPT_DODEFAULT, EmptyParam, EmptyParam); 
    if QueryStatusWB(OLECMDID_PRINTPREVIEW)=3 then
      ExecWB(OLECMDID_PRINTPREVIEW, OLECMDEXECOPT_DODEFAULT, EmptyParam,EmptyParam);
    ExecWB(OLECMDID_PAGESETUP, OLECMDEXECOPT_DODEFAULT, EmptyParam, EmptyParam);
```

+ **剪切、复制、粘贴、全选**
   
   功能无须多说，需要注意的是：剪切和粘贴不仅对编辑框文字，而且对网页上的非编辑框文字同样有效，用得好的话，也许可以做出功能特殊的东东。获得其命令使能状态和执行命令的方法有两种（以复制为例，剪切、粘贴和全选分别将各自的关键字替换即可，分别为CUT,PASTE和SELECTALL）：
   

```pascal
    {用`TWebBrowser`的QueryStatusWB方法。}
    if(QueryStatusWB(OLECMDID_COPY)=OLECMDF_ENABLED) or OLECMDF_SUPPORTED then
      ExecWB(OLECMDID_COPY, OLECMDEXECOPT_DODEFAULT, EmptyParam, EmptyParam); 
```

```pascal
    {用IHTMLDocument2的QueryCommandEnabled方法。}
    var
      Doc: IHTMLDocument2;
    begin
      Doc := webbrowser.Document as IHTMLDocument2;
      if Doc.QueryCommandEnabled('Copy') then
        Doc.ExecCommand('Copy',false,EmptyParam);
    end; 
```

+ **查找**：参考[第九条“查找”功能](#FindText)。


### 4. 字体大小

类似“字体”菜单上的从“最大”到“最小”五项（对应整数0～4，Largest等假设为五个菜单项的名字，Tag 属性分别设为0～4）。

+ **读取当前页面字体大小**
  
```pascal
var
  t: OleVariant;
begin
  webbrowser.ExecWB(OLECMDID_ZOOM, OLECMDEXECOPT_DONTPROMPTUSER, EmptyParam,t);
  case t of 
    4: Largest.Checked := true;
    3: Larger.Checked := true;
    2: Middle.Checked := true;
    1: Small.Checked := true;
    0: Smallest.Checked := true;
  end;
end;
```   

+ **设置页面字体大小**

```pascal
  Largest.Checked := false;
  Larger.Checked := false;
  Middle.Checked := false;
  Small.Checked := false;
  Smallest.Checked := false;
  TMenuItem(Sender).Checked := true;
  t := TMenuItem(Sender).Tag;
  webbrowser.ExecWB(OLECMDID_ZOOM, OLECMDEXECOPT_DONTPROMPTUSER, t,t);
```


### 5. 添加到收藏夹和整理收藏夹

```pascal
const
  CLSID_ShellUIHelper: TGUID = '{64AB4BB7-111E-11D1-8F79-00C04FC2FBE1}';
var
  p:procedure(Handle: THandle; Path: PChar); stdcall;

procedure TForm1.`OrganizeFavorite`(Sender: Tobject);

var
  H: HWnd;
begin
  H := LoadLibrary(PChar('shdocvw.dll'));
  if H <> 0 then
  begin
    p := GetProcAddress(H, PChar('DoOrganizeFavDlg'));
    if Assigned(p) then 
      p(Application.Handle, PChar(FavFolder));
  end;

  FreeLibrary(h);
end;

procedure TForm1.AddFavorite(Sender: TObject);
var
  ShellUIHelper: ISHellUIHelper;
  url, title: Olevariant;
begin
  Title := webbrowser.LocationName;
  Url := webbrowser.LocationUrl;
  if Url <> '' then
  begin
    ShellUIHelper := CreateComObject(CLSID_SHELLUIHELPER) as IShellUIHelper;ShellUIHelper.AddFavorite(url, title);
  end;
end; 
```

用上面的通过ISHellUIHelper接口来打开“添加到收藏夹”对话框的方法比较简单，但是有个缺陷，就是打开的窗口不是模式窗口，而是独立于应用程序的。可以想象，如果使用与`OrganizeFavorite`过程同样的方法来打开对话框，由于可以指定父窗口的句柄，自然可以实现模式窗口（效果与在资源管理器和IE中打开“添加到收藏夹”对话框相同）。问题显然是这样的，上面两个过程的作者当时只知道`shdocvw.dll`中`DoOrganizeFavDlg`的原型而不知道`DoAddToFavDlg`的原型，所以只好用ISHellUIHelper接口来实现（或许是他不够严谨，认为是否是模式窗口无所谓？）。

下面的过程就告诉你`DoAddToFavDlg`的函数原型。需要注意的是，这样打开的对话框并不执行“添加到收藏夹”的操作，它只是告诉应用程序用户是否选择了“确定”，同时在`DoAddToFavDlg`的第二个参数中返回用户希望放置Internet快捷方式的路径，建立.Url文件的工作由应用程序自己来完成。

```pascal
procedure TForm1.AddFavorite(IE: TEmbeddedWB);
  procedure CreateUrl(AUrlPath, AUrl: PChar);
  var
    URLfile: TIniFile;
  begin
    URLfile := TIniFile.Create(String(AUrlPath));
    RLfile.WriteString('InternetShortcut', 'URL', String(AUrl));
    RLfile.Free;
  end; 

var
  AddFav: function(Handle: THandle;
  UrlPath: PChar; UrlPathSize: Cardinal;
  Title: PChar; TitleSize: Cardinal;
  FavIDLIST: pItemIDList): Bool; stdcall;
  FDoc: IHTMLDocument2;
  UrlPath, url, title: array[0..MAX_PATH] of char;
  H: HWnd;
  pidl: pItemIDList;
  FRetOK: Bool;
begin
  FDoc := IHTMLDocument2(IE.Document);
  if FDoc = nil then 
    exit;
  StrPCopy(Title, FDoc.Get_title);
  StrPCopy(url, FDoc.Get_url);
  if Url <> '' then
  begin
    H := LoadLibrary(PChar('shdocvw.dll'));
    if H <> 0 then
    begin
      SHGetSpecialFolderLocation(0, CSIDL_FAVORITES, pidl);
      AddFav := GetProcAddress(H, PChar('DoAddToFavDlg'));
      if Assigned(AddFav) then
        FRetOK := AddFav(Handle, UrlPath, Sizeof(UrlPath), Title, Sizeof(Title), pidl)
    end;
    FreeLibrary(h);
    if FRetOK then
      CreateUrl(UrlPath, Url);
  end
end;
```

### 6. 使WebBrowser获得焦点

`TWebBrowser`非常特殊，它从TWinControl继承来的SetFocus方法并不能使得它所包含的文档获得焦点，从而不能立即使用Internet Explorer本身具有得快捷键，解决方法如下：

```pascal
procedure TForm1.SetFocusToDoc;
begin
  if webbrowser.Document <> nil then
    with webbrowser.Application as IOleobject do
      DoVerb(OLEIVERB_UIACTIVATE, nil, webbrowser, 0, Handle, GetClientRect);
end; 
```

除此之外，我还找到一种更简单的方法，这里一并列出：

```pascal
if webbrowser.Document <> nil then
  IHTMLWindow2(IHTMLDocument2(webbrowser.Document).ParentWindow).focus 
```

刚找到了更简单的方法，也许是最简单的：

```pascal
if webbrowser.Document <> nil then
  IHTMLWindow4(webbrowser.Document).focus 
```

还有，需要判断文档是否获得焦点这样来做：

```pascal
if IHTMLWindow4(webbrowser.Document).hasfocus then 
  ...
```


### 7. 点击“提交”按钮

如同程序里每个窗体上有一个“缺省”按钮一样，Web页面上的每个Form也有一个“缺省”按钮——即属性为“Submit”的按钮，当用户按下回车键时就相当于鼠标单击了“Submit”。但是`TWebBrowser`似乎并不响应回车键，并且，即使把包含`TWebBrowser`的窗体的KeyPreview设为True，在窗体的KeyPress事件里还是不能截获用户向`TWebBrowser`发出的按键。

我的解决办法是用ApplicatinEvents构件或者自己编写TApplication对象的OnMessage事件，在其中判断消息类型，对键盘消息做出响应。至于点击“提交”按钮，可以通过分析网页源代码的方法来实现，不过我找到了更为简单快捷的方法，有两种，第一种是我自己想出来的，另一种是别人写的代码，这里都提供给大家，以做参考。

+ 用SendKeys函数向WebBrowser发送回车键

    在`Delphi` 5光盘上的Info/Extras/SendKeys目录下有一个SndKey32.pas文件，其中包含了两个函数SendKeys和AppActivate，我们可以用SendKeys函数来向WebBrowser发送回车键，我现在用的就是这个方法，使用很简单，在WebBrowser获得焦点的情况下（不要求WebBrowser所包含的文档获得焦点），用一条语句即可（SendKeys函数的详细参数说明等，均包含在SndKey32.pas文件中）。

```pascal
Sendkeys('~',true);// press RETURN key
```

+ 在OnMessage事件中将接受到的键盘消息传递给WebBrowser。

```pascal
procedure TForm1.ApplicationEvents1Message(var Msg: TMsg; var Handled: Boolean); 
  {fixes the malfunction of some keys within webbrowser control}
const
  StdKeys = [VK_TAB, VK_RETURN]; { standard keys }
  ExtKeys = [VK_DELETE, VK_BACK, VK_LEFT, VK_RIGHT]; { extended keys }
  fExtended = $01000000; { extended key flag }
begin
  Handled := False;
  with Msg do
    if ((Message >= WM_KEYFIRST) and (Message <= WM_KEYLAST)) and ((wParam in StdKeys) or {$IFDEF VER120}(GetKeyState(VK_CONTROL) < 0) or {$ENDIF}(wParam in ExtKeys) and ((lParam and fExtended) = fExtended)) then
    try
      if IsChild(Handle, hWnd) then 
        { handles all browser related messages }
      begin
        with {$IFDEF VER120}Application_{$ELSE}Application{$ENDIF} asIOleInPlaceActiveObject do
          Handled := TranslateAccelerator(Msg) = S_OK;
          if not Handled then
          begin
            Handled := True;
            TranslateMessage(Msg);
            DispatchMessage(Msg);
          end;
      end;
    except
    end;
end; // MessageHandler
```
（此方法来自EmbeddedWB.pas）

### 8. 直接从`TWebBrowser`得到网页源码及Html

下面先介绍一种极其简单的得到`TWebBrowser`正在访问的网页源码的方法。一般方法是利用`TWebBrowser`控件中的Document对象提供的IPersistStreamInit接口来实现，具体就是：先检查WebBrowser.Document对象是否有效，无效则退出；然后取得IPersistStreamInit接口，接着取得HTML源码的大小，分配全局堆内存块，建立流，再将HTML文本写到流中。程序虽然不算复杂，但是有更简单的方法，所以实现代码不再给出。其实基本上所有IE的功能`TWebBrowser`都应该有较为简单的方法来实现，获取网页源码也是一样。下面的代码将网页源码显示在Memo中。

```pascal
Memo.Lines.Add(IHtmlDocument2(webbrowser.Document).Body.OuterHtml); 
```
同时，在用`TWebBrowser`浏览HTML文件的时候要将其保存为文本文件就很简单了，不需要任何的语法解析工具，因为`TWebBrowser`也完成了，如下：

```pascal
Memo.Lines.Add(IHtmlDocument2(webbrowser.Document).Body.OuterText); 
```

### <a name="FindText"></a>9. “查找”功能

查找对话框可以在文档获得焦点的时候通过按键Ctrl-F来调出，程序中则调用`IOleCommandTarget`对象的成员函数Exec执行OLECMDID_FIND操作来调用，下面给出的方法是如何在程序中用代码来做出文字选择，即你可以自己设计查找对话框。

```pascal
var
  Doc: IHtmlDocument2;
  TxtRange: IHtmlTxtRange;
begin
  Doc := webbrowser.Document as IHtmlDocument2;
  Doc.SelectAll;{此处为简写，选择全部文档的方法请参见第三条命令操作。这句话尤为重要，因为IHtmlTxtRange对象的方法能够操作的前提是Document已经有一个文字选择区域。由于接着执行下面的语句，所以不会看到文档全选的过程。}
  TxtRange := Doc.Selection.CreateRange as IHtmlTxtRange;
  TxtRange.FindText('Text to be searched',0.0);
  TxtRange.Select;
end;
```

还有，从Txt.Get_text可以得到当前选中的文字内容，某些时候是有用的。

### 10. 提取网页中所有链接

这个方法来自大富翁论坛hopfield朋友的对一个问题的回答，我本想自己试验，但总是没成功。

```pascal
var
  doc:IHTMLDocument2;
  all:IHTMLElementCollection;
  len,i:integer;
  item:OleVariant;
begin
  doc:= webbrowser .Document as IHTMLDocument2;
  all:= doc.Get_links;//doc.Links亦可
  len:= all.length;
  for i:= 0 to len-1 do 
  begin
    item:= all.item(i,varempty);//EmpryParam亦可
    Memo.lines.add(item.href);
  end;
end;
```
### 11. 设置`TWebBrowser`的编码

为什么我总是错过很多机会？其实早就该想到的，但是一念之差，便即天壤之别。当时我要是肯再多考虑一下，多试验一下，这就不会排到第11条了。下面给出一个函数，搞定，难以想象的简单。

```pascal
procedure SetCharSet(AWebBrowser: `TWebBrowser`; ACharSet: String);
var
  RefreshLevel: OleVariant;
begin
  IHTMLDocument2(AWebBrowser.Document).Set_CharSet(ACharSet);
  RefreshLevel := 7;//这个7应该从注册表来，帮助有Bug。
  AWebBrowser.Refresh2(RefreshLevel);
end; 
```

### 12. 在`TWebBrowser`中输入字符时激活菜单的解决

许多朋友编程的时候都遇到了这样一个问题，在`TWebBrowser`中输入时，键入的字符如果与菜单（用ToolBar做的菜单）的加速键相同就会激活菜单。有朋友解决办法是把加速键前面的“&”符号去掉，使得字符失去“加速”功能，这种方法未尝不可，只不过显得不够“专业”。其实略加分析我们就可以想到，是ToolBar抢先处理了按键（因为ToolBar本身就设计为用来实现具有Windows新风格的菜单），所以只需要修改ToolBar的源代码中处理菜单按键的那部分代码即可，方法如下：

1. 在$(`Delphi`)/source/vcl目录下找到comctrls.pas，拷贝到自己的程序所在目录，然后打开它。
2. 找到TToolBar.CMDialogChar过程，把过程体注释掉（如果你愿意的话，可以修改它）。
2. 重新编译自己的程序。

怎么样，是不是很简单？但它确实有效。

### 13. 去掉`TWebBrowser`的滚动条

缺省地，`TWebBrowser`是滚动条的，虽然我们可以在网页中设置不需要滚动条，不过，有些时候可能会有特殊的要求，比如，网页是有滚动条的，但又想去掉它该怎么办呢？很简单，下面给出两行代码，都可以达到目的，可谓殊途同归。

```pascal
IHTMLBodyElementDisp(IHTMLDocument2(webbrowser.document).body).scroll:= 'no';
```

```pascal
webbrowser.oleobject.Document.body.Scroll := 'no'
```

注：第一种方法需要在uses部分加上MSHTML_TLB或者MSHTML。

### 14. 通过IUniformResourceLocator接口建立Internet快捷方式

前面说到的显示“添加到收藏夹”模式对话框的方法中举了一个建立Internet快捷方式的例子，就其本身来说不太规范，属于取巧一类的方法。下面介绍的方法是通过接口来实现的。

```pascal
procedure CreateIntShotCut(aFileName, aURL: PChar);
var
  IURL: IUniformResourceLocator;
  PersistFile: IPersistfile;
begin
  if Succeeded(CoCreateInstance(CLSID_InternetShortcut,nil,CLSCTX_INPROC_SERVER, IID_IUniformResourceLocator,IURL)) then
  begin
    IUrl.SetURL(aURL, 0);
    Persistfile := IUrl as IPersistFile;
    PersistFile.Save(StringToOleStr(aFileName), False);
  end;
end;
```
其中IUniformResourceLocator接口的声明在IeConst.pas中，IeConst.pas可以在网站IE & `Delphi`找到； IPersistfile接口的声明在`ActiveX`.pas中。

注：这个函数的AURL参数必须包含协议前缀，如“Http://eagleboost.myrice.com”。
 
+ 最先发表日期：2000年07月25日
+ 最后修改日期：2001年02月07日