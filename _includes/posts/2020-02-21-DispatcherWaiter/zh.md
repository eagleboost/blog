## 问题

`Dispatcher.BeginInvoke`是WPF中最常用的方法之一，其使用场合一般说来有两种：
1. 跨线程调度——在某个后台线程完成了计算，需要通知程序界面更新，比如把从网络服务取回的天气信息显示在一个文本框里。
2. 延迟执行——为了让主程序界面对用户输入保持响应，有时候我们希望一段代码在主线程里以较低的优先级执行。

`BeginInvoke`有多个重载，其中最常用的两个签名如下：
```c#
public DispatcherOperation BeginInvoke(Action action)
public DispatcherOperation BeginInvoke(Action action, DispatcherPriority priority)
```

用起来很简单，然而却存在几个难以回避的问题：
1. 不便于单元测试——为某个显式使用了`Dispatcher`的`ViewModel`写单元测试会非常麻烦。一来`Dispatcher`需要`STA`线程，但测试常常不需要验证延迟行为，而且创建`STA`线程会对测试环境造成不小的压力。二来就算有了`Dispatcher`，其延迟执行的行为测试起来也令人头痛。
   
2. 添加到执行队列的操作无法取消——道理上来说`BeginInvoke`一个`Action`跟创建一个`Task`类似，但却不能像`Task`一样可以通过`CancellationToken`来取消。一个被`BeginInvoke`了`n`次的`Action`也必然会执行`n`次，这有时不是我们想要的。
   
3. 与异步风格的代码格格不入。

## 分析

先说第三问题。从上面的签名可以看到`BeginInvoke`实际上返回了一个`DispatcherOperation`，而后者是可以被`await`的，所以下面的代码合法：

```c#
await Dispatcher.BeginInvoke(() => DoSomething() );
```

但是在需要返回值的情况下`BeginInvoke`就不适用了。好在`dotnet 4.5`中增加了新的`InvokeAsync`方法，为异步返回值和用于取消的`CancellationToken`提供了支持，于是第二个问题也顺带被解决了。

```c#
public DispatcherOperation<TResult> InvokeAsync<TResult>(
  Func<TResult> callback,
  DispatcherPriority priority,
  CancellationToken ct)
```

回到第一个问题，关于单元测试。为了便于测试和节省资源，应该避免直接使用`Dispatcher`。一种做法是抽象出一个包含`BeginInvoke`方法的`IDispatcher`接口，这样就可以`mock`该接口来测试。但问题在于对`callback`的`mock`需要额外的`setup`，不同的`mock`框架做法不尽相同，但总的来说想要在测试中断言`BeginInvoke`的方法被调用所需要的准备工作并不容易。

既然`BeginInvoke`这种`callback`的做法对测试并不友好，我们就需要换一个角度来思考。当调用`BeginInvoke`的时候开发者想要的是＂把这个`Action`添加到`ApplicationIdle`任务队列，等`Dispatcher`开始处理该队列的时候执行它＂，它等价于＂`Dispatcher`开始处理`ApplicationIdle`队列的时候通知我，我将执行这个`Action`＂。

于是问题就立刻变简单了，假设有这样一个接口。

```c#
public interface IDispatcherWaiter
{
  Task<TaskStatus> WaitAsync(DispatcherPriority priority, CancellationToken ct)
}
```

调用就变成这样：
```c#
private async Task DoSomethingAsync()
{
  ////waiter是一个IDispatcherWaiter的实例
  await waiter.WaitAsync(DispatcherPriority.ApplicationIdle);
  DoSomething();
}

private async Task DoSomethingAsync(CancellationToken ct)
{
  ////waiter是一个IDispatcherWaiter的实例
  var status = await waiter.WaitAsync(DispatcherPriority.ApplicationIdle, ct);
  if (status == TaskStatus.RanToCompletion)
  {
    DoSomething();
  }  
}
```

在单元测试中只需要`mock`一个`IDispatcherWaiter`即可，或者直接提供一个可以重用的`TestDispatcherWaiter`让`WaitAsync`直接返回`TaskStatus.RanToCompletion`。

实现似乎很容易——`await InvokeAsync`并且传入`CancellationToken`。

```c#
public class DispatcherWaiter :IDispatcherWaiter
{
  public async Task<TaskStatus> WaitAsync(DispatcherPriority priority, CancellationToken ct)
  {
    return await _dispatcher.InvokeAsync(() => GetResult(ct), priority, ct);
  }

  private static TaskStatus GetResult(CancellationToken ct)
  {
    return ct.IsCancellationRequested ? TaskStatus.Canceled : TaskStatus.RanToCompletion;
  }
}
```

## 再分析

上面的实现直接`await DispatcherOperation`看似解决了问题，但有一个致命的缺陷。

我们知道，`await`一个`Task`的时候，可以通过`ConfigureAwait(true|false)`来控制在`Task`执行结束后是否回到`await`之前捕捉的`SynchronizationContext`，默认行为是`true`，即回到之前的`SynchronizationContext`。

`await DispatcherOperation`的行为等价于`ConfigureAwait(true)`，所以上面两个`DoSomethingAsync()`方法中`DoSomething()`最终会在主线程上被调用的前提是`DoSomethingAsync()`在主线程上被调用。

假如`DoSomethingAsync()`在某个后台线程X被调用，如下所示，那么当`WaitAsync`结束时`DoSomething()`会在线程X上执行，这可能会导致与`BeginInvoke(()=> DoSomething() )`不同的行为，因为`BeginInvoke`的`callback`是在`Dispatcher`所在的线程上执行的。

