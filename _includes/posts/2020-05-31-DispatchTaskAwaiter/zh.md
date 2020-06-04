## 1. 问题

前文《[*从BeginInvoke到async/await*](https://eagleboost.com/2020/02/21/DispatcherWaiter/)》
给出了[`DispatcherWaiter`](https://github.com/eagleboost/BeginInvokeToAsyncAwaitApp/blob/master/BeginInvokeToAsyncAwaitApp/DispatcherWaiter.cs)的概念和实现，近乎完美地解决了`await`某个`Task`之后优雅地切换到`GUI`线程的问题：

```c#
private async Task DoSomethingAsync()
{
  ////此处为任何线程
  await SomeTask.ConfigureAwait(false);

  ////此处为任何线程
  await waiter.WaitAsync();

  ////此处为waiter所包含的Dispatcher所在线程，即GUI线程
  DoSomethingOnGUIThread();
}
```

然而如果我们把追求完美的步伐再往前推进一步，上述代码还是略显累赘，能否省掉`await waiter.WaitAsync()`呢？

先简单回顾一下`Task`后面的`ConfigureAwait(bool continueOnCapturedContext)`的作用，它实际上提供了一个[改变`await`行为的钩子](https://devblogs.microsoft.com/dotnet/configureawait-faq/)

+ `ConfigureAwait(true)`——缺省值，可不写。使得编译器生成的状态机代码在异步代码执行完成后回到调用异步代码的线程。也就是说如果在主线程`await`了一个在后台线程执行的任务，任务结束后会自动回到主线程。

+ `ConfigureAwait(false)`——需明确指明。使得编译器生成的状态机代码在异步代码执行完成后不必切换线程。一般推荐在库代码中一路使用到底，因为在库函数内部虽然可能调用其它在后台执行的异步代码，但通常并不关心`GUI`线程，这样做可以省去线程切换带来的额外开销，也可以避免使用不当情况下的死锁。

`ConfigureAwait(bool)`一般来说下够用，然而在界面代码中难免会有调用多个`Task`后回到主线程的情形。比如下面的代码异步获得两个`Task`的结果并在主线程上使用。

如下的例子中，在没有`ConfigureAwait(true)`的缺省情况下，如果`DoSomethingAsync`在主线程（线程`1`）上被调用，那么线程`2`也会是主线程，同样线程`3`也会是主线程。最终`DoSomethingOnGUIThread`将会正确地在主线程上被调用。

```c#
private async Task DoSomethingAsyncV1()
{
  ////线程 1 => 主线程
  var result1 = await BackgroundTask1; ////等价于ConfigureAwait(true)
  ////线程 2 => 主线程
  var result2 = await BackgroundTask2; ////等价于ConfigureAwait(true)

  ////线程 3 => 主线程
  DoSomethingOnGUIThread(result1, result2);
}
```

但如果我们希望省去`BackgroundTask1`结束后的线程切换而加上`ConfigureAwait(false)`，那么代码会有问题——线程`2`和线程`3`都可能会是后台线程（取决于`BackgroundTask1`从开始执行到结束有多快）

通常的解决办法是重构一下，把获取`result1`和`result2`的过程合并到`GetResultAsync()`，在其内部一路`ConfigureAwait(false)`到底。下面的代码麻烦了一点（实际上是正确的做法），但可以正常工作。

```c#
private async Task DoSomethingAsyncV2()
{
  ////线程 1 => 主线程
  var tuple = await GetResultAsync(); ////等价于ConfigureAwait(true)

  ////线程 2 => 主线程。
  DoSomethingOnGUIThread(tuple.Item1, tuple.Item2);
}

private async Task<Tuple<int, int>> GetResultAsync()
{
  ////线程 1 => 主线程
  var result1 = await BackgroundTask1.ConfigureAwait(false);
  ////线程 2 => 后台线程
  var result2 = await BackgroundTask2.ConfigureAwait(false);

  ////线程 3 => 后台线程
  return Tuple.Create(result1, result2);
}
```

尽管`DoSomethingAsyncV2`是正确的做法，但开发者也许意识不到而仍旧使用了原来的调用方式并在`BackgroundTask1`后面加上`ConfigureAwait(false)`而最终报错。这时候如果使用[`DispatcherWaiter`](https://github.com/eagleboost/BeginInvokeToAsyncAwaitApp/blob/master/BeginInvokeToAsyncAwaitApp/DispatcherWaiter.cs)则上述重构可以省去并简化如下：

```c#
private async Task DoSomethingAsyncV3()
{
  ////线程 1 => 主线程
  var result1 = await BackgroundTask1.ConfigureAwait(false);
  ////线程 2 => 后台线程
  var result2 = await BackgroundTask2.ConfigureAwait(false);

  ////线程 3 => 后台线程
  await waiter.WaitAsync();

  ////线程 4 => 主线程
  DoSomethingOnGUIThread(result1, result2);
}
```

虽然使用[`DispatcherWaiter`](https://github.com/eagleboost/BeginInvokeToAsyncAwaitApp/blob/master/BeginInvokeToAsyncAwaitApp/DispatcherWaiter.cs)已经算是优雅，但是如果我们能把`await waiter.WaitAsync()`那一行省掉，代码会更加简洁——在回到主线程之前无论有多少个`ConfigureAwait(false)`，只要在最后一个后台线程调用后面加上`ConfigureAwait(DispatcherWaiter)`即可。


```c#
private async Task DoSomethingAsync()
{
  ////线程 1 => 主线程
  var result1 = await BackgroundTask1.ConfigureAwait(false);
  ////线程 2 => 后台线程
  var result2 = await BackgroundTask2.ConfigureAwait(waiter); ////注意此处参数为waiter

  ////线程 4 => 主线程
  DoSomethingOnGUIThread(result1, result2);
}
```

## 2. 分析

由于[`DispatcherWaiter`](https://github.com/eagleboost/BeginInvokeToAsyncAwaitApp/blob/master/BeginInvokeToAsyncAwaitApp/DispatcherWaiter.cs)的行为基于[`Dispatcher`](https://docs.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher?view=netcore-3.1)，为简化叙述我们仅讨论面向`Dispatcher`的实现，即`ConfigureAwait(Dispatcher)`，结论很容易可以推广到[`DispatcherWaiter`](https://github.com/eagleboost/BeginInvokeToAsyncAwaitApp/blob/master/BeginInvokeToAsyncAwaitApp/DispatcherWaiter.cs)。

一般说来实现`awaitable`的行为有两种方式：

1) 实现编译器可识别的签名，也就是`await`关键字后面的对象需要提供一个`GetAwaiter`方法，该方法返回的对象需要至少实现`INotifyCompletion`接口，一个布尔型`IsCompleted`属性和一个`GetResult`方法。如下参考来自[await anything](https://devblogs.microsoft.com/pfxteam/await-anything/)。

> The languages support awaiting any instance that exposes the right method (either instance method or extension method): `GetAwaiter`.  A GetAwaiter needs to implement the `INotifyCompletion` interface (and optionally the `ICriticalNotifyCompletion` interface) and return a type that itself exposes three members:

```c#
bool IsCompleted { get; }
void OnCompleted(Action continuation);
TResult GetResult(); // TResult can also be void
```

打开`ConfigureAwait(bool)`的[代码](https://referencesource.microsoft.com/#mscorlib/system/threading/Tasks/Task.cs,9ca6b2f012ce7587)可以看到一个`ConfiguredTaskAwaitable`的实例被创建，该实例包含的`GetAwaiter`方法返回一个`ConfiguredTaskAwaiter`的实例，后者实现了`ICriticalNotifyCompletion`和`INotifyCompletion`，编译器检测到代码满足规定的模式于是生成相应状态机代码来实现`awaitable`的行为。

```c#
public ConfiguredTaskAwaitable ConfigureAwait(bool continueOnCapturedContext)
{
  return new ConfiguredTaskAwaitable(this, continueOnCapturedContext);
}

public struct ConfiguredTaskAwaitable
{
  internal ConfiguredTaskAwaitable(Task task, bool continueOnCapturedContext)
  {
    this.m_configuredTaskAwaiter = new ConfiguredTaskAwaitable.ConfiguredTaskAwaiter(task, continueOnCapturedContext);
  }

  public ConfiguredTaskAwaitable.ConfiguredTaskAwaiter GetAwaiter()
  {
    return this.m_configuredTaskAwaiter;
  }

  public struct ConfiguredTaskAwaiter : ICriticalNotifyCompletion, INotifyCompletion
  {
  }
}
```

2) 使用`AsyncMethodBuilder`。`AsyncMethodBuilder`是高级版的用法，通常用于实现某种类似于`Task`的类型以提供自定义的`awaitable`行为，比如`ValueTask`，[*Async Task Types in C#*](https://github.com/dotnet/roslyn/blob/master/docs/features/task-types.md)中有更多叙述。

> A *task type* is a class or struct with an associated builder type identified with `System.Runtime.CompilerServices.AsyncMethodBuilderAttribute`. The task type may be non-generic, for async methods that do not return a value, or generic, for methods that return a value.

[*medium.com*](https://medium.com/criteo-labs/switching-back-to-the-ui-thread-in-wpf-uwp-in-modern-c-5dc1cc8efa5e)上有一篇使用`AsyncMethodBuilder`来实现本文类似功能的例子，但正如[Stephen Toub](https://github.com/stephentoub)在[*dotnet/csharplang*](https://github.com/dotnet/csharplang/issues/1407)的一个proposal中给出的如下概要，目前来说没有办法传递上下文信息给`AsyncMethodBuilder`，因为`Attribute`只能提供基于类型的静态信息。

```c#
public struct AsyncCoolTypeMethodBuilder
{
  public static AsyncCoolTypeMethodBuilder Create();
  ......
}

[AsyncMethodBuilder(typeof(AsyncCoolTypeMethodBuilder))]
public struct CoolType { ...... }

public async CoolType SomeMethodAsync() { ...... } // will implicitly use AsyncCoolTypeMethodBuilder.Create()
```

要求不高的情况下在自定义的`Task Type`中可直接访问`Application.Current.Dispatcher`，但对于包含多个`GUI`线程的应用来说`Dispatcher`就变成了必须传递给`AsyncMethodBuilder`的上下文。所以`AsyncMethodBuilder`虽然高级，但是对于解决本文要解决的问题实际上用处不大，因此在目前的`.net`版本下实际上只有`#1`华山一条路。

## 3. 实现

一切搞清楚之后代码实现起来非常简单。注意到非泛型`Task`和泛型`Task<T>`在行为上唯一的区别是有无返回值，我们把公用的代码提取到`DispatchTaskAwaiterHelper`中加以重用，非泛型的`DispatchTaskAwaiter`和泛型`DispatchTaskAwaiter<T>`分别调用无返回值的`GetResult`和有返回值的`GetResult<T>`即可。

值得注意的是虽然`ICriticalNotifyCompletion`继承自`INotifyCompletion`，但`.net`默认使用的`AsyncMethodBuilder`生成的针对`Task`的代码只会调用`UnsafeOnCompleted(Action)`，因此无需实现`OnCompleted(Action)`。

```c#
public readonly struct DispatchTaskAwaiter : ICriticalNotifyCompletion
{
  private readonly DispatchTaskAwaiterHelper _helper;

  public DispatchTaskAwaiter(Task task, Dispatcher dispatcher)
  {
    _helper = new DispatchTaskAwaiterHelper(task, dispatcher);
  }
  
  public DispatchTaskAwaiter GetAwaiter()
  {
    return this;
  }
  
  public bool IsCompleted
  {
    get { return _helper.IsCompleted; }
  }
    
  public void OnCompleted(Action continuation)
  {
    ////This is not called
    throw new NotImplementedException();
  }

  public void UnsafeOnCompleted(Action continuation)
  {
    _helper.UnsafeOnCompleted(continuation);
  }

  public void GetResult()
  {
    _helper.GetResult();
  }
}
```

`DispatchTaskAwaiterHelper`是真实行为的实现者。其原理简述如下：

1. 由于一个`Task`可以多次被`await`，因此`AsyncMethodBuilder`生成的代码首先会访问`IsCompleted`属性来确定是否满足调用`UnsafeOnCompleted`的条件。如果`IsCompleted==true`则会直接调用`GetResult`。所以在`IsCompleted`中我们简单地检查原始`Task`是否已完成以及是否当前处于`GUI`线程。

2. 如果`IsCompleted==false`则调用`UnsafeOnCompleted`并传入一个`continuation`委托作为`callback`。对于本例来说只需要在原始`Task`结束后，调用`continuation`之前通过`Dispatcher.BeginInvoke`切换到`GUI`线程。

3. 异常处理。需要确保原始`Task`中抛出的异常能被调用者的`try...catch`代码块捕捉到。容易想到在`UnsafeOnCompleted`中判断原始`Task`的状态并抛出异常，但因为异常处理的逻辑包含在`continuation`中，所以`UnsafeOnCompleted`中抛出的异常没法被状态机代码捕获，只能通过挂载`TaskScheduler`的`UnobservedTaskException`事件处理函数在`GC`发生的时候接收到。唯一可以处理异常的地方是`GetResult`方法，`VerifyException`方法抛出的异常会在`GUI`线程被捕捉到，这与`ConfigureAwait(true)`的行为一致。

```c#
public readonly struct DispatchTaskAwaiterHelper
{
  private readonly Task _task;
  private readonly Dispatcher _dispatcher;
  
  public DispatchTaskAwaiterHelper(Task task, Dispatcher dispatcher)
  {
    _task = task;
    _dispatcher = dispatcher;
  }
  
  public bool IsCompleted
  {
    get { return _task.IsCompleted && _dispatcher.CheckAccess(); }
  }
    
  public void UnsafeOnCompleted(Action continuation)
  {
    var tmp = this;
    _task.ContinueWith(t => tmp._dispatcher.BeginInvoke(continuation));
  }
  
  /// <summary>
  /// 供非泛型DispatchTaskAwaiter调用
  /// </summary>
  public void GetResult()
  {
    VerifyException();
  }
  
  /// <summary>
  /// 供泛型DispatchTaskAwaiter<T>调用
  /// </summary>
  public T GetResult<T>()
  {
    VerifyException();
    
    return ((Task<T>) _task).Result;
  }
  
  private void VerifyException()
  {
    if (!_task.IsCompleted)
    {
      throw new InvalidOperationException("Task is unexpectedly not completed");
    }
    
    if (_task.Exception != null)
    {
      throw _task.Exception;
    }
  }
}
```

## 4. 使用

实现了`DispatchTaskAwaiter`后我们只需要为非泛型`Task`和泛型`Task<T>`各添加一个扩展方法:


```c#
public static class TaskExt
{
  public static DispatchTaskAwaiter ConfigureAwait(this Task task, Dispatcher dispatcher)
  {
    return new DispatchTaskAwaiter(task, dispatcher);
  }
  
  public static DispatchTaskAwaiter<T> ConfigureAwait<T>(this Task<T> task, Dispatcher dispatcher)
  {
    return new DispatchTaskAwaiter<T>(task, dispatcher);
  }
}
```

然后这样使用，极为清爽：

```c#
private async Task DoSomethingAsync()
{
  ////线程 1 => 主线程
  var result1 = await BackgroundTask1.ConfigureAwait(false);
  
  ////此处_dispatcher为创建ViewModel时保存的GUI线程Dispatcher
  ////线程 2 => 后台线程
  var result2 = await BackgroundTask2.ConfigureAwait(_dispatcher);

  ////线程 4 => 主线程
  DoSomethingOnGUIThread(result1, result2);
}
```

上述代码扩展为`ConfigureAwait(DispatcherWaiter)`极其容易，不再赘述。


## 参考资料
+ [await anything](https://devblogs.microsoft.com/pfxteam/await-anything/)
+ [从BeginInvoke到async/await](https://eagleboost.com/2020/02/21/DispatcherWaiter/)
+ [Async Task Types in C#](https://github.com/dotnet/roslyn/blob/master/docs/features/task-types.md)
+ [Proposal: Allow [AsyncMethodBuilder(...)] on methods](https://github.com/dotnet/csharplang/issues/1407)