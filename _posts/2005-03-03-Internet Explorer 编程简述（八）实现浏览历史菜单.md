---
layout:     post
title:      "Internet Explorer 编程简述（八）实现浏览历史菜单"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2005-03-03 00:36:00
author:     "eagleboost"
header-img: "img/post-bg-lights-on-trees.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - ITravelLogStg
    - IEnumTravelLogEntry
    - ITravelLogEntry
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2005年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/308620)

### 1. 概述

`Internet Explorer`的浏览历史菜单在4.0版本开始出现，但直到5.5之前，微软都未公布用于访问浏览历史的COM接口，如今已是IE6.0大行其道的年代，用于访问浏览历史的接口也早已公布多时，本文的目的则是试图抛砖引玉，简单介绍用于访问浏览历史的`Travel Log`接口，并用一个小小的类`CIETravelLog`来实现对`Travel Log`的封装。

### 2. `IOmHistory`接口

在早些时候的`MSDN`中，我们能够查阅到关于浏览历史的接口仅有`IOmHistory`，而该接口实际上对应的是浏览器中可以通过脚本访问的“history”对象。对于“history”对象，`MSDN`中是这样说的：
 
> For security reasons, the history object does not expose the actual URLs in the browser history. It does allow navigation through the browser history by exposing the back, forward, and go methods. A particular document in the browser history can be identified as an index relative to the current page. For example, specifying -1 as a parameter for the go method is the equivalent of clicking the Back button.
> 
> This object is available in script as of Microsoft Internet Explorer 3.0.
 
即为了安全的原因，`IOmHistory`接口仅提供了有限的几个方法来完成在浏览器中前进、后退等操作，并没有提供访问历史列表Url的能力。这也难怪，该接口在IE 3.0时代已经存在，而当时IE并不成熟，可编程能力也不甚强大。一直到IE 4.0通过与Windows 98捆绑销售一统天下之后，相关的文档才逐渐丰富，多窗口浏览器等基于`Internet Explorer`/`WebBrowser Control`的应用软件也才铺天盖地开来。但在IE 5.5接口公布之前，要完全模拟IE的`Travel Log`行为，并不是一件容易的事。最容易想到的方法就是在`BeforeNavigate`、`DocumentComplete`等事件发生之时记录/修改Url并加以保存（我在早些时候也这样做过），但是效果不甚理想，尤其是浏览包含Frame的网页时，处理更是麻烦。当然，要完全模拟亦非难事，只不过开发人员都知道微软公布接口是早晚的事，所以也没有人花大力气在模拟IE的`Travel Log`行为上。


### 3. `Travel Log`简介

`Internet Explorer` 5.5推出以后，`Travel Log`接口也就开始出现在`MSDN`中，它是专门为OLE嵌入`WebBrowser Control`的应用程序设计的，其目的是“提高和加强用户的访问日志体验”（improve and enhance the user's travel log experience）。事实上，稍后我会提到，`Travel Log`接口正日益成为应用程序中的重要接口之一。

微软公布的`Travel Log`共包含三个接口：`ITravelLogStg`, `IEnumTravelLogEntry`和`ITravelLogEntry`。

+ `ITravelLogStg`——该接口提供了用于在`Travel Log`中添加、删除、枚举日志（浏览历史）的方法，本文需要用到的几个方法列举如下：


|方法名    | 用途|
|--|--|
|EnumEntries |为访问日志项创建枚举器（`IEnumTravelLogEntry`接口指针）|
|`GetRelativeEntry` |返回一个日志项 |
|`TravelTo` |访问一个日志项|

+ `IEnumTravelLogEntry`——该接口提供用于枚举日志项所必需的方法，本文只用到一个方法：

|方法名    | 用途|
|--|--|
|Next   | 枚举下一个日志项（返回`ITravelLogEntry`接口指针） |

+ `ITravelLogEntry`——该接口只有两个方法，分别用于返回日志项的Title和Url：

|方法名    | 用途|
|--|--|
|GetTitle  |返回日志项的Title|
|GetURL  |返回日志项的Url|

接口准备好了，我们也就很容易得知它们之间的关系：
+ 要得到相对于当前页面的日志项列表，首先应通过`ITravelLogStg`接口创建一个枚举器（`IEnumTravelLogEntry`接口）。 
  