```c#
private Task DoSomethingFirstAsync()
{
  return Task.Run(()=>
  {
    ////此处为后台线程X
    DoSomethingAsync();
  });
}
```

与`BeginInvoke(()=> DoSomething() )`等价的行为应该是这样：

```c#
private Task DoSomethingAsync()
{
  ////此处为任何线程
  await waiter.WaitAsync(DispatcherPriority.ApplicationIdle);
  ////此处为waiter所包含的Dispatcher所在线程
  DoSomething();
}
```

因此`await Dispatcher.InvokeAsync`实际上不能用。


## 解决方案

好在`dotnet`的设计者允许我们实现自定义的`awaitable`，微软工程师[`Stephen Toub`](https://devblogs.microsoft.com/pfxteam/author/toub/)有一篇[`await anything`](https://devblogs.microsoft.com/pfxteam/await-anything/)的博客介绍了实现自定义`awaitable`所需的一切知识。

在下面的`DispatcherWaiter`实现中，当`await waiter.WaitAsync`执行结束后并不会切换到调用之前的线程，从而保证了接下来的代码在期望的线程上执行。

以下为完整实现，代码很简单，注释略去。

```c#
////首先定义一个IAwaitable的通用接口，实现该接口的对象即可被await
public interface IAwaitable<out T, out TResult> : INotifyCompletion where T : class
{
  T GetAwaiter();

  TResult GetResult();

  bool IsCompleted { get; }
}

////定义IDispatcherWaiter
public interface IDispatcherWaiter : IAwaitable<IDispatcherWaiter, TaskStatus>
{
  /// <summary>
  /// 返回Dispatcher.CheckAccess
  /// </summary>
  /// <returns></returns>
  bool CheckAccess();
  
  /// <summary>
  /// 调用Dispatcher.VerifyAccess
  /// </summary>
  /// <returns></returns>
  void VerifyAccess();

  /// <summary>
  /// 假如调用时处于Dispatcher同一线程则直接返回，否则会调度到Dispatcher线程再返回 
  /// </summary>
  /// <returns></returns>
  IDispatcherWaiter CheckedWaitAsync();

  /// <summary>
  /// 以指定优先级调度到Dispatcher线程并允许取消
  /// </summary>
  /// <param name="priority"></param>
  /// <param name="ct"></param>
  /// <returns></returns>  
  IDispatcherWaiter WaitAsync(DispatcherPriority priority = DispatcherPriority.Normal, CancellationToken ct = default(CancellationToken));
}

////具体实现
public class DispatcherWaiter : IDispatcherWaiter
{
  private readonly Dispatcher _dispatcher;
  private DispatcherPriority _priority;
  private CancellationToken _ct;
  private bool _isCompleted;

  public DispatcherWaiter(Dispatcher d)
  {
    _dispatcher = d;
  }

  public IDispatcherWaiter GetAwaiter()
  {
    return this;
  }
  
  public TaskStatus GetResult()
  {
    return _ct.IsCancellationRequested ? TaskStatus.Canceled : TaskStatus.RanToCompletion;
  }

  public bool IsCompleted
  {
    get { return _isCompleted; }
  }

  public void OnCompleted(Action continuation)
  {
    _dispatcher.InvokeAsync(continuation, _priority, _ct);
  }

  public bool CheckAccess()
  {
    return _dispatcher.CheckAccess();
  }

  public void VerifyAccess()
  {
    _dispatcher.VerifyAccess();
  }

  public IDispatcherWaiter CheckedWaitAsync()
  {
    _isCompleted = _dispatcher == Dispatcher.CurrentDispatcher;
    return this;
  }

  public IDispatcherWaiter WaitAsync(DispatcherPriority priority = DispatcherPriority.Normal, CancellationToken ct = default(CancellationToken))
  {
    if(priority== DispatcherPriority.Send)
    {
      throw new InvalidOperationException("Send priority is not allowed");
    }
    
    _priority = priority;
    _ct = ct;
    return this;
  }
}
```

## 结论

至此，文章开头提出的几个问题就都得到了完美解决，在任何需要调用`Dispatcher.BeginInvoke`的地方都可以用`IDispatcherWaiter`来替换，代码更为流畅，也更易于编写单元测试。

总结一下使用方式：

```c#
////Async/Await风格
private async Task DoSomethingAsync(CancellationToken ct)
{
  var status = await waiter.WaitAsync(DispatcherPriority.ApplicationIdle, ct);
  if (status == TaskStatus.RanToCompletion)
  {
    DoSomething();
  }
}

////类似于BeginInvoke(()=> DoSomething() )的传统风格
private DoSomething(CancellationToken ct)
{
  waiter.WaitAsync(DispatcherPriority.ApplicationIdle, ct).OnCompleted(() =>
  {
    if (!ct.IsCancellationRequested)
    {
      DoSomething();
    }
  });
}
```

我们还可以写一系列扩展方法来帮助简化代码进一步提高可读性：

```c#
public static class DispatcherWaiterExt
{
  public static IDispatcherWaiter WaitForAppIdleAsync(this IDispatcherWaiter waiter, CancellationToken ct = default(CancellationToken))
  {
    return waiter.WaitAsync(DispatcherPriority.ApplicationIdle, ct);
  }
}

private async Task DoSomethingAsync(CancellationToken ct)
{
  if (await waiter.WaitForAppIdleAsync(ct) == TaskStatus.RanToCompletion)
  {
    DoSomething();
  }
}
```

## 参考资料
+ [await anything](https://devblogs.microsoft.com/pfxteam/await-anything/)