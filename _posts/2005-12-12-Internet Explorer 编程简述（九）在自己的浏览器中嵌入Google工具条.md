---
layout:     post
title:      "Internet Explorer 编程简述（九）在自己的浏览器中嵌入Google工具条"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2005-12-12 22:05:00
author:     "eagleboost"
header-img: "img/post-bg-ocean.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - Google Toolbar
    - Explorer Bars
    - ToolBands
    - IObjectWithSite
    - IDeskBand
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2005年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/550698)

### 1. 概述

`Internet Explorer`强大而方便的可编程能力和可扩展能力为其抢占浏览器市场可谓是立下了汗马功劳。可编程主要体现两方面：

+ 实现浏览功能的部分被包装成一个控件——`WebBrowser Control`，开发人员可以在自己的应用程序中嵌入它从而使程序具有访问`Internet`上网页的能力，同时`WebBrowser Control`也能够被灵活地自定义以满足不同的需要。
  
+ 可对`Microsoft Internet Explorer`应用程序本身嵌入的浏览器控件编程，一般通过`BHO（Browser Helper Object）`来实现。
 

可扩展能力则体现在几个方面：

+ 嵌入式面板型扩展，包括`Explorer Bars`（如收藏夹、搜索、历史等嵌入`IE`主窗口的大型面板）, `Tool Bands`（如`Google Toolbar`、`MSN Toolbar`等嵌入`IE`的工具条）, 和`Desk Bands`（如快速启动这类嵌入任务栏的面板，实际上是`Explorer`外壳的扩展）。这几种面板的编写方法相差无几，不同之处主要在于向系统注册方式的不同。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Explorer_Bars.png)
`Explorer Bars`

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Tool_Bands.png)
`Tool Bands`

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Desk_Bands.png)
`Desk Bands`

+ 是参数型扩展，包括为浏览器增加上下文菜单项（调用脚本）、为浏览器主菜单增加菜单项、为浏览器“标准按钮”工具条添加按钮等。
  
+ 其他扩展，如文件下载的扩展（`Custom Download Manager`）、地址栏扩展（搜索扩展）等。

 
随着IE的发展，各种类型的扩展遍地开花，其中最为引人注目的，当属地址栏扩展和工具条扩展（几乎成了兵家必争之地）。本文讨论的主题，正是工具条的扩展。

> 此处有个小插曲。[csdn原文](https://blog.csdn.net/CathyEagle/article/details/550698)中的全部图片经已失效，之前的系列文章我都在[Google Image](https://www.google.com/search?q=Internet+Explorer+%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0%EF%BC%88%E4%B9%9D%EF%BC%89%E5%9C%A8%E8%87%AA%E5%B7%B1%E7%9A%84%E6%B5%8F%E8%A7%88%E5%99%A8%E4%B8%AD%E5%B5%8C%E5%85%A5Google%E5%B7%A5%E5%85%B7%E6%9D%A1&newwindow=1&source=lnms&tbm=isch&sa=X&ved=2ahUKEwi428Sk6c3pAhUF1qwKHdoiCRcQ_AUoAnoECBUQBA)找到了其他网站转载的图片，但这一篇死活找不到。好在最后发现“有人”把全文打包成`Word`文档放在[百度文库](https://wenku.baidu.com/view/8cfbd56eb84ae45c3b358ce8.html?pn=51)供人收费下载。我没有百度文库`VIP`下不了，所幸在线浏览还开放，于是屏幕截图了事。

> 尝试下载的时候还有如下界面，其中“版权人”的定义不详，但我想大概率不会是我。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Baidu_Wenku.png)


### 2. 问题的提出

两个原因促成了`Google Toolbar`的流行，一是广告窗口的泛滥、二是`Google Search`。`Google`“简单”（实则一点都不简单，没有搜索引擎的强力支持，`Toolbar`的用途就大受限制）地抓住了这两点，迅速占领了市场。

