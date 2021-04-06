> Many large scale applications adopt the concept of `IoC`, it's especially useful in terms of component de-coupling and writing reusable codes. Among all of the IoC containers on the market, [Unity Container](http://Unity Container.org/) is no doubt on top of the list for its rich features and flexibilities. This blog series would focus on introducing some Unity Container advanced usages based on scenarios of real projects to help reduce duplicated works in developments.


## 1. The problem

Logging is one of the features that almost every application cannot miss. Libraries like [log4net](https://logging.apache.org/log4net/) are already written so developers don't need to reinvent the wheels. Those logging support libraries are usually also easy to use, take `log4net` as an example, we just need to call `LogManager.GetLogger` to get an `ILog` instance and start to use it, like the `SomeComponent` class below.


```c#
namespace log4net
{
  public class LogManager
  {
    public static ILog GetLogger(string name);
    public static ILog GetLogger(Type type);
  }
}

public class SomeComponent
{
  private static ILog Logger = LogManager.GetLogger(typeof(SomeComponent));
}
```

While it's straightforward enough to get started, the problems are also obvious enough to notice and should be avoided in large scale aplications:

1. Explicit dependencies to `log4net`. Although it's not common to replace a logging library entirely, but it would be nightmares to developers once it needs to happen.
2. Class name is used to distinguish the context in the log, but explicitly passing class name seems not necessary
3. It's not possible to mock the `ILog` interface when it's a static member variable in unit tests.
4. It's not friendly to advanced scenarios. Assume the `SomeComponent` is a reusable class that different instances have different `Context`, it's natural to also write the `Context` into log to help locate source of the bugs. Using `ILog` directly the above way requires passing `Context` to the logger whenever `ILog.Info` is called, this can easily cause massive duplicated and tedious codes. Using extension methods can reduce some duplications but it's not good enough.

This blog demonstrate a [Unity Container](http://Unity Container.org/) based solution to use the most concise codes to solve the above problems.

## 2. Analysis

#1 can be easily fixed by introducing a new interface, for example [Prism](https://dotnetfoundation.org/projects/prism) provides an `ILogFacade` interface for the same purpose. We'll define an `ILogger` interface for demo purpose. The goal is to use dependency injection to simplify the codes：

```c#
public interface ILogger
{
  void Info(string msg);
}

public class SomeComponent
{
  [Dependency]
  public ILogger Logger { get; set; }   //The Logger instance already contain name of the SomeComponent class, or even the instance of SomeComponent 
}
```

1. `ILogger` implementation can be easily switched to another logging library without touch the client codes.
2. Name of the class being injected can be obtained by `Unity Container` during object construction, it's transparent to the client codes.
3. Dependency injection based approach is easy to `mock` and unit test friendly.
4. Assume the `SomeComponent` class implements a common interface `ILogContext`, during dependency injection, we can create a special instance of `ILogger` and pass the instance of the current `SomeComponent`, then this special logger would be responsible for writing `Context` to log so the client codes can completely be freed from such works.


```c#
public interface ILogContext
{
  object Context { get; }
}
```

> Note 1：`#4` can only be applied to property injection, not constructor injection because the instance of `SomeComponent` is not created yet.

> Note 2：[Unity Container](http://Unity Container.org/) provides flexible customization via `Extension` that allows developers create custom codes to control creation of the objects. But the customization has changed a lot in the version of `Unity Container 4.0` and later, so if not specified, all of the codes in this blog is based on `4.0` and later versions.

We know that there're several `stages` of the object creation in `Unity Container`, below table is quoted from 《[Dependency Injection With Unity](https://download.microsoft.com/download/4/d/b/4dbc771d-9e24-4211-adc5-65812115e52d/dependencyinjectionwithunity.pdf)》：

| Stage                          | Description|
| --------                      | -----  |
| Setup|The first stage. By default, nothing happens here.|
| TypeMapping|The second stage. Type mapping takes place here.|
| Lifetime|The third stage. The container checks for a lifetime manager.|
| PreCreation|The fourth stage. The container uses reflection to discover the constructors, properties, and methods of the type being resolved.|
| Creation|The fifth stage. The container creates the instance of the type being resolved.|
| Initialization|The sixth stage. The container performs any property and method injection on the instance it has just created.|
| PostInitialization|The last stage. By default, nothing happens here.|

For the `ILogger`, we can place a `BuilderStrategy` at the `PreCreation` stage to create `ILogger` instance based on certain conditions, for example, write the log to `Debug Output` or disk file, passing class name etc.

> One article on Microsoft Docs [Case study provided by Dan Piessens](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn178464(v=pandp.30)) shows how to pass class name to `ILogger` based on earlier versions of `Unity Container`, I have enriched the missing details  of the article and put the full workable sample on [github](https://github.com/eagleboost/DanPiessensUnityCaseStudy).

## 3. Implementaions

Core logic is implemented by the `LoggerStrategy` derived from `BuilderStrategy`. Please note that `unsafe` is used the when handling `ILogContext` to retrieve the instance of the object being injected from the `Parent` property of the `BuilderContext`. The reason of `unsafe` is that somehow the `Parent Context` is converted and stored as `IntPtr`, so we have to reverse the process `unsafely`.

```c#
var context = new BuilderContext
{
  ......
#if !NET40
  Parent = new IntPtr(Unsafe.AsPointer(ref thisContext))
#endif
};
```

```c#
public class LoggerStrategy : BuilderStrategy
{
  public override void PreBuildUp(ref BuilderContext context)
  {
    base.PreBuildUp(ref context);

    if (typeof(ILogger).IsAssignableFrom(context.Type))
    {
      context.Existing = CreateLogger(ref context);
      context.BuildComplete = true;
    }
  }

  private static ILogger CreateLogger(ref BuilderContext context)
  {
    var result = LoggerManager.GetLogger(context.DeclaringType);

    ////Try creating ContextLogger if the declaring type implements ILogContext
    if (typeof(ILogContext).IsAssignableFrom(context.DeclaringType))
    {
      if (context.Existing == null && context.Parent != IntPtr.Zero)
      {
        unsafe
        {
          var pContext = Unsafe.AsRef<BuilderContext>(context.Parent.ToPointer());
          ////pContext.Existing is null when ILogger is being injected to the ctor
          if (pContext.Existing is ILogContext logContext)
          {
            ////If ILogger is being injected to a property, we create a ContextLogger to wrap the real logger
            result = new ContextLogger(logContext, result);
          }
        }
      }
    }

    return result;
  }
}
```

`LoggerManager.GetLogger` returns an instance of `ILogger` based on the given type, `LoggerManager` can internally cache the `ILogger` instances per type as well as switch implementations. Below codes show a way of switching to `TraceLogger` when the debugger is attached to write the log to `Debug Output`.

```c#
public static class LoggerManager
{
  private static readonly ConcurrentDictionary<Type, ILogger> Loggers = new ConcurrentDictionary<Type, ILogger>();
  
  public static ILogger GetLogger(Type type)
  {
    return Loggers.GetOrAdd(type, CreateLogger);
  }

  private static ILogger CreateLogger(Type type)
  {
    if (Debugger.IsAttached)
    {
      return (ILogger) Activator.CreateInstance(typeof(TraceLogger<>).MakeGenericType(type));
    }
    
    return (ILogger) Activator.CreateInstance(typeof(Log4NetLogger<>).MakeGenericType(type));
  }
}
```

`ContextLogger` simply write `ILogContext.Context` to log along with the log message.

```c#
public sealed class ContextLogger : ILogger
{
  private readonly ILogger _logger;
  private readonly ILogContext _logContext;

  public ContextLogger(ILogContext logContext, ILogger logger)
  {
    _logContext = logContext ?? throw new ArgumentNullException(nameof(logContext));
    _logger = logger ?? throw new ArgumentNullException(nameof(logger));
  }

  public void Info(string msg)
  {
    LogCore(msg);
  }
  
  private void LogCore(string msg)
  {
    _logger.Info($"{_logContext.Context} : {msg}");
  }
}
```

Please visit [*github*](https://github.com/eagleboost/LoggerInjection) for more details.
