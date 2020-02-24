## The problem

`Dispatcher.BeginInvoke` is one of the useful functions that is widely used in WPF applications. It's often used in two scenarios:

1. Cross thread scheduling - When some calculation is completed on a background thread, we need to notify GUI to refresh and pickup the new data, for example, refresh display of weather information returned from a network weather service provider.
2. Delayed execution - To keep main GUI responsive to user inputs, sometimes we dispatch calls on main GUI thread with lower priorities.

There're quite a few overloads for the `BeginInvoke` metho, below are the two being used most often:

```c#
public DispatcherOperation BeginInvoke(Action action)
public DispatcherOperation BeginInvoke(Action action, DispatcherPriority priority)
```

It's simple and straightforward to use, only with some unavoidable problems:

1. Not unit test friendly - To write unit tests for a `ViewModel` that explicitly uses `Dispatcher` is troublesome. One one hand an `STA` thread is needed in order for `Dispatcher` to be created, but most of the time there's no need to verify the _delayed execution_ behavior in the the unit tests, and the creation of `STA` threads would put unnecessary pressure to the test environment. One the other hand, even if we already have a `Dispatcher`, to test the _delayed execution_ behavior still needs some work.
   
2. Dispatched calls cannot be canceled - To `BeginInvoke` an `Action` is similar to creating a `Task`, however, the `Action` cannot be canceled like `Task` by passing an `CancellationToken`. If an `Action` dispatched by calling `BeginInvoked` n times, it would eventually be executed n times, which may not be what we want in some cases.
   
3. Coding is not as smooth as async/await style.

## Analysis

First let's look at problem `#3`. The signature of the `BeginInvoke` method shows the return value is a `DispatcherOperation`, and `DispatcherOperation` is `awaitable`, so the code piece below is a valid usage: 

```c#
await Dispatcher.BeginInvoke(() => DoSomething());
```

But `BeginInvoke` doesn't work for cases that return value is needed. The good news is that a new `InvokeAsync` method has been added to the `Dispatcher` in `dotnet 4.5` to  support return value and `CancellationToken`, so `#2` is not a problem anymore.

```c#
public DispatcherOperation<TResult> InvokeAsync<TResult>(
  Func<TResult> callback,
  DispatcherPriority priority,
  CancellationToken ct)
```

Now we can go back to problem `#1` about unit tests. In general, explicit dependency of `Dispatcher` should be avoided in order to simplify unit tests. To do that one approach is to abstract an `IDispatcher` interface with a `BeginInvoke` method, then `mock` the interface for tests. However, this approach requires extra `setup` work for the `mocked` object for the `callback`. Different `mock` framework has different ways of doing so, but after all it's not straightforward to verify invocation of calls being `BeginInvoked`. 

Since the `callback` style of `BeginInvoke` is not unit test friendly, we have to find other way out. When `BeginInvoke` is used, what developers want is _sending this `Action` to the `ApplicationIdle` job queue, and execute it when the `Dispatcher` starts to handle the `ApplicationIdle` job queue_, which is equivalent to _let me know when the `Dispatcher` starts to handle the `ApplicationIdle` job queue, then I'll execute this `Action`_. 

With the separation of _wait and execute_ the problem becomes simplified instantly. Suppose we define an interface like this: 

```c#
public interface IDispatcherWaiter
{
  Task<TaskStatus> WaitAsync(DispatcherPriority priority, CancellationToken ct)
}
```

Usage would be like this: 
```c#
private async Task DoSomethingAsync()
{
  ////waiter is an instance of IDispatcherWaiter
  await waiter.WaitAsync(DispatcherPriority.ApplicationIdle);
  DoSomething();
}

private async Task DoSomethingAsync(CancellationToken ct)
{
  ////waiter is an instance of IDispatcherWaiter
  var status = await waiter.WaitAsync(DispatcherPriority.ApplicationIdle, ct);
  if (status == TaskStatus.RanToCompletion)
  {
    DoSomething();
  }  
}
```

In the unit test we just need to `mock` a `IDispatcherWaiter`, or simplify provide a reusable `TestDispatcherWaiter` that directly returns `TaskStatus.RanToCompletion` in the `WaitAsync` method.