+ 通过`IEnumTravelLogEntry`枚举器的Next方法枚举出一个个的日志项（`ITravelLogEntry`接口）。 
  
+ 由`ITravelLogEntry`接口获取日志项所代表的网页的Title和Url并加以处理。 
  
+ 访问相对于当前页面的某个日志项时，首先由`ITravelLogStg`的`GetRelativeEntry`方法根据与当前页的距离得到`ITravelLogEntry`接口，再将后者传入`ITravelLogStg`的`TravelTo`方法以达到访问日志项的目的（如前进和后退）。

### 4. 封装`Travel Log`

接下来，我们就用一个简单的类来完成对`Travel Log`的封装。如下所示，`tlogstg.h`包含了`Travel Log`的相关接口声明，该头文件可以在`Platform SDK`中找到。

```c++
#include "tlogstg.h"
 
class CIETravelLog
{
  private:
    ITravelLogStg *m_pTravelLogStg;
    IEnumTravelLogEntry *m_pEnumLogEntry;
    ITravelLogEntry *m_pTravalLogEntry;
    IWebBrowser2* m_pWebBrowser;
  public:
    CIETravelLog(void);
    ~CIETravelLog(void);
    void SetWebBrowser(IWebBrowser2* pWebBrowser);
    void BuildHistoryMenu(CMenu * pMenu, unsigned char nCount, bool bForward);
    void TravelTo(int nPosition);
};
 
CIETravelLog::CIETravelLog(void)
: m_pTravelLogStg(NULL), m_pEnumLogEntry(NULL), m_pTravalLogEntry(NULL), m_pWebBrowser(NULL)
{
}
 
CIETravelLog::~CIETravelLog(void)
{
  if ( m_pTravelLogStg != NULL )
  {
    m_pTravelLogStg->Release();
  }

  if ( m_pEnumLogEntry != NULL )
  {
    m_pEnumLogEntry->Release();
  }

  if ( m_pTravalLogEntry != NULL )
  {
    m_pTravalLogEntry->Release();
  }

  if ( m_pWebBrowser != NULL )
  {
    m_pWebBrowser->Release();
  }
}
 
//将浏览器的IWebBrowser2接口指针赋予CIETravelLog的实例
void CIETravelLog::SetWebBrowser(IWebBrowser2* pWebBrowser)
{
  if ( (m_pWebBrowser == pWebBrowser) || (m_pWebBrowser == NULL) )
  {
    return;
  }

  if ( m_pWebBrowser != NULL )
  {
    m_pWebBrowser->Release();
  }

  m_pWebBrowser = pWebBrowser;
  
  IServiceProvider *pSP;
  HRESULT hr = m_pWebBrowser->QueryInterface(IID_IServiceProvider, (LPVOID*)&pSP);
  m_pWebBrowser->Release();
  if (SUCCEEDED(hr))
  {
    hr = pSP->QueryService(SID_STravelLogCursor, IID_ITravelLogStg, (LPVOID*)&m_pTravelLogStg);
  pSP->Release();
  }
}
 
//创建浏览历史菜单，bForward指明是前进还是后退菜单
void CIETravelLog::BuildHistoryMenu(CMenu * pMenu, unsigned char nCount, bool bForward)
{
  if ( m_pTravelLogStg == NULL )
  {
    return;
  }

  TLENUMF eFlag = bForward ? TLEF_RELATIVE_FORE : TLEF_RELATIVE_BACK;
  if ( FAILED(m_pTravelLogStg->EnumEntries( eFlag, &m_pEnumLogEntry ) ) )
  {
    return;
  }
 
  ULONG uFetched;
  int i=0;
  if ( m_pEnumLogEntry !=NULL )
  {
    while ( SUCCEEDED( m_pEnumLogEntry->Next( 1, &m_pTravalLogEntry, &uFetched ) ) && 
    m_pTravalLogEntry && i<10 )//我们最多只需要10条历史菜单，可根据实际情况修改
    {
      LPOLESTR pszTitle;
      m_pTravalLogEntry->GetTitle( &pszTitle );
      CString strTitle = pszTitle;
      if ( bForward )
      {
        //ID_IEHISTORY_MIDDLE是预定义的某个菜单项ID，从该ID开始前后可以创建10个菜单项，参见下一节
        pMenu->InsertMenu( 0, MF_STRING, ID_IEHISTORY_MIDDLE + ++i, strTitle );
      }
      else
      {
        pMenu->InsertMenu( 0, MF_STRING, ID_IEHISTORY_MIDDLE - ++i, strTitle );
      }

      CoTaskMemFree( pszTitle );
      m_pTravalLogEntry->Release();
    }
  }
}
 
//根据与当前页面的相对距离来访问历史网页
void CIETravelLog::TravelTo(int nPosition)
{
  if ( m_pTravelLogStg == NULL )
  {
    return;
  }

  if SUCCEEDED(m_pTravelLogStg->GetRelativeEntry( nPosition, &m_pTravalLogEntry ))
  {
    m_pTravelLogStg->TravelTo( m_pTravalLogEntry );
  }
}
```

