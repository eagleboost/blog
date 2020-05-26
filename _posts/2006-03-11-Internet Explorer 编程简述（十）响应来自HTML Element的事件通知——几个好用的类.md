---
layout:     post
title:      "Internet Explorer 编程简述（十）响应来自HTML Element的事件通知——几个好用的类"
subtitle:   "——谨以怀念研究Internet Explorer编程的青春岁月"
date:       2006-03-11 20:52:00
author:     "eagleboost"
header-img: "img/post-bg-sunset-ave.jpg"
catalog: true
tags:
    - Internet Explorer编程简述
    - Internet Explorer编程
    - HTML Element
    - Sink
    - IHTMLAnchorElement
    - CathyEagle
    - 存档
    - csdn
---

> 本文转载自[我2006年在csdn发布的博客](http://blog.csdn.net/CathyEagle/archive/2006/03/11/621961.aspx)(原链接已失效)

### 1. 概述

实现了对`Webbrowser`的`resue`之后我们便会发现有时候我们还需要处理浏览器中的元素（`HTML Element`）。这种处理包括主动和被动两个方面，像《[FAQ：如何访问Webbrowser的滚动条](https://eagleboost.com/2004/11/05/0-FAQ-%E5%A6%82%E4%BD%95%E8%AE%BF%E9%97%AEWebBrowser%E7%9A%84%E6%BB%9A%E5%8A%A8%E6%9D%A1/)》、《[FAQ：操纵下拉列表](https://eagleboost.com/2004/10/15/FAQ-%E6%93%8D%E7%BA%B5%E4%B8%8B%E6%8B%89%E5%88%97%E8%A1%A8/)》、《[FAQ：两种方法访问多层嵌套的frame](https://eagleboost.com/2004/09/30/FAQ-%E4%B8%A4%E7%A7%8D%E6%96%B9%E6%B3%95%E8%AE%BF%E9%97%AE%E5%A4%9A%E5%B1%82%E5%B5%8C%E5%A5%97%E7%9A%84frame/)》等 文章所演示的就是主动的处理。通常我们从`Webbrowser`获得一个Web文档接口（`IHTMLDocumentx`），从它出发便可访问到浏览器所包含 的一切`HTML`元素。而被动的处理则是在`COM`技术中称为`Sink`的技术，我更喜欢的说法是事件通知。当文档的下载进度发生变化时，我们可以获得`ProgressChange`通知，当`Webbrowser`下载完`HTML`文档时，我们可以获得`DocumentComplete`的通知，而当链接被点 击，或图片被拖动时，我们如何获得通知呢？本文希望能够给出部分的答案。

### 2. HtmlObj Template

如何`Sink`一个`HTML Element`并不是本文的重点，其理论我不是太了解，也懒得去搞透彻，所以使用现成的库来实现。[CodeProject](http://www.codeproject.com)上的一篇文章《[HtmlObj Template](https://www.codeproject.com/Articles/4875/HtmlObj-Template)》给出的一个模板类`CHtmlObj`就非常好用。下面的例子是针对`Html Anchor Element`的一个实例化。

```c++
#include "HtmlObj.h"
 
class CHtmlAnchorElement : public CHtmlObj<IHTMLAnchorElement, &DIID_HTMLAnchorEvents> 
{
public:
  CHtmlAnchorElement(CHtmlDocument2* pParentDoc2);
  virtual ~CHtmlAnchorElement();
  virtual HRESULT OnInvoke(DISPID dispidMember, REFIID riid, LCID lcid, WORD wFlags,DISPPARAMS * pdispparams, VARIANT * pvarResult, EXCEPINFO * pexcepinfo, UINT * puArgErr);
};

HRESULT CHtmlAnchorElement::OnInvoke(DISPID dispidMember, REFIID riid, LCID lcid, WORD wFlags,DISPPARAMS * pdispparams, VARIANT * pvarResult, EXCEPINFO * pexcepinfo, UINT * puArgErr)
{
  HRESULT hr = E_NOTIMPL;
  switch(dispidMember)
  {
  case DISPID_HTMLELEMENTEVENTS_ONMOUSEOVER :
  { //当鼠标经过链接时，我们在这里获得通知
    hr = S_OK;
    // TODO: add code to handle on mouse over events
    break;
  }
  case DISPID_HTMLELEMENTEVENTS_ONMOUSEOUT :
  { //当鼠标从链接上移开时，我们在这里获得通知，其它的Dispatch ID可根据需要添加
    hr = S_OK;
    // TODO: add code to handle on mouse out events
    break;
  }
  default:
    break;
  }
  
  return hr;
}
```

当我们得到某个链接的`HTML`接口指针，便可调用`CHtmlAnchorElement`继承自`CHtmlObj`的`SetSite(IUnknown *pUnkSite)`成员函数传入该接口指针。在`CHtmlObj`类内部用一个智能指针`m_spHtmlObj`来保存相应的`HTML Element`接口指针，所以当上面的`ONMOUSEHOVER`和`ONMOUSEOUT`两个事件通知到达时，从`m_spHtmlObj`就可以访问`IHTMLAnchorElement`的所有成员，如从`href`获得链接的`Url`等，此处不再赘述。

### 3. CHtmlElements类

有 了`CHtmlObj`之后我们又会发现实践中常常会需要多个相同类型的`CHtmlObj`。比如包含`Frame`的网页中每个`Frame`的`HTML Document`都需要一个`CHtmlObj`来Sink其事件。所以我们还需要有效地管理这些相同类型的`CHtmlObj`。下面是我写的一个简单的模板类`CHtmlElements`，它通过`CMap`来管理多个`CHtmlObj`对象。

```c++
template<class THtmlElement> class CHtmlElements
{
  typedef CMap<LPDISPATCH, LPDISPATCH, THtmlElement*, THtmlElement*> CMapDispToHtmlElement;
CMapDispToHtmlElement m_htmlElements;
  BOOL IsSiteConnected( LPDISPATCH pDisp )
  {
    THtmlElement *pElement;
    return m_htmlElements.Lookup( pDisp, pElement );
  }
public :
  CHtmlElements( void )
  {
  }
  ~CHtmlElements( void )
  {
  }
public :
  void SetSite( LPDISPATCH pDisp )
  {
    if ( IsSiteConnected( pDisp ) ) //检查以避免多余的Sink
    {
      return ;
    }
    THtmlElement *pElement = new THtmlElement; //通过模板类型创建相应的类的实例进行连接
    pElement->SetSite( pDisp );
    m_htmlElements.SetAt( pDisp, pElement );
  }
 
  //在合适的地方调用Clear释放所管理的内存

  void Clear(void)
  {
    POSITION pos = m_htmlElements.GetStartPosition();

    THtmlElement *pElement = NULL;
    LPDISPATCH pDisp = NULL;
    while (pos != NULL)
    {
      m_htmlElements.GetNextAssoc( pos, pDisp, pElement );
      m_htmlElements.RemoveKey( pDisp );
      delete pElement;
    }
  }
};
```

假设我们有一个象`CHtmlAnchorElement`那样派生自`CHtmlObj`的类`CHtmlDocument2`，使用`CHtmlElements`时这样声明：

```c++
typedef CHtmlElements<CHtmlDocument2> CHtmlDocuments;
typedef CHtmlElements<CHtmlAnchorElement> CHtmlAnchors;
 
class CMyView : public CHtmlView
{
  private :
  CHtmlDocuments m_htmlDocs;
  CHtmlAnchors m_htmlAnchors;
}
```

在`DocumentComplete`时就可以这样连接到浏览器的文档对象：

```c++
void CMyView ::OnDocumentComplete(LPDISPATCH pDisp, LPCTSTR lpszURL)
{
  m_htmlDocs.SetSite(pDisp);
}
```

如果想一次性连接上文档中所有的Anchor Element，可以通过`IHTMLDocument2::get_anchors`获得包含所有`IHTMLAnchorElement`接口指针的`IHTMLElementCollection`，再遍历其中的每个元素，分别调用`m_htmlAnchors.SetSite`即可。当然，一次性的`Sink`全部链接可能并不是个好注意，我更愿意在`CHtmlDocument2`中响应事件再通过其它手段来访问当前位置的`HTML Element`。

### 4. 结论

响应`HTML Element`的事件通知对于浏览器编程来说是一个非常强大的手段，它可以更深入细化地控制浏览器中的文档及其`HTML`元素，实现更为高级的功能，比如所谓的“超级拖放”（许多多窗口浏览器都提供了该功能，但实际上没有哪个浏览器完美地实现了对`URL`、文字及图片的拖放）。

### 5. 参考资料

Codeproject:《[HtmlObj Template](https://www.codeproject.com/Articles/4875/HtmlObj-Template)》