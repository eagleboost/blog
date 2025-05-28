---
layout:     post
title:      "Executing Tasks Exclusively"
subtitle:   "——确保任务顺序执行"
date:       2025-05-25
author:     "eagleboost"
header-img: "img/post-bg-cave.jpg"
tags:
    - Task
    - TaskFactory
    - TaskScheduler
    - Semaphore
    - TPL
    - AsyncLock
    - ConcurrentExclusiveSchedulerPair
    - ExclusiveScheduler
---

&emsp;&emsp;优化项目中某项功能时我提出了一个需求，类似于访问`WPF`的界面控件需要在`GUI`线程上一样，我希望某些代码在后台线程执行，但同一时间只能干一件事，这样可以简化代码不需要显示使用锁。

&emsp;&emsp;`ConcurrentExclusiveSchedulerPair`有一个`ExclusiveTaskScheduler`看起来可以用。
>Provides task schedulers that coordinate to execute tasks while ensuring that concurrent tasks may run concurrently and exclusive tasks never do.

>提供任务调度器，协调执行任务，确保并发任务可以同时运行，而独占任务则永远不会同时执行。

&emsp;&emsp;我一开始也这么想，然而事实并非如此。首先我的“`某些代码`”实际上是几段`async/await`的异步代码，即一个`Task`，并非同步执行的`Action`。而基于`ExclusiveTaskScheduler`创建的`TaskFactory`是用来调度一个`Action`或者`Func`而不是调度一个`Task`的执行。如果打开`TaskFactory.StartNew()`方法的提示，是下面这样：

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/TaskFactory.StartNew.png)

&emsp;&emsp;当传入`Func<Task>`后，`TaskFactory.StartNew()在`调用`Func<Task>`把`Task`创建出来就立即返回了，并不会等到创建的`Task`执行完成，因此`TaskFactoryRunner`只实现了我需要的功能的一半——其实是对前面关于`ExclusiveTaskScheduler`的文档描述有误解，其中说的`tasks`指的是某些需要执行的代码，并非`.Net`中的`Task`对象。

```c#
public class TaskFactoryRunner
{
  private readonly TaskFactory _taskFactory = new(new ConcurrentExclusiveSchedulerPair().ExclusiveScheduler);

  public Task RunAsync(Func<Task> taskFunc)
  {
   return _taskFactory.StartNew(() => taskFunc());
  }
}
```