Implementation seems also simple enough - `await InvokeAsync` and pass a `CancellationToken`: 

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

## More Analysis

The `await InvokeAsync` above seems to work, only with one critical defect.

We all know that when `await` a `Task`, `ConfigureAwait(true|false)` can also be used to control whether to marshal the continuation back to the original captured `context` or not. `True` is the default behavior: marshal back to the original `context`.

`await DispatcherOperation` behaves the same way as `ConfigureAwait(true)`, so in order for `DoSomething()` to be executed on the main GUI thread, `DoSomethingAsync()` needs to be called on the main GUI thread too.

If `DoSomethingAsync()` is called on some worker thread X as shown below, then `DoSomething()` would be called on the worker thread X when `WaitAsync` is completed. This may cause different behavior compare to `BeginInvoke(()=> DoSomething())` because the `callback` of `BeginInvoke` is invoked on the `Dispatcher` thread. 

```c#
private Task DoSomethingFirstAsync()
{
  return Task.Run(()=>
  {
    ////This is on worker thread X
    DoSomethingAsync();
  });
}
```

The `BeginInvoke(()=> DoSomething())` equivalent should be like this (see comments):

```c#
private Task DoSomethingAsync()
{
  ////The current thread can be any thread
  await waiter.WaitAsync(DispatcherPriority.ApplicationIdle);
  ////The current thread should be the thread of the Dispatcher that is held by the waiter
  DoSomething();
}
```

In short `await Dispatcher.InvokeAsync` would not work. 

## Solution

Fortunately `dotnet` design allows us implement custom `awaitable` objects. The blog [`await anything`](https://devblogs.microsoft.com/pfxteam/await-anything/) wrote by `Microsoft` software engineer [`Stephen Toub`](https://devblogs.microsoft.com/pfxteam/author/toub/) describes everything you need to know about writing custom `awaitable`. 

In the implementation details of the `DispatcherWaiter` below, when `await waiter.WaitAsync` is completed, it would stay on the same `Dispatcher` thread instead of marshal back to the original captured context, which ensures the codes after that be executed on the expected thread.

```c#
////First, define a generic IAwaitable interface, objects implement this interface would be awaitable
public interface IAwaitable<out T, out TResult> : INotifyCompletion where T : class
{
  T GetAwaiter();

  TResult GetResult();

  bool IsCompleted { get; }
}

////Second, define IDispatcherWaiter to extend IAwaitable
public interface IDispatcherWaiter : IAwaitable<IDispatcherWaiter, TaskStatus>
{
  /// <summary>
  /// Returns Dispatcher.CheckAccess
  /// </summary>
  /// <returns></returns>
  bool CheckAccess();
  
  /// <summary>
  /// Calls Dispatcher.VerifyAccess
  /// </summary>
  /// <returns></returns>
  void VerifyAccess();

  /// <summary>
  /// Returns immediately if the caller is onl the Dispatcher thread already, otherwise dispatch to the Dispatcher thread
  /// </summary>
  /// <returns></returns>
  IDispatcherWaiter CheckedWaitAsync();

  /// <summary>
  /// Specify DispatcherPriority and allows cancellation
  /// </summary>
  /// <param name="priority"></param>
  /// <param name="ct"></param>
  /// <returns></returns>  
  IDispatcherWaiter WaitAsync(DispatcherPriority priority = DispatcherPriority.Normal, CancellationToken ct = default(CancellationToken));
}

////Last, the implementation details
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

## Conclusion

Now all of the 3 problems listed above can be considered resolved. Wherever `Dispatcher.BeginInvoke` is needed just replace it with `IDispatcherWaiter.WaitAsync` to have smoother coding experience as well as easier unit tests. 

To recap the usages: 

```c#
////async/await style
private async Task DoSomethingAsync(CancellationToken ct)
{
  var status = await waiter.WaitAsync(DispatcherPriority.ApplicationIdle, ct);
  if (status == TaskStatus.RanToCompletion)
  {
    DoSomething();
  }
}

////callback style similar to BeginInvoke()
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

Extension methods can also be used to further simplify codes and improves readabilities:

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

## Reference
+ [await anything](https://devblogs.microsoft.com/pfxteam/await-anything/)