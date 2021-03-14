## 1. The Problem

When using `Tasks` on `.net` platform, exception handling is one of most common problems for developers to deal with. In this blog, however, instead of talking about topics like how to properly `propagate` exceptions in a task chain, we'll only focus on how to gracefully handle exceptions (assume all exceptions are properly propagated).

The most common way is to call `Task.ContinueWith` to create a `continuation` for the task, then check for any exceptions when the task is completed and notify users in some way like displaying a MessageBox.

```c#
CallSomethingAsync().ContinueWith(t =>
{
  if (t.Exception != null)
  {
    ////Handle exceptions here
  }
});
```

This is perfectly fine for small applications or some `POC`, but it would likely cause maintenance headaches for large scale applications:

1. To display a MessageBox when exceptions happen in a `Task`, extra dependencies are required, `MessageBox service` for instance.
2. When `async` calls are many, the same piece of exception handling code need to be repeated as well as the dependencies for MessageBox.

We need some simpler way to handle the `Task` exceptions gracefully.


## 2. Analysis

The `TaskScheduler.UnobservedTaskException` event is one of the solutions for lazy developers. This event is used to receive task exceptions thrown but not observed, e.g. the `Task.Exception` property has never been accessed by external codes. It works, but should not be the top one choice of responsible developers. On one hand, it leave the responsibilities of developers to `CLR`, on the other hand, this event is only triggered when `GC` happens, so it might cause miscommunications in production because the exceptions are actually `delayed` to show to the user. The `Unobserved` button in the sample code demonstrate this behavior。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/HandleTaskExceptionsGracefully.png)

In order to show the exceptions to the user with codes and dependencies as less as possible, the easiest way would be adding an extension method, the method in turn capture the exceptions and send them to `someone` so they can be displayed.

```c#
CallSomethingAsync().WhenFaultedAsync("Failed to xxx");
```

And the `someone` here is the subscribers of the `Dispatcher.DispatcherUnhandledException` event. Not like the `TaskScheduler.UnobservedTaskException` event, the former would trigger as long as the `Dispatcher` flushes its priority queue.

## 3. Implementations

Talk is cheap, let's show the code first. The `WhenFaultedAsync` method accepts two parameters, a `Dispatcher` and a string used to tell the context of the exception. Given that most of the applications only have one `Dispatcher`, the second overload of `WhenFaultedAsync` uses `Application.Dispatcher` as default. For applications with more than one `Dispatchers`, the first overload can be used to pass the current `Dispatcher` if different from `Application.Dispatcher`.

The core part of the code is to marshal to the `Dispatcher` thread and throw an `AggregateException` that contains the context so that is can be captured by the subscribers of the `Dispatcher.DispatcherUnhandledException` event.

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

Below is a sample to display a `MessageBox` in the event handler. For cases of multiple exceptions being thrown in a short time, here are two possible strategies, details won't be covered in this blog:

+ One way is to show the first exception in a `MessageBox` and only log the rest of the exceptions before the `MessageBox` is closed by the user. This is to avoid the famous annoying case of user seeing another `MessageBox` after he closed one.
+ Alternatively, we can send all exceptions to a centralized place, e.g. an always live modeless window, and show them in a list/grid. So all of the exceptions can be presented to the user without interrupting the user's current focus.

```c#
Application.Current.DispatcherUnhandledException += (_, e) =>
{
  ShowException(e.Exception.Message, e.Exception);
  e.Handled = true;
};
```

In certain cases we can even save the call to extension method like `WhenFaultedAsync`. For example, in an application that uses `Unity Container`, `Interception` can be used to intercept all of the async calls of a class (usually a `Service` impalements some interfaces) and automatically inject calls to the `WhenFaultedAsync` extension method. This can help reduce some maintenance efforts for applications with clear architectures

Please visit [*github*](https://github.com/eagleboost/HandleTaskException) for more details.
