---
layout:   post
title:  "代码重温：TZoCInetChecker——一个检测网络连接的类"
subtitle:   "——谨以怀念写Delphi的青春岁月"
date:   2006-11-19 10:40:00 
author:   "eagleboost"
header-img: "img/post-bg-color-feather.jpg"
catalog: true
tags:
  - 网络小灵通
  - InternetOpen
  - InternetSetStatusCallback
  - InternetOpenUrl
  - INTERNET_STATUS_CALLBACK
  - Delphi
  - Windows编程
  - CathyEagle
  - 存档
  - csdn
---

> 本文转载自[我2006年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/1395744)

## 1、由来

几年前读书的时候有很长一段时间学校的网络很烂，一来上网的人多网络就可能断掉，过一段时间又会恢复；二来一幢楼只有一个网段，学校显然是低估了学生们对网络对需求和对计算机购买能力，所以有些放学才开机的同学常为分不到IP而烦恼。


学校里大家最常做的事之一就是整天开着`FTP`下资源，所以网络断掉又恢复最叫人恼火了。因为`FTP`服务器通常都会限制连接，所以为了在网络恢复时抢先一步，同学们会设置`FTP`客户端自动重连，并把重连的时间间隔设置得尽可能小，这类同学通常是不怎么关机的，我就是其中之一`^_^b`；有的同学不怎么下资源，但是网络不知道什么时候恢复也很烦恼，所以干脆就一直打开`ping xxx.xxx.xxx.xxx -t`一直ping服务器，并且过一会儿就看看屏幕，“莫办法啊”；还有的苦于分不到IP，或者干脆就故意捣乱，要不就用ping不断向别人发大数据包，要不用“`网络执法官`”来发`ARP`欺骗把别人踢出去……总之是乱了套了。

 
对于这种情况，技术手段就应该发挥一下作用了。先是有同学写了个带GUI的ping，如果网络通畅就在通知栏显示绿灯的图标，否则就是红灯，但这种方法也有问题。大家都知道，教育网有其特殊之处，我们有时候把教育网内称为内网，而需要代理才能访问的外部网络称为外网（由于有外网的存在，proxy探测软件在学校里用得相当广）。ping的手段对于检测内网问题不大，速度快，把学校主页或者网关作为目标就行了，但对外网就有问题了。Internet上服务器为防止攻击，会采取的措施之一是把`ICMP`响应关掉，客户端不能ping通服务器。

 
由于既然带GUI的ping不能解决问题，我当时就决定写一个真正能解决问题的软件，后来女朋友帮我取名为“网络小灵通”——一个可爱而贴切的名字。

## 2、分析

既然ping不能用，当然要另寻出路。我于是想到了TCP ping。是啊，ping不通，但我还是可以上网，因为Internet上的服务器再怎么着也不会把``HTTP``的端口封掉吧，所以解决的方案其实非常的Straightforward，剩下就是如何实现了。以我的习惯，先上网去搜，有现成的代码就直接为我所用，实在不行再自己写。

 
结果居然找了很久没找到，看来老外的网络没这么烂，许多检测是否On-Line的代码也根本不能解决问题。像url.dll提供的`InetIsOffline`函数，简直一点用的都没有（也许是IE4.0时代用于拨号网络的），同样`RasEnumConnections`也不能用。既然没有直接的函数可用，看来是要写一写了，当然我不打算直接用纯`socket`的方式，我也不熟悉，`WinINet API`（`Windows® Internet API`）应该是最好的选择，名字不就叫做`Internet API`么？
 

下面是最先写出的一个函数：

```pascal
function CheckUrl(url:string):boolean; 
var 
  hSession, hfile, hRequest: hInternet; 
  dwindex,dwcodelen :dword; 
  dwcode:array[1..20] of char; 
  res : pchar; 
begin 
  //检查URL是否包含http://，如果不包含则加上
  if pos('http://',lowercase(url))=0 then 
    url := 'http://'+url; 

  Result := false; 

  hSession := InternetOpen('InetURL:/1.0', 
    INTERNET_OPEN_TYPE_PRECONFIG,nil, nil, 0); //建立会话句柄
  if assigned(hsession) then 
  begin 
    hfile := InternetOpenUrl(hsession, pchar(url), nil, 0,
      INTERNET_FLAG_RELOAD, 0);    //打开URL所指资源

    dwIndex := 0; 
    dwCodeLen := 10; 
    HttpQueryInfo(hfile, `HTTP`_QUERY_STATUS_CODE, 
      @dwcode, dwcodeLen, dwIndex); //获取返回的`HTTP`头信息

    res := pchar(@dwcode); 
    result:= (res ='200') or (res ='302'); 

    if assigned(hfile) then 
      InternetCloseHandle(hfile);   //关闭URL资源句柄
    InternetCloseHandle(hsession);   //关闭Internet会话句柄
  end; 
end;
```

