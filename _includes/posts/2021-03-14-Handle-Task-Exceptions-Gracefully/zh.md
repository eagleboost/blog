## 1. 问题

异常处理是在`.net`平台上使用`Task`时最常遇到的问题之一。异常处理本身是个不小的话题，本文不打算把焦点放在如何在一连串的`Task Chain`中正确`Propagate`异常之类的话题，只讨论如何优雅地处理一个异步调用（可能包含一连串的`Task`）中可能发生的异常（假设所有异常都会被正确地送回来）。

原生的做法是调用`ContinueWith`来创建一个`Continuation`，当任务结束后在其中判断是否有`Exception`发生并处理之——比如显示一个`MessageBox`告诉用户有错误发生。

```c#
CallSomethingAsync().ContinueWith(t =>
{
  if (t.Exception != null)
  {
    ////在这里处理异常
  }
});
```

这样做在小型应用程序或某个`POC`中完全正常，但是对于颇具规模的中大型应用程序中却会导致维护上的麻烦：

1. 异常发生的时候如果要显示`MessageBox`则需要引入额外的依赖，比如某个`MessageBox service`。
2. 异步调用数量很多的时候在每个地方都需要重复同样的代码以及引入额外的依赖。

我们需要一种简洁而优雅的方式来处理`Task`的异常。


## 2. 分析

一种偷懒的方式是使用`TaskScheduler.UnobservedTaskException`事件——该事件用于接收任何在`Task`中发生但未被观察到的异常，即`Task.Exception`属性从未被外部代码显示访问过。这种方法可行但属于不负责任的做法，一方面把原本该由开发者捕获并做出相应处理的异常丢给`CLR`，另一方面该事件只有等`GC`发生的时候才会触发，实时性差，在生产环境中容易引起误解。本文例子中的`Unobserved`按钮演示了这一行为。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/HandleTaskExceptionsGracefully.png)

针对我们的需求：用尽量简洁以及依赖最少的代码来把异常显示出来，能想到最简单的方式莫过于添加一个扩展方法，并让该方法把异常传播到合适的地方以便显示出来。

```c#
CallSomethingAsync().WhenFaultedAsync("xxx调用失败。");
```

而这个合适的地方，则是`Dispatcher.DispatcherUnhandledException`事件的订阅者。跟`TaskScheduler.UnobservedTaskException`事件不同的是，前者在`Dispatcher`空闲的时候即会触发。

## 3. 实现

废话少说，先上代码。`WhenFaultedAsync`接受两个参数，一是适当的`Dispatcher`，二是一个用于区分异常来源的字符串。考虑到绝大多数应用程序实际上只有一个`Dispatcher`，另一个重载则默认使用`Application.Dispatcher`。有多个`Dispatcher`的情形可以选择性调用第一个方法把`Dispatcher`显示传递过去。

代码的重点在于异常发生时切换到`Dispatcher`所在线程抛出一个包含上下文信息的`AggregateException`，以便被`Dispatcher.DispatcherUnhandledException`捕捉到。

```c#
const TaskContinuationOptions FaultedFlag = TaskContinuationOptions.OnlyOnFaulted | TaskContinuationOptions.ExecuteSynchronously;

public static Task WhenFaultedAsync(this Task task, Dispatcher dispatcher, string context)
{
  task.ContinueWith(t =>
  {
    if (t.Exception != null)
    {
      dispatcher.BeginInvoke(() => throw new AggregateException(context, t.Exception.InnerExceptions));
    }
  }, FaultedFlag);
  
  return task;
}

public static Task WhenFaultedAsync(this Task task, string context)
{
  return task.WhenFaultedAsync(Application.Current.Dispatcher, context);
}
```

用于显示异常的代码可以是个简单的事件处理方法。当然这里有所简化，对于多个异常连续发生的情形有两个可用的策略，具体实现不再赘述：

+ 其一，显示第一个异常为`MessageBox`，在`MessageBox`被用户关掉之前，后续的异常只写入日志。避免出现用户关掉一个`MessageBox`又来一个的尴尬场景。
+ 其二，把异常全部发送到某个一直存在于内存中会自动弹出的非模态窗口，显示为列表或表格，一来所有异常一目了然，二来避免打断用户的当前操作。

```c#
Application.Current.DispatcherUnhandledException += (_, e) =>
{
  ShowException(e.Exception.Message, e.Exception);
  e.Handled = true;
};
```

在某些情况下下我们甚至可以省去`WhenFaultedAsync`这样的扩展方法。比如一个使用了`Unity Container`的系统可以通过`Interception`截获某个类（通常是实现了某些接口的`Service`）的所有异步调用，并动态生成代码来隐式调用`WhenFaultedAsync`，这对于有清晰层级划分的系统来说可以一定程度简化应用层代码的维护。

完成实现请移步[*github*](https://github.com/eagleboost/HandleTaskException)