### 5. 使用`CIETravelLog`

假设是在我们自己编写的多窗口浏览器中使用`Travel Log`。为简单起见，我们声明一个`CIETravelLog`的全局对象`g_IETravelLog`，以便在任何地方调用。然后在适当的地方，如`CMainFrame`的`TBN_DROPDOWN`消息（工具条菜单下拉消息）处理函数`OnDropDown`中，添加下面的代码，用以创建浏览历史菜单：

```c++
//GetActiveWebBrowserPtr返回活动的浏览器IWebBrowser2接口指针
IETravelLog.SetWebBrowser( GetActiveWebBrowserPtr );
//bForward为true则创建“前进”菜单，否则创建“后退”菜单
IETravelLog.BuildHistoryMenu( &Menu, 10, bForward);
```

以下定义为菜单项ID的范围，前后共可以容纳10个菜单项，可根据实际情况修改。

```c++
#define ID_IEHISTORY_FIRST  60200
#define ID_IEHISTORY_MIDDLE 60210
#define ID_IEHISTORY_LAST   60220
```

添加命令处理函数`OnTravelHistoryUrl`用以响应从`ID_IEHISTORY_FIRST`到`ID_IEHISTORY_LAST`的菜单命令。

```c++
ON_COMMAND_RANGE(ID_IEHISTORY_FIRST, ID_IEHISTORY_LAST, OnTravelHistoryUrl)
 
void CMainFrame::OnTravelHistoryUrl(UINT nID /* Command ID */)
{
  //nID - ID_IEHISTORY_MIDDLE即为要访问的浏览历史到当前页面的距离
  g_IETravelLog.TravelTo( nID - ID_IEHISTORY_MIDDLE );
}
```

### 6. 再谈`Travel Log`

前面我提到“`Travel Log`接口正日益成为应用程序中的重要接口之一”，此处加以说明。从微软平台的开发模式及导向来看，基于`Internet Explorer`/`WebBrowser Control`的应用势必会成为主流。在下一代的操作系统Longhorn中，应用程序界面的描述将完全由XML的一个特化——XAML来完成，而XAML的解析将由浏览器完成。微软说未来应用程序的部署将会十分容易，本地应用和基于浏览器的应用之间的差异将会被逐渐淡化，而实现这一目标的一个重要表现就是，在将来的操作系统平台上，应用程序实际上时刻都将运行在`Internet Explorer`中，`Internet Explorer`在某种程度上来说变成了一个容器。

于是，扎根于`Internet Explorer`的`Travel Log`自然而然地就被整合到了我们的应用程序中。君不见，我们每天在资源管理器和浏览器上完成的工作，不就是在`Travel Log`中来来回回地跑吗？如果所有的应用程序都嵌入到`Internet Explorer`中运行，那么我们在应用程序中所作的操作便自然得到了记录，“前进”和“后退”也就很Easy了。

很多软件都已经或多或少地开始采用基于`Internet Explorer`的模式，如`Microsoft Money`、`Microsoft Encarta`、`Visual Studio.net`的安装程序等等，都是很好的范例。所以，就目前来说，将我们的应用程序按这种模式编写（可参考《[利用浏览器实现程序界面与实现的分离](https://eagleboost.com/2004/08/09/0-利用浏览器实现程序界面与实现的分离/)》），不是可以早一点获得“访问日志的体验”吗？