插件的一大好处在于可以不修改主程序，只需换一个样子差不多但功能更强的东西就可以使得整个应用程序功能增强，所以`IE`不升级大家也觉得`Google Toobar`越来越好用。于是利用`WebBrowser Control`编写浏览器的开发人员就想，如果能像`IE`一样支持上述这些扩展，不就能把`Google Toolbar`拿过来用了吗？其他的事交给`Google`去做就行了。这就是我们要讨论的问题，“如何在自己的浏览器中嵌入`Google Toolbar`”。


### 3. 分析

微软并未在`MSDN`中说明如何将`Google Toolbar`这类IE的工具条插件嵌入自己的应用程序，但其基于`COM`的设计方法实际上给予了我们这个能力（创建嵌入式的工具条的方法并不是本文的重点，此处略去，有兴趣的朋友可以参考`MSDN`）。我们知道，除了`IUnknown`接口外，`Bands`和`Bars`（以下简称Band对象）还需要实现三个接口：`IObjectWithSite`，`IPersistStream`和`IDeskBand`。当用户选择工具条或面板时，其容器（如IE的外壳框架）就会调用`Band`对象的`IObjectWithSite::SetSite`方法（该方法仅需要一个`IUnknown`类型的指针），将自己实现的`IUnknown`指针传递给`Band`对象。这就是整个插件开始真正激活的入口，也是我们的着手点。


`MSDN`中说到，一般来说，`Band`对象对于`SetSite`方法的实现需要完成以下几件事：
1. 如果当前`Band`对象持有另外的`Site`指针，则首先释放该指针。

2. 如果容器向`SetSite`方法传入的是一个空指针，则表示要删除该`Band`对象，此时`SetSite`返回`S_OK`即可。

3. 如果容器传入的不是空指针，则需要设置新的`Site`：

+ 对此`IUnknown`指针所指的新`Site`调用`QueryInterface`查询得到其`IOleWindow`接口。
  
+ 调用得到的`IOleWindow`接口的`GetWindow`方法获取父窗口的句柄（此窗口即是`Band`对象的栖身之处）并保存下来。如果以后不会再用到`IOleWindow`接口的话就对其调用`Release`。
  
+ 现在可以创建`Band`对象的窗口了，当然，要以第2步得到的窗口为父窗口来创建，并且该窗口目前只能以不可见状态存在。
  
+ 如果Band对象实现了`IInputObject`接口，即需要接收键盘输入，则还需要向容器传来的`Site`查询（`QueryInterface`）`IInputObjectSite`接口，此接口指针也需要保存下来。

+ 上述步骤完成后即可返回`S_OK`，否则应返回`OLE-defined`的`error code`告知容器什么地方出了错。
 

显然，就我们要讨论的问题而言，只需换个角度（编写IE外壳的的角度）来考虑即可。首先，我们需要一个`IUnknown`接口（即`Band`对象所需的`Site`），其次需要一个`IInputObjectSite`接口，用以和`Band`对象的`IInputObject`接口交互，处理输入焦点转移的情况。接下来就可以通过`Band`对象的`IDeskBand`接口来显示、隐藏`Band`对象了（注意`IDeskBand`接口派生自`IDockingWindow`接口,后者又派生自`IOleWindow`接口）。

### 4. 实现

实现分为两个部分，其一是一个简单的类，用以模拟IE外壳，我取名为`CIESimulator`。其二是一个管理`IE`扩展的类`CIEBandPlugInManager`，用以管理`Band`对象的方方面面。

