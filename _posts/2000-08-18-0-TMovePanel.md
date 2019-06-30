---
layout:     post
title:      "TMovePanel"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:       2000-08-18 00:00:00
author:     "eagleboost"
header-img: "img/post-bg-hongkong-street.jpg"
catalog: true
tags:
    - Delphi
    - Windows编程
    - UX
    - 用户体验
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2004年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/106228)，原文于2000年发布[阿甘的家](http://eagleboost.myrice.com/)。

### 两个老生常谈的问题：

1. 如何实现鼠标点住客户区拖动窗体？如何移动没有标题栏的窗体？
2. 如何在程序运行期间用鼠标拖动窗体上的控件？
  
在我这里，这两个问题是这样解决的——

### 拖动窗体

经典的做法："欺骗"系统,让它以为点中的是窗体的标题栏

```pascal
type
  TForm1 = class(TForm)
  private
    procedure WMNCHitTest(var M: TWMNCHitTest); message wm_NCHitTest; 
end; 

var
  Form1: TForm1; 
implementation 

procedure TForm1.WMNCHitTest(var M: TWMNCHitTest); 
begin
  inherited;      //call the inherited message handler
  if M.Result := htClient then //is the click in the client area?
    M.Result := htCaption;     //if so, make Windows think it's on the caption bar.
end;
```

这种做法看似巧妙，但实际上有缺陷，你会发现，窗体的客户区不可能向上移出屏幕。再来，把下面的代码做成一个控件，精彩的还在后面——

```pascal
unit MovePanel;
interface
uses
  Windows, Classes, Controls,ExtCtrls;
type
  MovePanel = class(TPanel)  //这个控件是继承Tpanel类的
  private
    PrePoint:TPoint;
    Down:Boolean;
    { Private declarations }
  protected
    { Protected declarations }
  public
    constructor Create(AOwner:TComponent);override;   
    //重载鼠标事件，抢先处理消息
    procedure MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);override;
    procedure MouseUp(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);override;
    procedure MouseMove(Shift: TShiftState;X, Y: Integer);override;
    { Public declarations }
  published
    { Published declarations }
end;

procedure Register;

implementation

constructor TMovePanel.Create(AOwner:TComponent);
begin
  inherited Create(AOwner); //继承父类的Create方法
end;

procedure TMovePanel.MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  if (Button=MBLeft) then 
  begin
    Down:=true;
    GetCursorPos(PrePoint);
  end; 
  //如果方法已存在，就触发相应事件去调用它，若不加此语句会造成访存异常
  if assigned(OnMouseDown) then
    OnMouseDown(self,Button,shift,x,y);
end;

procedure TMovePanel.MouseUp(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  if (Button=MBLeft) and Down then
    Down:=False;
  if assigned(OnMouseUp) then
    OnMouseUp(Self,Button,shift,X,y);
end;

procedure TMovePanel.MouseMove(Shift: TShiftState; X, Y: Integer);
Var
  NowPoint:TPoint;
begin
  if down then 
  begin
    GetCursorPos(nowPoint);
    //self.Parent在Form中就是MovePanel所在的窗体，或是MovePanel所在的容器像Panel
    self.Parent.Left:=self.Parent.left+NowPoint.x-PrePoint.x;
    self.parent.Top:=self.Parent.Top+NowPoint.y-PrePoint.y;
    PrePoint:=NowPoint;
  end;
  if Assigned(OnMouseMove) then
    OnMouseMove(self,Shift,X,y);
end;

procedure Register;
begin
  RegisterComponents('Md3', [TMovePanel]);
end;

end.
```

接下来，看看怎么用它吧。
+ 用法一：拖一个Form下来，加上我们的MovePanel，Align属性设为alClient，运行一下，移动窗体的效果还不错吧！想取消此功能，把MovePanel的Enabled属性设为False即可，简单吧！ 
  
+ 用法二：拖一个Form下来，加上普通的Panel，调整好大小，再在Panel上加上MovePanel, Align属性设为alClient，运行一下，这一次在我们拖动MovePanel时不是窗体在移动，而是Panel和MovePanel一起在窗体上移动，如果我们再把其他的控件放在MovePanel上，就成了可以在窗体上任意移动的控件了，就这么简单！ 

（原作者：福州大学 王骏）

 达到要求了吗？好像是的。再苛刻点儿，要求包括窗体在内的每一个控件都可以独立地用鼠标点住拖动，又该怎么办？

### 移动控件！

在一个新的Form中放入一个Panel，加入如下代码：

```pascal
procedure TForm1.Panel1MouseDown(Sender: TObject; Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
const 
  SC_DragMove = $F012; // a magic number 
begin 
  ReleaseCapture; 
  panel1.perform(WM_SysCommand, SC_DragMove, 0);
end; 
```

试试就知道了，你想怎么拖就怎么拖！这个方法很不错！在拖动单个控件时非常有效。

### 总结

一般情况下用MovePanel就够了，如果还要拖动单个控件，就再用上面最后一种方法，只要控件可以响应MouseDown事件就可以用！ 
