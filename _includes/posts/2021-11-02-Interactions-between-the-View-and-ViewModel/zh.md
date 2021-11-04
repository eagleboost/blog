## 1. 问题

显示窗口是桌面应用程序中永恒的主题之一。最常见的是模态窗口，比如`MessageBox`，或者更复杂一些如`用户登录`这类需要输入信息的窗口。非模态窗口也有，但不多见，小型应用程序通常就没有使用非模态窗口的场景。

上面的描述是从使用的角度出发，而对于开发人员来说“`如何显示窗口`”则是一个看似简单但不容易实现好的问题，因为需要考虑的不仅仅是“`创建一个窗口并显示出来`”这么简单：

+ **视图与数据分离**——这在`M-V-VM`中尤为重要，因为视图（窗口）的具体实现要能够独立地更新和优化。
+ **易于测试**——直接创建并打开窗口的代码测试起来相当麻烦，而实现了视图与数据分离的代码则天然易于测试。
+ **异步交互**——复杂的交互逻辑常常伴随着异步操作，比如在异步登录过程中弹出对话框等待用户输入信息后再继续。
+ **统一的处理方式**——模态窗口需要等待子窗口关闭后焦点才会回到父窗口，从直觉上说有一个“`交互`”的过程。但实际上非模态窗口也应该能用同样的方式处理。举例来说，点击主窗口上的按钮显示一个非模态窗口（或者把数据传递给一个已经存在的非模态窗口），点击非模态窗口上的某个按钮表示操作完成，之后可以关闭该窗口或者清空所有数据等待下一次操作。


常见的实现方式大都只考虑了前两种情况，而且有不小的缺陷。

 
## 2. Dialog Service

这是最容易想到的方法，毕竟通过接口抽象把实现隐藏起来是现代程序开发者的基本素养之一。以`MessageBox`为例，人们会定义这样的接口:

```c#
public interface IMessageBox
{
  bool Show(string header, string content, MessageBoxType type, MessageBoxIcon icon);
}
```

`IMessageBox`接口的实现可以创建自定义的窗口或调用`WPF`包装自`Win32 API`的`MessageBox`类的方法，简单直接，实现了视图与数据分离和易于测试，但问题也显而易见，比如：

+ 某些需要引起用户注意的重要消息我们希望在消息框加上闪烁的效果。
+ 某些情况下用户希望整个消息框的背景使用不同颜色。
+ 再复杂一点，比如对话框需要支持“`是`”，“`否`”，“`取消`”按钮，甚至是显示一个“`下次不再提示`”的勾选框。
+ 需要其它类型的窗口，比如一个`InputBox`。
+ 在同一个`ViewModel`中需要显示多个不同类型对话框的时候难以区分这些调用和测试相应代码。

满足这些需求会导致要么接口参数增多（可以用对象代替参数列表的方式解决），要么接口方法增多，要么接口增多，但总体说来灵活性太差，与异步交互也先天不兼容，更无法处理非模态窗口。


## 3. InteractionRequest