&emsp;&emsp;想明白关节后我发现`Stephen Cleary`[在一篇博客](https://blog.stephencleary.com/2012/08/async-and-scheduled-concurrency.html)中也有类似叙述，并提到`Stephen Toub`的[AsyncLock](https://devblogs.microsoft.com/dotnet/building-async-coordination-primitives-part-6-asynclock/)可以用来提供异步锁机制——题外话，`Stephen Toub`这个系列的`Async Coordination Primitives`博客文章非常值得读，我在项目中也用到了`AsyncLock`。

>Note: When an asynchronous method awaits, it returns back to its context. This means that ExclusiveScheduler is perfectly happy to run one task at a time, not one task until it completes. As soon as an asynchronous method awaits, it’s no longer the “owner” of the ExclusiveScheduler. Stephen Toub’s async-friendly primitives like AsyncLock use a different strategy, allowing an asynchronous method to hold the lock while it awaits.

>注意： 异步方法在等待时会返回到其上下文。这意味着 ExclusiveScheduler 非常乐意一次运行一个任务 （而非一个任务直到其完成 ）。一旦异步方法开始等待，它就不再是 ExclusiveScheduler 的“拥有者”。Stephen Toub 的异步友好原语如 AsyncLock 采用了不同策略，允许异步方法在等待期间保持锁。

&emsp;&emsp;其实解决办法很简单，只需要调用`Wait()`方法等待`Task`执行完成即可：

```c#
public class TaskFactoryRunnerWithWait
{
  private readonly TaskFactory _taskFactory = new(new ConcurrentExclusiveSchedulerPair().ExclusiveScheduler);

  public Task RunAsync(Func<Task> taskFunc)
  {
   return _taskFactory.StartNew(() => taskFunc().Wait());
  }
}
```


&emsp;&emsp;当然使用`AsyncLock`也能轻松实现我需要的功能，像下面这样：

```c#
public class TaskRunnerWithAsyncLock
{
  private readonly AsyncLock _asyncLock = new();

  public async Task RunAsync(Func<Task> taskFunc)
  {
    using var async = await _asyncLock.LockAsync().ConfigureAwait(false);
    await taskFunc().ConfigureAwait(false);
  }
}
```
&emsp;&emsp;到这里问题似乎差不多解决了，但并没有完。我把这两天刚发布的`Claude AI 4`拉出来问了问。它给出了好几个答案，其中直接能工作的有两个：

```c#
////使用TPL Task chainning
public class SequentialTaskExecutor
{
  private Task _lastTask = Task.CompletedTask;
  private readonly object _lock = new object();

  public Task RunAsync(Func<Task> taskFunc)
  {
    lock (_lock)
    {
      _lastTask = _lastTask.ContinueWith(async _ => await taskFunc()).Unwrap();
      return _lastTask;
    }
  }
}

public class TaskRunnerWithSemaphoreSlim
{
  private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(1, 1);

  public async Task RunAsync(Func<Task> taskFunc)
  {
    await _semaphore.WaitAsync();
    try
    {
      await taskFunc().ConfigureAwait(false);
    }
    finally
    {
      _semaphore.Release();
    }
  }
}
```

&emsp;&emsp;当我问它能否使用`ExclusiveTaskScheduler`来实现的时候它先是给出了与`TaskFactoryRunner`初始版本类似的错误答案。我告诉它代码不能正常工作并让它找问题，它听懂并给出了下面的正确代码，非常不错。


```c#
public class TaskFactoryRunnerWithGetAwaiter
{
  private readonly TaskFactory _taskFactory = new(new ConcurrentExclusiveSchedulerPair().ExclusiveScheduler);

  public Task RunAsync(Func<Task> taskFunc)
  {
    return _taskFactory.StartNew(() => taskFunc().GetAwaiter().GetResult());
  }
}
```

&emsp;&emsp;现在我们有了几个版本的实现，那么该选哪一个呢？回归一下需求会发现`TaskRunnerWithAsyncLock`和`TaskRunnerWithSemaphoreSlim`并不能直接用，因为需要在“`后台线程`”执行代码避免阻塞主界面，而它们会在调用线程（通常是主线程）执行，所以需要额外调用`Task.Run`才行：

```c#
await Task.Run(taskFunc).ConfigureAwait(false);
```

&emsp;&emsp;下面是以`TaskFactoryRunnerWithGetAwaiter`为基准连续调度`100`个`Task`的测试结果：

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/TaskRunnerBenchmark.png)

&emsp;&emsp;不意外，所有版本的执行效率相差无几。

&emsp;&emsp;`TaskRunnerWithAsyncLock`通过`AsyncLock`实现了`100%`优雅的`async/await`代码，但`AsyncLock`本身有开销，而且需要调用`Task.Run`产生额外内存开销，所以`#1`和`#2`出局。

&emsp;&emsp;`SemaphoreSlim`内存开销虽然最小，但是也需要额外调用`Task.Run`才能保证代码在后台线程运行，所以`#3`和`#4`也出局。

&emsp;&emsp;`SequentialTaskExecutor`创建`Task Chain`有额外开销无法进一步优化也出局。

&emsp;&emsp;最后剩下`#5`和`#6`两个基于`TaskFactory`的实现，内存开销也最小。`#5`使用`Task.Wait()`当异常发生时会被包装进一个`AggregateException`，而`#6`使用`Task.GetAwaiter().GetResult()`会抛出原始异常，因此`#6`，也就是基准测试`TaskFactoryRunnerWithGetAwaiter`胜出。

&emsp;&emsp;需要注意的是`TaskFactoryRunnerWithGetAwaiter`能用的前提是`taskFunc()`创建的`Task`必须始终在后台线程执行，否则会死锁。如果需要处理`Task`可能切换到主线程执行的情况，最好的办法其实是`#4`，虽然有额外开销，但不会阻塞调用线程。