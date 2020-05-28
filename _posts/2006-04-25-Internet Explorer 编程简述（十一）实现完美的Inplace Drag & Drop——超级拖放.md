---
layout:     post
title:      "Internet Explorer 编程简述（十一）实现完美的Inplace Drag & Drop——超级拖放"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2006-04-25 20:52:00
author:     "eagleboost"
header-img: "img/post-bg-mountain-night.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - 超级拖放
    - GetDropTarget
    - ondragover
    - IHTMLDataTransfer
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2006年在csdn发布的博客](https://blog.csdn.net/CathyEagle/article/details/677425)

### 1. 概述

许多多窗口浏览器都提供了一种被称为“超级拖放”（或“超级拖拽”、“随心拖放”等等，不一而足）的功能。作为对IE拖拽行为对扩展，“超级拖放”实现了一些非常实用的功能：

+ 拖放网页链接：通常是在新窗口中打开
  
+ 拖放选中的文字：保存文字、作为关键字通过搜索引擎搜索网络、作为Url打开等
  
+ 拖放图片：通常是保存图片到指定文件夹
  
+ 当然，还有很关键的一点：拖动对象时鼠标指针反馈不同的拖拽效果
  
在《[Internet Explorer 编程简述（十）响应来自HTML Element的事件通知——几个好用的类](https://eagleboost.com/2006/03/11/Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E5%8D%81-%E5%93%8D%E5%BA%94%E6%9D%A5%E8%87%AAHTML-Element%E7%9A%84%E4%BA%8B%E4%BB%B6%E9%80%9A%E7%9F%A5-%E5%87%A0%E4%B8%AA%E5%A5%BD%E7%94%A8%E7%9A%84%E7%B1%BB/)》中曾提到，尽管许多浏览器都提供了超级拖放的功能，但与IE的缺省实现相比，除了具备鼠标指针拖拽效果外，还没有哪个浏览器的实现能够实现：
+ 文字在页面内与输入框之间的交互拖放（这一点最为重要）
  
+ 来自外部的文字与网页输入框之间的交互拖放
 
+ 拖拽时滚动页面（这一点是被忽略了）
 
本文的目的，一是介绍实现超级拖放的两种方法，二是说明如何实现“完美”的拖放——即扩展IE拖拽行为的同时，保留IE默认的拖拽行为。三是给出一个最为直接和简洁的实现，至于拖放不同的对象以实现不同的功能，不在本文讨论的范围，略去。

### 2. 标准的实现方法

标准方法即通过`IDocHostUIHandler`的`GetDropTarget`成员函数来实现，在MSDN这样说到：

> **`IDocHostUIHandler::GetDropTarget`** Method——Called by **`MSHTML`** when it is used as a Drop target. This method enables the host to supply an alternative IDropTarget interface.

即在适当的时候，`MSHTML`引擎会调用`IDocHostUIHandler`的`GetDropTarget`方法，为应用程序提供一个机会来替换`MSHTML`缺省的`DropTarget`实现。我们就可以通过这个自定义的`DropTarget`实现来完成上述的“超级拖放”功能。方法示例如下，其中略去的部分可参考`MFC`中`CHtmlControlSite`和`CHtmlView`的源代码：

```c++
STDMETHODIMP CHtmlControlSite::XDocHostUIHandler::GetDropTarget(LPDropTarget pDropTarget, LPDropTarget* ppDropTarget)
{
  METHOD_PROLOGUE_EX_(CHtmlControlSite, DocHostUIHandler)
  *ppDropTarget = g_pDropTarget;//将自定义的实现告知MSHTML引擎
  return S_OK;
}
```

其中`g_pDropTarget`指向某个全局的`IDropTarget`接口的实现，我们假定为`CIEDropTarget`，`CIEDropTarget`实现了`IDropTarget`的几个成员函数`DragEnter`、`DragOver`、`DragLeave`和`Drop`。在`DragEnter`中可以决定是否接受一个`Drop`以及如果接受这个`Drop`的话该提供怎样的鼠标拖拽反馈，在持续触发的`DragOver`中同样可以设定鼠标拖拽反馈，从而实现在拖放不同的对象（文字、链接、图像等）时提供不同的拖拽视觉效果，实现相当简单，此处不再赘述。
但上面的实现存在一些问题。首先是选中的文字在页面内与输入框之间交互的拖放没有了。这是自然的，既然我们用自定义的`DropTarget`替换掉了`IE`的缺省实现，那这种交互的拖放理应由我们自己实现。难处并非在于不能实现，而是在于实现起来比较麻烦——光是得到鼠标下的`HTML Element`就够我们烦了；当输入框中有文字的时候，光标还应该随着鼠标的移动而移动——所以这个费力还不一定讨好的功能似乎没有哪个浏览器去做。其次，作为输入框文字拖放的衍生物，拖拽滚动没有了。当鼠标向某个方向拖拽时，网页应该随着将不可见的部分滚动出来，比如某个输入框，让我们有机会将文字拖拽过去。这个`Feature`的实现并不困难，不过一来是被忽略了（注意到拖拽滚动的人并不多），二来主要`Feature`都没有实现，这个滚动也意义不大了。

### 3. 打入`MSHTML`内部

既然从`GetDropTarget`提供外部实现难以得到与输入框的交互式拖放，那就换个角度来考虑问题，让我们打入`MSHTML`的内部。
着手点是`IHTMLDocumentX`接口——操纵`IE`的`DOM`的法宝。我们注意到`IHTMLDocument2`有个`ondragstart`事件，进而想到应该也有诸如`ondragenter`、`ondragover`、`ondrop`之类的事件（事实上也是有的），如果响应这些事件，处理同输入框的交互式拖放应该就能够解决。因为这些拖放在`MSHTML`的缺省`DropTarget`实现中发生，因而当鼠标拖拽到某个输入框上时，肯定会触发一个`ondragover`事件，而在`IHTMLEventObj`的辅助下我们能轻松得到相关的`HTML Element`，其它的操作就容易进行了。再细心一点，我们还发现`IHTMLEventObj2`接口有个`dataTransfer`属性——可以得到一个`IHTMLDataTransfer`的指针，而`IHTMLDataTransfer`接口正是浏览器内部用于数据交换的重要手段之一（看看它的属性就知道会很有用了）：

`IHTMLdataTransfer` Members

| Method        | Description                  |
|---------------|---------|
| **clearData**     | Removes one or more data formats from the clipboard through dataTransfer or clipboardData object                |
| **DropEffect**    | Sets or retrieves the type of drag-and-Drop operation and the type of cursor to display                         |
| **effectAllowed** | Sets or retrieves, on the source element,which data transfer operations are allowed for the object              |
| **getData**       | Retrieves the data in the specified format from the clipboard through the dataTransfer or clipboardData objects |
| **setData**       | Assigns data in a specified format to the dataTransfer or clipboardData object                                  |


更进一步，从`IHTMLDataTransfer`接口还可以访问到`IDataObject`接口，在进行`Ole`拖放时，数据就是通过`IDataObject`接口来传递的。具体用法稍后讨论。

### 4. 思路

提供鼠标反馈效果与实现`GetDropTarget`的方法类似，有了`IHTMLDataTransfer`接口，便可在`ondragstart`及`ondragover`事件触发时通过`DropEffect`属性设置拖拽的效果（可根据需要自行设定，不设置的话使用默认的效果）。再者，“拖”和“放”都在`MSHTML`的缺省实现中发生，我们从`IHTMLEventObj`的`SrcElement`即可得知鼠标所位置的`HTML Element`是否是输入框。

### 5. 实现

要接收到`ondragstart`之类的事件，可以采用《[Internet Explorer 编程简述（十）响应来自HTML Element的事件通知——几个好用的类](https://eagleboost.com/2006/03/11/Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E5%8D%81-%E5%93%8D%E5%BA%94%E6%9D%A5%E8%87%AAHTML-Element%E7%9A%84%E4%BA%8B%E4%BB%B6%E9%80%9A%E7%9F%A5-%E5%87%A0%E4%B8%AA%E5%A5%BD%E7%94%A8%E7%9A%84%E7%B1%BB/)》中提到的`CHtmlObj`类和`CHtmlElements`类，并在适当的地方连接到`Document`，示例代码如下所示：

```c++
HRESULT CHtmlDocument2::OnInvoke(DISPID dispidMember, REFIID riid, LCID lcid, WORD wFlags,
DISPPARAMS * pdispparams, VARIANT * pvarResult,EXCEPINFO * pexcepinfo,
UINT * puArgErr)
{
  ......
  //如果只是要设置鼠标拖拽效果的话，这个事件可以不处理
  case DISPID_HTMLELEMENTEVENTS_ONDRAGSTART :
  {
    OnDragStart();
    break;
  }
  //重点在这里
  case DISPID_HTMLELEMENTEVENTS_ONDRAGOVER :
  {
    OnDragOver();
    break;
  }
  case DISPID_HTMLELEMENTEVENTS_ONDROP :
  {
    OnDrop();
    break;
  }
  ......
}

void CHtmlDocument2::OnDragOver(void)
{
  SetDragEffect();               //设置鼠标拖拽效果
}
 
void CHtmlDocument2::SetDragEffect(void)
{
  CComQIPtr<IHTMLWindow2>  pWindow;
  CComQIPtr<IHTMLEventObj>  pEventObj;
  CComQIPtr<IHTMLEventObj2>  pEventObj2;
  CComQIPtr<IHTMLElement>  pElement;
   
  HRESULT hr = m_spHtmlObj->get_parentWindow( &pWindow );
  hr = pWindow->get_event( &pEventObj );
   
  //ondragover发生时IE的默认行为是“没有鼠标拖拽效果”。
  //将IHTMLEventObj的返回值设为false即可取消该事件的默认行为，所以执行完下面这句话，拖拽效果就出现了。
  AllowDisplayDragCursor(pEventObj, FALSE);  
   
  CComBSTR bstrTagName;
  pEventObj->get_srcElement(&pElement);    //获得当前HTML Element
  pElement->get_tagName(&bstrTagName);    
  if (IsEditArea(bstrTagName)) //根据Tag Name判断是否鼠标位于输入框，以便设置焦点使得光标随鼠标移动
  {
    CComQIPtr<IHTMLElement2>  pElement2;
    if (SUCCEEDED(pElement->QueryInterface(IID_IHTMLElement2, (void **) &pElement2 )) && pElement2)
    {
      pElement2->focus();
    }
    //默认情况下，当拖拽文档到输入框时，鼠标会变成拖拽的光标，所以这里使用IE的默认行为。
    AllowDisplayDragCursor(pEventObj, TRUE);
  }
}
 
BOOL CHtmlDocument2::IsEditArea(CComBSTR bstrTagName)
{
  return bstrTagName == "INPUT" || bstrTagName == "TEXTAREA";
}
 
void CHtmlDocument2::AllowDisplayDragCursor(CComQIPtr<IHTMLEventObj> pEventObj, BOOL bAllow)
{
  VARIANT v;
  v.vt = VT_BOOL;
   
  v.boolVal = !bAllow ? VARIANT_FALSE : VARIANT_TRUE;
  pEventObj->put_returnValue(v);
}
 
void CHtmlDocument2::OnDrop(void)
{
  CComQIPtr<IHTMLWindow2>  pWindow;
  CComQIPtr<IHTMLEventObj>  pEventObj;
  CComQIPtr<IHTMLEventObj2>  pEventObj2;
  CComQIPtr<IHTMLElement>  pElement;
  CComQIPtr<IHTMLDataTransfer>   pdt; //此处演示如何使用IHTMLDataTransfer
   
  HRESULT hr = m_spHtmlObj->get_parentWindow( &pWindow );
  hr = pWindow->get_event( &pEventObj );
  hr = pEventObj->QueryInterface(IID_IHTMLEventObj2, (void **) &pEventObj2 );
  hr = pEventObj2->get_dataTransfer(&pdt);
   
  CComBSTR bstrFormat = "URL"; //首先尝试获取URL
  VARIANT Data;
  hr = pdt->getData(bstrFormat, &Data);
  if (Data.vt != VT_NULL )
  {     //获取成功，拖放的对象是Url
    DoOpenUrl(CString(Data.bstrVal));
  }
  else
  {     //否则尝试获取选中的文本
    bstrFormat = "Text";
    hr = pdt->getData(bstrFormat, &Data);
    if (Data.vt != VT_NULL )
    {     //获取成功，拖放的内容是文本
      CComBSTR bstrTagName;
      pEventObj->get_srcElement(&pElement);
      pElement->get_tagName(&bstrTagName);
      if (IsEditArea(bstrTagName))
      {
        //Drop target是输入框，不做任何操作，由IE进行默认处理
        return;
      }
      else
      { //否则我们自己处理文本，或保存，或检测是否链接后打开，等等
        DoProcessText(CString(Data.bstrVal));
        //Process the text
      }
    }
    else
    {     //既不是链接，也不是文本，可认为是来自外部（如Windows Shell）的文件拖放
      DoOnDropFiles(pdt);
    }
  }
}
 
//演示如何从IHTMLDataTransfer得到IDataObject
void CHtmlDocument2::DoOnDropFiles(CComQIPtr<IHTMLDataTransfer> pDataTransfer)
{
  CComQIPtr<IServiceProvider>  psp;
  CComQIPtr<IDataObject>  pdo;
  if (FAILED(pDataTransfer->QueryInterface(IID_IServiceProvider, (void **) &psp)))
  {
    return;
  }
  if (FAILED(psp->QueryService(IID_IDataObject, IID_IDataObject, (void **) &pdo)))
  {
    return;
  }
  
  COleDataObject DataObject;
  DataObject.Attach(pdo);
  ......
}
```

### 6. 再次回到标准方法


上述通过`Event Sink`响应网页拖拽的方法已经能够很好地工作，可说“趋于完美”了，但仍有两个“小”问题：第一，必须与`document`建立连接才能工作，而建立连接的时机不容易掌握（`MSDN`中推荐的位置是`DocumentComplete`，但在`NavigateComplete`中也可，或者是检测到`WebBrowser`的`readystate`变为`READYSTATE_INTERACTIVE`时进行连接）。第二，实现方法还是略显复杂。
有没有更简单的方法呢？我决定再次对`GetDropTarget`进行“调研”。所谓“踏破铁鞋无觅处，得来全不费功夫”，晃了一眼`GetDropTarget`方法的声明后，灵机一动，我忽然想到了办法。事实证明，这是完美的解决办法。
 
让我们再来看看`GetDropTarget`的声明，其中第一个参数指向`MSHTML`提供的缺省`DropTarget`实现，而第二个参数用以返回应用程序的自定义`DropTarget`实现，如果在`GetDropTarget`中返回`S_OK`，`MSHTML`将以应用程序提供的自定义`DropTarget`替换缺省的`DropTarget`实现。


```c++
HRESULT GetDropTarget( IDropTarget *pDropTarget, IDropTarget **ppDropTarget);

pDropTarget

[in] Pointer to an IDropTarget interface for the current drop target object supplied by MSHTML.

ppDropTarget

[out] Address of a pointer variable that receives an IDropTarget interface pointer for the alternative drop target object supplied by the host.
```

想到了吗？解决问题的关键就在于第一个参数`pDropTarget`。相信很多浏览器在处理的时候都忽略掉了第一个参数而只是将自己的实现通过第二个参数告知`MSHTML`，因而丢失了`IE`缺省的行为。既然如此，将缺省的`IDropTarget`接口的指针保存下来，在适当的时候调用，不就能够保留IE的原始拖放行为了吗？

### 7. 完美实现

完整的代码就不再给出，我们只列出关键的部分作为示例。假设我们用来实现I`DropTarget`接口的类叫做`CBrowserDropTarget`：

```c++
//构造函数，传入参数即是从GetDropTarget得到的那个pDropTarget，它是MSHTML的缺省实现
CBrowserDropTarget::CBrowserDropTarget(IDropTarget *pOrginalDropTarget) :  m_bDragTextToInputBox(FALSE)
//这个布尔变量用来判断是否正在向InputBox拖拽文字
,  m_pOrginalDropTarget(pOrginalDropTarget)
//m_pOrginalDropTarget用来保存MSHTML的缺省实现
{
}
 
STDMETHODIMP CBrowserDropTarget::DragEnter(/* [unique][in] */IDataObject __RPC_FAR *pDataObj,
/* [in] */ DWORD grfKeyState,
/* [in] */ POINTL pt,
/* [out][in] */ DWORD __RPC_FAR *pdwEffect)
{
  //调用缺省的行为
  return m_pOrginalDropTarget->DragEnter(pDataObj, grfKeyState, pt, pdwEffect);
}
 
STDMETHODIMP CBrowserDropTarget::DragOver(/* [in] */ DWORD grfKeyState,
/* [in] */ POINTL pt,
/* [out][in] */ DWORD __RPC_FAR *pdwEffect)
{
  //在网页内拖拽文字时这个值是DROPEFFECT_COPY（拖拽的文字不属于输入框中）
  //或DROPEFFECT_COPY | DROPEFFECT_MOVE（拖拽的文字是输入框中的文字）
  DWORD dwTempEffect = *pdwEffect;
  
  //接下来调用IE的缺省行为
  HRESULT hr = m_pOrginalDropTarget->DragOver(grfKeyState, pt, pdwEffect);
  
  //判断是否是往输入框拖拽文字
  m_bDragTextToInputBox = IsDragTextToInputBox(dwOldEffect, *pdwEffect);
  if ( !m_bDragTextToInputBox )
  {
    //不是往输入框拖拽文字，则使用原始的拖拽效果。否则和IE的缺省效果一样——也就是没有效果
    *pdwEffect = dwTempEffect;
  }
  return S_OK;
}
 
//根据调用缺省行为前后的Effect值判断是否是往输入框拖拽文字
BOOL CBrowserDropTarget::IsDragTextToInputBox(DWORD dwOldEffect, DWORD dwNewEffect)
{
  //如果是把非输入框中文字往输入框拖动，则dwOldEffect与dwNewEffect相等，都是DROPEFFECT_COPY
  BOOL bTextSelectionToInputBox = ( dwOldEffect == DROPEFFECT_COPY ) && ( dwOldEffect == dwNewEffect );
  
  //如果是把文字从一个输入框拖到另一个输入框，则dwOldEffect为DROPEFFECT_COPY | DROPEFFECT_MOVE，
  //而dwNewEffect的值可能为DROPEFFECT_MOVE（默认情况），也可能为DROPEFFECT_COPY（按下Ctrl键时）
  BOOL bInputBoxToInputBox = ( dwOldEffect == (DROPEFFECT_COPY | DROPEFFECT_MOVE) ) && ( dwNewEffect == DROPEFFECT_MOVE || dwNewEffect == DROPEFFECT_COPY );
  
  //来自Microsoft Word的拖拽特殊一些，dwOldEffect是所有效果的组合值
  BOOL bMSWordToInputBox = ( dwOldEffect == (DROPEFFECT_COPY | DROPEFFECT_MOVE | DROPEFFECT_LINK) ) && ( dwNewEffect == DROPEFFECT_MOVE || dwNewEffect == DROPEFFECT_COPY );
  
  //来自Edit Plus的拖拽过也特殊一些，dwOldEffect是个负数（怀疑是Edit Plus的拖拽实现有问题）
  BOOL bEditPlusToInputBox = ( dwOldEffect < 0 ) && ( dwNewEffect == DROPEFFECT_MOVE || dwNewEffect == DROPEFFECT_COPY );
  
  //也许还有些例外，可再添加
  ......
  return bTextSelectionToInputBox || bInputBoxToInputBox || bMSWordToInputBox || bEditPlusToInputBox;
}
 
STDMETHODIMP CBrowserDropTarget::DragLeave()
{
  //调用缺省的行为
  return m_pOrginalDropTarget->DragLeave();
}
 
STDMETHODIMP CBrowserDropTarget::Drop(/* [unique][in] */ IDataObject __RPC_FAR *pDataObj,
/* [in] */ DWORD grfKeyState,
/* [in] */ POINTL pt,
/* [out][in] */ DWORD __RPC_FAR *pdwEffect)
{
  if ( m_bDragTextToInputBox )
  {
    //是文字拖放，调用IE的缺省行为
    return m_pOrginalDropTarget->Drop(pDataObj, grfKeyState, pt, pdwEffect);
  }
  
  //否则是拖放链接、图片、文件等，按常规的IDataObject处理方式
  ......
  return S_OK;
}
```

至此，我们就得到了一个完美的“超级拖放”的基本框架，它在扩展的同时保留了IE的默认行为：
+ 文字在页面内与输入框之间能够交互拖放。
  
+ 来自外部的文字与网页输入框之间也能交互拖放
  
+ 拖拽时能够自动滚动页面
 
其余的功能，如向不同的方向拖拽以完成不同的工作，左键右键拖放执行不同的功能，按住Alt保存文字等等，可根据需要自行实现，不再讨论。

### 8. 修正
今天和[Stanley Xu](https://www.csdndoc.com/article/634725)聊了几个钟头，受益匪浅。根据Stanley的提议，毋须再作是否往输入框拖拽文字的判断，因为我们需要的只是在`IE`的缺省行为没有鼠标拖拽效果的时候让它有拖拽效果，因此只需要简单地判断调用`IE`缺省行为后的`Effect`值是否为`0`即可，如下：

```c++
//判断是否是往输入框拖拽文字
m_bDragTextToInputBox = *pdwEffect != 0;
```

简单而直接，当然更重要的是：可用。

### 9. 参考资料

MSDN: [IHTMLEventObj Interface](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwi09c6kz9bpAhUhgK0KHT3HDbEQFjAAegQIAxAC&url=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fprevious-versions%2Fwindows%2Finternet-explorer%2Fie-developer%2Fplatform-apis%2Faa703876(v%253Dvs.85)&usg=AOvVaw1VDuSJ6ANki0PCPk_u0apw)

MSDN: [IHTMLdataTransfer Interface](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwj6zbyuz9bpAhUEH6wKHYwRBAgQFjAAegQIBBAC&url=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fprevious-versions%2Fwindows%2Finternet-explorer%2Fie-developer%2Fplatform-apis%2Faa752703(v%253Dvs.85)&usg=AOvVaw1wcL8JdoV9B_Y-Ito4xlSs)

[Internet Explorer 编程简述（十）响应来自HTML Element的事件通知——几个好用的类](https://eagleboost.com/2006/03/11/Internet-Explorer-%E7%BC%96%E7%A8%8B%E7%AE%80%E8%BF%B0-%E5%8D%81-%E5%93%8D%E5%BA%94%E6%9D%A5%E8%87%AAHTML-Element%E7%9A%84%E4%BA%8B%E4%BB%B6%E9%80%9A%E7%9F%A5-%E5%87%A0%E4%B8%AA%E5%A5%BD%E7%94%A8%E7%9A%84%E7%B1%BB/)》