`M-V-VM`领域的老牌劲旅[Prism](https://prismlibrary.com)——也是我一直使用的框架——对于这个问题给出的答案是基于`System.Windows.Interactivity`的[InteractionRequest](https://prismlibrary.com/docs/wpf/legacy/Advanced-MVVM.html)。原理很简单，在`ViewModel`中创建一个`IInteractionRequest`对象，在`View`中则通过一个`InteractionRequestTrigger`订阅该对象的`Raised`事件，当`Raised`事件被触发时会根据`XAML`里配置好的窗口类型创建一个窗口并显示出来，最后窗口关闭的时候调用随着事件参数传递过来的`callback`把结果传回给`ViewModel`。


```c#
public interface IInteractionRequest
{
  event EventHandler<InteractionRequestedEventArgs> Raised;
}
```

乍一看很美，我在项目中就曾经大量使用了这项技术。[InteractionRequest](https://prismlibrary.com/docs/wpf/legacy/Advanced-MVVM.html)创造性地使用事件解决了数据传递的问题，但也因为事件的存在，使得跨线程的异步交互难于实现，而且事件必须有订阅者整个流程才能工作，反而增加了耦合度。比如在后台线程显示一个对话框的情形（尽管并不推荐这么做），需要附加一个`View`来订阅事件是不可想象的。

把异步交互撇开不谈，这项技术还有一个常常被忽视的问题——容易导致内存泄露。由于`InteractionRequestTrigger`通过数据绑定发现`IInteractionRequest`对象来订阅`Raised`事件，所以在一个组合式的`XAML`中`Raised`事件很容易在不知不觉中被订阅多次。其症状则是同样的窗口关闭了又显示出来，因为事件有多个订阅者。

这种问题一方面排查麻烦，另一方面修补起来容易导致代码变得臃肿。比如一个封装这项技术的可重用自定义控件就需要添加一些额外的属性用来控制是否允许`InteractionRequestTrigger`工作，这样才能在多个同样的控件出现在同一个`View`的场景下只让其中一个工作来避免泄露。

[Prism](https://prismlibrary.com)开发组也意识到了`InteractionRequest`的一些[问题](https://github.com/PrismLibrary/Prism/issues/864)，其目前的主要维护者[Brian Lagunas](https://github.com/brianlagunas)则直接表示不喜欢该方法，并在`2019`年提出了一个[新的方法](https://github.com/PrismLibrary/Prism/issues/1666)并纳入[官方文档](https://prismlibrary.com/docs/wpf/dialog-service.html)，但这个“`新方法`”乏善可陈，实际上回到了`Dialog Service`的老路，唯一改进是把参数列表换成了一个`IDialogParameters`接口，同样不支持异步交互和非模态窗口支持。


## 3. AsyncInteraction

回归到问题本身，我们会发现“`显示对话框`”要做的事总结下来就是“`显示一个窗口，等待用户输入后返回并获取结果`”，跟“`启动一个任务，等待任务结束后返回并获取结果`”一模一样。概念澄清之后解决方案也就顺理成章了。

首先我们把“`显示对话框`”这件事抽象成一个`IAsyncInteraction<T>`接口，`StartAsync`方法接受一个参数并返回一个`Task`，`AsyncInteractionArgs`可用来传递任意数目的参数以增加灵活性。

```c#
public interface IAsyncInteraction<T> where T : class
{
  Task<AsyncInteractionResult<T>> StartAsync(AsyncInteractionArgs args);
}

public sealed class AsyncInteractionArgs : ReadOnlyCollection<object>
{
  public AsyncInteractionArgs(params object[] args) : base(args)
  {
  }
}

public class AsyncInteractionResult<T> where T : class
{
  public readonly bool IsConfirmed;
  
  public readonly T Result;

  public AsyncInteractionResult(bool isConfirmed, T result)
  {
    IsConfirmed = isConfirmed;
    Result = result ?? throw new ArgumentNullException(nameof(result));
  }
}
```

容易看出，类型参数`<T>`的驱动使得基于接口的依赖注入成为可能，比如一个需要与消息框和登录框交互的`ViewModel`可以包含如下依赖注入属性（以`UnityContainer`为例），这样一来测试更加容易，也更具有针对性。

```c#
[Depencency]
public IAsyncInteraction<MessageBoxData> MessageBox { get; set; }
    
[Depencency]
public IAsyncInteraction<ILoginViewModel> LoginBox { get; set; }
```

使用则是与`Task`一脉相承的异步操作：把输入参数（通常也是返回值）通过`AsyncInteractionArgs`传递过去，由接口的具体实现并切换到指定的`GUI`线程（主线程或其它`GUI`线程），根据输入创建（不同类型的窗口）和渲染窗口（根据数据类型选择数据模版等），设置父窗口并显示，当窗口关闭后（或者非模态窗口完成处理后）把结果从`Task`传回给调用者。

下面的代码以显示`MessageBox`为例：

```c#
private async Task<AsyncInteractionResult<MessageBoxData>> ShowMessageAsync()
{
  var data = CreateMessageBoxData();
  var args = new AsyncInteractionArgs(data);
  return MessageBox.StartAsync(args);
}
```

如此一来本文开头说到的几个问题就得到了完美解决：

+ **视图与数据分离**——`ViewModel`只关心接口，视图的创建完全隐藏在具体实现中，视图的样式则由数据驱动，开发者有最大的自由度。
+ **易于测试**——接口容易模拟，基于`Task`的代码测试非常方便，相比之下基于事件的测试则麻烦一些。
+ **异步交互**——`Task`与异步交互浑然天成，即便是在某个`Service`的后台线程显示一个对话框，等待用户输入后继续的场景也能轻松处理。
+ **统一的处理方式**——由于调用方只关心`Task`的状态，模态还是非模态窗口也就没有了本质区别，能用统一的接口来完成。
+ **对比`Dialog Service`**——一方面泛型参数更为灵活，无需实现类似`IDialogParameters`的接口，另一方面根据泛型参数能够注入不同实现，依赖更为清晰。
+ **对比`InteractionRequest`**——一方面避免了事件必须有订阅者才能工作的限制，另一方面杜绝了因为事件而导致的多次订阅（内存泄露）的问题产生。


## 4. 结语

实现细节以及实例请异步[github](https://github.com/eagleboost/AsyncInteraction)。为方便讨论，文中的代码及实例全部基于`.net 5`，但`IAsyncInteraction<T>`接口所表达的原理本身是没有局限的。不论是`WPF`还是`WinForm`，`.net Core`还是`.net Framework`，甚至是其它编程语言，只要有类似于`Task`的支持都可以完成相应的实现。