毫无疑问，这个函数可以完成任务。它尝试Open一个`Internet Session`，然后检查返回的`HTTP`头信息。状态值200表示连接服务器成功，302表示Redirect之后连接成功，对这两种情况我们都可以认为能够成功地连接到Internet并打开网页了。但我认为这还不够好，一个函数的实现虽然简单，但因为是阻塞的调用，所以不够优美，如果能够知道Windows在连接的时候处于什么状态岂不是更好。试想如果大家用浏览器上网的时候总会感觉“阻塞”一下才能看到网页，那可是非常差的Experience。

### 3、TZoCInetChecker实现

所以非阻塞的异步探测是非常必要的，`WinInet`也提供了这样的调用方式，关键在于调用`InternetSetStatusCallback`函数通知`WinInet`在有状态反馈时通知我们。如果是用MFC的话`CInternetSession`类倒是可以用，不过Delphi的VCL没有这样直接的类，只好自己写了，我们的封装能力也决不弱于C++ ^_^

 
下面就是那时写的一个类`TZoCInetChecker`，提供了一个Method，几个Properties，还有几个Events，用起来和上面那个阻塞调用的函数一样方便，当然，它也包含了阻塞方式的检测：

```pascal
方法
    Execute：      调用我就好了，其它的交给我去办
属性
    Busy：         我正忙着呢，还没检测完，一会儿再来
    AccessType：   你希望我怎么连接网络，直接连还是用代理？
    AsynRequest：  同步检测的话我们要有一会儿看不到对方了，呜呜
    UserAgent：    想告诉Web服务器是阿猫还是阿狗在连接？在这里设置就好了
    Proxy：        让我用代理连网络，用哪个代理啊？
    Url：          原来是要检测这个地址能不能访问，保证办到
事件
    OnStart：      开始检测之前我会在这里通知你一声
    OnStatusChange：检测过程中发生什么事我会在这里及时通知你的
    OnComplete：    检测完成了，成功不成功我都会告诉你
{**********************************************************}
{                                                          }
{  TZoCInetChecker Component Version 1.00                  }
{                                                          }
{  Function: Asynchronously Check if a given web page      }
{            can be successfully retreived and feed back   }
{            the user with callback status infomation      }
{                                                          }
{                                                          }
{  This is a freeware. If you made cool changes on it,     }
{  please send them to me.                                 }
{                                                          }
{  Email: eagleboost@msn.com                               }
{  URL: http://www.ZoCsoft.com                             }
{                                                          }
{  History: + New Feature, - Removed, * Modified           }
{                                                          }
{           version 1.00 2005-06-03                        }
{             The first version                            }
{                                                          }
{**********************************************************}


unit ZoCInetChecker;


interface


uses
  Windows, Messages, SysUtils, Classes, `WinInet`;


const
  INTERNET_STATUS_DETECTING_PROXY       = 80;
{$EXTERNALSYM INTERNET_STATUS_DETECTING_PROXY}
  WM_STATUSCHANGE                       = WM_USER + 200;
  WM_CHECKCOMPLETE                      = WM_USER + 201;


type
  LPINTERNET_ASYNC_RESULT = ^INTERNET_ASYNC_RESULT;
  INTERNET_ASYNC_RESULT = record
    dwResult: DWORD;
    dwError: DWORD;
  end;


  pREQUEST_CONTEXT = ^REQUEST_CONTEXT;
  REQUEST_CONTEXT = record
    hWindow: HWND;                      
    hOpen: HINTERNET;                   //HINTERNET handle created by InternetOpen
  end;


  TOnStatusChangeEvent = procedure(Sender: TObject; StatusCode: Cardinal) of object;
  TOnCompleteEvent = procedure(Sender: TObject; Connected: Boolean) of object;
  TAccessType = (atDirectConnect, atPreConfig, atPreConfigWithNoProxy, atProxy);


  TZoCInetChecker = class(TComponent)
  private
    { Private declarations }
    FUrl: string;
    FAccessType: TAccessType;
    FProxy: string;
    FOnStart: TNotifyEvent;
    FOnStatusChange: TOnStatusChangeEvent;
    FOnComplete: TOnCompleteEvent;
    FBusy: Boolean;
    hOpenUrl, hOpen: HINTERNET;
    FUserAgent: string;
    FWindowHandle: HWnd;
    iscCallback: INTERNET_STATUS_CALLBACK;
    RC: REQUEST_CONTEXT;
    FAsynRequest: Boolean;
    procedure WndProc(var Msg: TMessage);
  protected
    { Protected declarations }
    procedure DoOnStatusChange(StatusCode: Cardinal); dynamic;
  public
    { Public declarations }
    property Busy: Boolean read FBusy write FBusy;
    function Execute: Boolean;
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
  published
    { Published declarations }
    property AccessType: TAccessType read FAccessType write FAccessType
      default atDirectConnect;
    property AsynRequest: Boolean read FAsynRequest write FAsynRequest
      default True;
    property UserAgent: string read FUserAgent write FUserAgent;
    property Proxy: string read FProxy write FProxy;
    property Url: string read FUrl write FUrl;
    property OnStart: TNotifyEvent read FOnStart write FOnStart;
    property OnStatusChange: TOnStatusChangeEvent read FOnStatusChange write
      FOnStatusChange;
    property OnComplete: TOnCompleteEvent read FOnComplete write FOnComplete;
  end;


procedure Register;
function StatusCode2StatusText(StatusCode: Cardinal): string;


implementation


procedure Register;
begin
  RegisterComponents('ZoC', [TZoCInetChecker]);
end;


{ TZoCInetChecker }
procedure InternetCallback(hInternet: HINTERNET; dwContext: DWORD;
  dwInternetStatus: DWORD; lpvStatusInformation: hwnd;
  dwStatusInformationLength: DWORD); stdcall;
var
  cpContext                             : pREQUEST_CONTEXT;
  FConnected                            : Boolean;
  dwindex, dwcodelen                    : dword;
  dwcode                                : array[1..20] of char;
  res                                   : pchar;
begin
  cpContext := pREQUEST_CONTEXT(dwContext);
  PostMessage(cpContext^.hWindow, WM_STATUSCHANGE, dwInternetStatus, 0);
  if dwInternetStatus = INTERNET_STATUS_REQUEST_COMPLETE then
  begin
    dwIndex := 0;
    dwCodeLen := 10;
    HttpQueryInfo(Pointer(LPINTERNET_ASYNC_RESULT(lpvStatusInformation).dwResult),
      HTTP_QUERY_STATUS_CODE, @dwcode, dwcodeLen, dwIndex);
    res := pchar(@dwcode);
    //HTTP_STATUS_OK 200 The request completed successfully
    //HTTP_STATUS_REDIRECT 302 The requested resource resides temporarily under a different URI
    FConnected := (res = '200') or (res = '302');
    InternetCloseHandle(Pointer(LPINTERNET_ASYNC_RESULT(lpvStatusInformation).dwResult));
    InternetCloseHandle(cpContext^.hOpen);
    PostMessage(cpContext^.hWindow, WM_CHECKCOMPLETE, Integer(FConnected), 0);
  end;
end;


constructor TZoCInetChecker.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FWindowHandle := AllocateHWnd(WndProc);
  FAccessType := atDirectConnect;
  FAsynRequest := True;
  FUserAgent := 'ZoCSoft Internet Checker 1.0';
end;


destructor TZoCInetChecker.Destroy;
begin
  InternetCloseHandle(hOpenUrl);
  if FWindowHandle <> 0 then
    DeallocateHWnd(FWindowHandle);
  inherited Destroy;
end;


procedure TZoCInetChecker.DoOnStatusChange(StatusCode: Cardinal);
begin
  if Assigned(FOnStatusChange) then
    FOnStatusChange(Self, StatusCode);
end;
function TZoCInetChecker.Execute: Boolean;
var
  Flags                                 : Cardinal;
begin
  if FUrl = '' then
    Exit;
  FBusy := True;
  if Assigned(FOnStart) then
    FOnStart(Self);
  if FAsynRequest then
    Flags := INTERNET_FLAG_ASYNC
  else
    Flags := 0;
  case FAccessType of
    atDirectConnect:
      hOpen := InternetOpen(PChar(FUserAgent), INTERNET_OPEN_TYPE_DIRECT,
        nil, nil, Flags);
    atPreConfig:
      hOpen := InternetOpen(PChar(FUserAgent), INTERNET_OPEN_TYPE_PRECONFIG,
        nil, nil, Flags);
    atPreConfigWithNoProxy:
      hOpen := InternetOpen(PChar(FUserAgent),
        INTERNET_OPEN_TYPE_PRECONFIG_WITH_NO_AUTOPROXY,
        nil, nil, Flags);
    atProxy:
      hOpen := InternetOpen(PChar(FUserAgent), INTERNET_OPEN_TYPE_PROXY,
        PChar(FProxy), nil, Flags);
  end;
  RC.hWindow := FWindowHandle;
  RC.hOpen := hOpen;
  if FAsynRequest then
    iscCallback := `InternetSetStatusCallback`(hOpen,
      INTERNET_STATUS_CALLBACK(@InternetCallback));
  hOpenUrl := InternetOpenUrl(hOpen, PChar(FUrl), nil, 0,
    INTERNET_FLAG_RELOAD, Cardinal(@RC));
  Result := hOpenUrl <> nil;
end;


procedure TZoCInetChecker.WndProc(var Msg: TMessage);
begin
  case Msg.Msg of
    WM_STATUSCHANGE:
      DoOnStatusChange(Msg.WParam);
    WM_CHECKCOMPLETE:
      begin
        FBusy := True;
        if Assigned(FOnComplete) then
          FOnComplete(Self, Boolean(Msg.WParam));
      end;
  else
    Msg.Result := DefWindowProc(FWindowHandle, Msg.Msg, Msg.wParam, Msg.lParam);
  end;
end;

function StatusCode2StatusText(StatusCode: Cardinal): string;
begin
  case StatusCode of
    INTERNET_STATUS_CLOSING_CONNECTION:
      Result := 'Closing connection to the server.';
    INTERNET_STATUS_CONNECTED_TO_SERVER:
      Result := 'Successfully connected to the `socket` address.';
    INTERNET_STATUS_CONNECTING_TO_SERVER:
      Result := 'Connecting to the `socket` address.';
    INTERNET_STATUS_CONNECTION_CLOSED:
      Result := 'Closed the connection to the server.';
    INTERNET_STATUS_DETECTING_PROXY:
      Result := 'Proxy has been detected.';
    INTERNET_STATUS_HANDLE_CLOSING:
      Result := 'Handle value has been terminated.';
    INTERNET_STATUS_HANDLE_CREATED:
      Result := 'InternetConnect has created a new handle.';
    INTERNET_STATUS_INTERMEDIATE_RESPONSE:
      Result := 'Received status code message from the server.';
    INTERNET_STATUS_NAME_RESOLVED:
      Result := 'Successfully found the IP address.';
    INTERNET_STATUS_RECEIVING_RESPONSE:
      Result := 'Waiting for the server to respond.';
    INTERNET_STATUS_REDIRECT:
      Result := 'Request is about to be redirected.';
    INTERNET_STATUS_REQUEST_COMPLETE:
      Result := 'Completed.';
    INTERNET_STATUS_REQUEST_SENT:
      Result := 'Successfully sent the information request to the server.';
    INTERNET_STATUS_RESOLVING_NAME:
      Result := 'Looking up the IP address of the name.';
    INTERNET_STATUS_RESPONSE_RECEIVED:
      Result := 'Successfully received a response from the server.';
    INTERNET_STATUS_SENDING_REQUEST:
      Result := 'Sending the information request to the server.';
    INTERNET_STATUS_STATE_CHANGE:
      Result := 'Security State Change.';
  else
    Result := 'Unknown Status.';
  end;
end;

end.
```

## 4、后记

不出所料，网络小灵通果然不负众望得到不少同学的支持，大家也不用坐在计算机前无聊地看网络是否恢复了，因为它不仅能检测内网是否连通，也能检测外网是否能上，在检测成功后它会播放自定义的声音通知，为此我还专门找寝室一个兄弟录制了两段粗犷的“通网了”/“断网了”音频为大家解闷；再后来还加入了查找本网段内空闲IP和检查哪个IP可能在搞ARP欺骗的功能，直到学校的网络秩序逐渐转好，网络状况也慢慢好转，网络小灵通才回到了我的Code Repository里面，安睡了。