```c++
class CIESimulator : public IInputObjectSite
{
private:
  IWebBrowser2 *m_pwb; //保存`WebBrowser Control`的接口指针
public:
  CIESimulator(void){};
  ~CIESimulator(void){};

  void SetIWebBrowser2(IWebBrowser2* pwb);

  //IUnknown methods
  STDMETHODIMP QueryInterface(REFIID, void **);

  STDMETHODIMP_(ULONG) AddRef(void);

  STDMETHODIMP_(ULONG) Release(void);

  //IInputObjectSite specific methods
  STDMETHOD(OnFocusChangeIS)(THIS_ IUnknown* punkObj, BOOL fSetFocus);
};

//IUnknown methods
STDMETHODIMP CIESimulator::QueryInterface( REFIID riid, void **ppv )
{
  if ( riid == IID_IInputObjectSite )  //这个接口需要自己处理
  {
    *ppv = static_cast<IInputObjectSite*>(this);
  }
  else if ( m_pwb )  //其它的交给`WebBrowser Control`去处理
  {
    m_pwb->QueryInterface( riid, ppv );
  }
  return S_OK;
}

//IInputObjectSite specific methods
STDMETHODIMP CIESimulator::OnFocusChangeIS(IUnknown* punkObj, BOOL fSetFocus)
{
  return S_OK;  //此处我们简单地返回
}

void CIESimulator::SetIWebBrowser2(IWebBrowser2* pwb)
{
  m_pwb = pwb;
}
```

注意这里我们并没有实现`IOleWindow`接口来向`Band`对象传递父窗口对象（窗口的宿主可以更改，所以`Band`对象创建的窗口的父窗口我们并不关心，`Band`对象查询`IOleWindow`接口的动作实际上是向`WebBrowser Control`查询），而是在稍后的`CIEBandPlugInManager`类中通过调用`IDeskBand`的`GetWindow`方法获得`Band`对象的窗口句柄，再手动将其嵌入我们指定的窗口中。

 
首先我们定义一个结构用以保存`Band`的信息：

```c++
enum eBANDTYPE
{
  btVertical = 0,
  btHorizontal = 1
};

 
enum eBANDSTATE
{
  bsUnInitialized = -1,
  bsVisible = 0,
  bsInVisible = 1,
  bsUnLoaded = 2
};
 
typedef struct tagIEBANDINFO
{
  char szCLSID[39];
  char szName[MAX_PATH];
  IUnknown *puk;
  HWND hBand;
  UINT uMinHeight;
  UINT uBandID;
  eBANDTYPE eBandType;
  eBANDSTATE eBandState;
} IEBANDINFO, *LPIEBANDINFO;
```

再用一个函数来获取所有`Band`的信息（以下代码为示例，具体实现是可从注册表把所有`Band`的信息一一读出）

```c++
void CIEBandPlugInManager::GetAllBandCLSID(void)
{
  LPIEBANDINFO pIEBandInfo;
  pIEBandInfo = new IEBANDINFO();

  strcpy( pIEBandInfo->szCLSID, "{2318C2B1-4965-11d4-9B18-009027A5CD4F}/0");  //Google Toolbar的CLSID

  strcpy( pIEBandInfo->szName, GetDisplayName(pIEBandInfo->szCLSID) );

  pIEBandInfo->uMinHeight = 22;
  pIEBandInfo->uBandID = m_BandCtrlID++;
  pIEBandInfo->eBandType = btHorizontal;
  pIEBandInfo->eBandState = bsUnInitialized;
  m_oaBand.Add( (CObject*)pIEBandInfo );//m_oaBand是一个CObArray
｝

//根据CLSID从注册表获取Band的名称
CString CIEBandPlugInManager::GetDisplayName(CString strCLSID)
{
  TCHAR sz[MAX_PATH];
  HKEY hKey;
  DWORD dwSize;

  strCLSID = "CLSID//" + strCLSID;

  if(RegOpenKey(HKEY_CLASSES_ROOT, strCLSID, &hKey) != ERROR_SUCCESS)
  {
    return _T("");
  }

  RegQueryValueEx(hKey, NULL, NULL, NULL, (LPBYTE)sz, &dwSize);

  RegCloseKey(hKey);

  return sz;
}

//通过Band的CLSID激活Band
bool CIEBandPlugInManager::ActivateBand( CString strCLSID )
{

  LPIEBANDINFO pIEBandInfo = GetBandInfo( strCLSID ); //从m_oaBand中找到符合条件的Band的信息
  if ( !pIEBandInfo )
  {
    return false;
  }

 
  WCHAR wsz[MAX_PATH]; 
  ::MultiByteToWideChar(CP_ACP, 0, strCLSID, -1, wsz, MAX_PATH);

  CLSID clsid;
  HRESULT hr2 = CLSIDFromString( wsz, &clsid);

  if ( hr2 != NOERROR )
    return false;

  HRESULT hr = ::CoCreateInstance(clsid, NULL, LSCTX_INPROC_SERVER, IID_IUnknown, (void**)&pIEBandInfo->puk); //创建Band对象的实例

  IUnknown* puk = pIEBandInfo->puk;
  if (FAILED(hr))
    return false;

  DoQueryBandInfo( pIEBandInfo );  //查询Band对象实例的信息

  switch( pIEBandInfo->eBandType )
  {
  case btVertical:
    break;
  //我们不处理Vertical的面板
  case btHorizontal:
  {
    g_pMainFrame->m_wndReBar.AddBar2( pIEBandInfo->hBand, pIEBandInfo->uBandID, pIEBandInfo->uMinHeight, 0, 0); //将Band嵌入主窗口的ReBar中

    REBARBANDINFO rbbi;
    rbbi.cbSize = sizeof(rbbi);
    rbbi.fMask = RBBIM_CHILDSIZE | RBBIM_IDEALSIZE | RBBIM_SIZE;
    rbbi.cxMinChild = 0;
    rbbi.cyMinChild = pIEBandInfo->uMinHeight;
    rbbi.cx = rbbi.cxIdeal = 250;
    UINT nCount = g_pMainFrame->m_wndReBar.GetReBarCtrl().GetBandCount();
    g_pMainFrame->m_wndReBar.GetReBarCtrl().SetBandInfo(nCount-1, &rbbi);
    break;
  }
  default:
    break;
  }

  pIEBandInfo->eBandState = bsVisible;
  return true;
}

//查询Band对象实例的信息
void CIEBandPlugInManager::DoQueryBandInfo(LPIEBANDINFO pIEBandInfo)
{
  IObjectWithSite *pOWS;

  //查询IObjectWithSite接口
  HRESULT hr = pIEBandInfo->puk->QueryInterface(IID_IObjectWithSite, (void**)&pOWS);
  if (SUCCEEDED(hr))
  {     //设置Site
    pOWS->SetSite( (IUnknown *)&m_IESimulator ); //m_IESimulator是CIESimulator的一个实例对象，对Band对象而言，它就像IE
  }
 
  IDeskBand *pdb;
  hr = pIEBandInfo->puk->QueryInterface(IID_IDeskBand, (void**)&pdb);
  if (SUCCEEDED(hr))
  {     //查询得到Band对象窗口的句柄，稍候通过该句柄将Band对象的窗口嵌入我们指定的窗口
    pdb->GetWindow(&pIEBandInfo->hBand);
  }

  ShowBand(pIEBandInfo, TRUE);//显示Band
}
 
bool CIEBandPlugInManager::ShowBand(LPIEBANDINFO pIEBandInfo, bool bShow)
{
  IDockingWindow *pdw;

  HRESULT hr = pIEBandInfo->puk->QueryInterface(IID_IDockingWindow, (void**)&pdw);
  if (SUCCEEDED(hr))
  {
    pdw->ShowDW(bShow);
  }
  else
  {
    return false;
  }

  return true;
}
```

下面是我测试把`Google`工具条嵌入我自己写的浏览器的一个截图，`Google`的搜索、广告窗口拦截均可正常工作。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Embedded_Google_Toolbar_Demo.png)


### 5. 总结

上述的原理看来很简单，但具体实现的时候仍然需要作较多的测试和考虑，`Band`对象的管理和缓存、接口的`AddRef`和`Release`等。时间有限，代码也很乱，不过只要原理交待清楚，相信会对有兴趣的朋友有所帮助。

### 6. 参考资料

MSDN:《Creating Custom Explorer Bars, Tool Bands, and Desk Bands》(原链接已失效)