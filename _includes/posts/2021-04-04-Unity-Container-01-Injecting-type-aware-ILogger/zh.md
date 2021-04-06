> 在大型应用程序中使用`IoC`对于解耦和开发可重用代码大有益处，选择一个强大的`IoC`容器则可能使得应用程序开发如虎添翼。[Unity Container](http://Unity Container.org/)无疑是众多`IoC`容器中的佼佼者。本系列旨基于实际项目介绍一些`Unity Container`的高级使用方式，帮助减少开发中的一些重复劳动。


## 1. 问题

记录日志几乎可算是每个应用程序都必不可少的功能，相关的支持库也不少，比如.net环境下最常用的[log4net](https://logging.apache.org/log4net/)。这些日志库通常使用起来也非常方便，添加引用，再简单配置一下`config`即可。

以`log4net`为例，一般是调用`LogManager`来获取一个`ILog`实例，比如下面的`SomeComponent`类。


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

但这样做的问题也是显而易见的，在大型应用程序中尤其需要避免：

1. 引入了对`log4net`的显示依赖。虽然现实中整体替换日志库的场景不多，但一旦出现对开发者来说就是噩梦。
2. 类名用于在日志中输出当前类信息，但是显示传递类名导致代码冗长。
3. 静态成员变量无法通过`moq`或`NSubstitue`之类的工具来`mock`日志`ILog`接口，不利于单元测试。
4. 对高级使用场景不友好。假如上面的`SomeComponent`是一个可重用的类，但不同的实例有不同的`Context`，记录日志的时候很自然需要记录该`Context`来帮助定位错误来源。直接使用`ILog`就需要在每个调用`ILog.Info`的地方把`Context`拼接为字符串的一部分传入，这容易导致大量重复而冗长的代码。引入扩展方法可以在一定程度上减少重复，但并不优美。

本文给出一种基于[Unity Container](http://Unity Container.org/)的解决方案，以最简洁的代码解决上述问题。

## 2. 分析

引入一个新的接口即可避免对`log4net`的显示依赖，比如[Prism](https://dotnetfoundation.org/projects/prism)库就有一个`ILogFacade`接口。为简单起见，本文定义一个`ILogger`接口，其成员可仿照`ILogFacade`接口，不再赘述。我们的目标是通过依赖注入`ILogger`实例的方式来简化代码，如下所示：

```c#
public interface ILogger
{
  void Info(string msg);
}

public class SomeComponent
{
  [Dependency]
  public ILogger Logger { get; set; }   //该实例已经包含了SomeComponent的类，甚至实例信息
}
```

1. `ILogger`的实现容易替换为另外的类库而无需更改其余代码。
2. 类名可以由`Unity Container`在注入的时候通过上下文获取，无需对使用者可见。
3. 易于`mock`接口，方便单元测试。
4. 以不同的实例有不同的`Context`为例，假设`SomeComponent`实现了一个基础接口`ILogContext`，依赖注入的时候一旦检测到`ILogContext`则可以构建一个特殊的`ILogger`并传入当前`SomeComponent`的实例，那么把`Context`一并输出到日志的工作则可以由该`ILogger`完成，对使用者来说完全透明。


```c#
public interface ILogContext
{
  object Context { get; }
}
```

> 注 1：`#4`由于与实例相关，因此仅适用于属性依赖注入的情形，因为注入到构造函数的时候`SomeComponent`的实例尚未创建完成。

> 注 2：[Unity Container](http://Unity Container.org/)作为老牌的`IOC`容器，通过`Extension`对于组件创建的全过程提供了灵活的可定制化功能使得开发者可以插入代码来控制组件的创建。`Unity Container 4.0`以后的版本在定制化方面有较大的改动，使得定制化的实现也大不相同。如无特殊说明，本文所有代码均以`4.0`以后的版本为例。

我们知道`Unity Container`中组件的创建有不同的`stage`，下面的表格来自《[Dependency Injection With Unity](https://download.microsoft.com/download/4/d/b/4dbc771d-9e24-4211-adc5-65812115e52d/dependencyinjectionwithunity.pdf)》：

| Stage                          | Description|
| --------                      | -----  |
| Setup|The first stage. By default, nothing happens here.|
| TypeMapping|The second stage. Type mapping takes place here.|
| Lifetime|The third stage. The container checks for a lifetime manager.|
| PreCreation|The fourth stage. The container uses reflection to discover the constructors, properties, and methods of the type being resolved.|
| Creation|The fifth stage. The container creates the instance of the type being resolved.|
| Initialization|The sixth stage. The container performs any property and method injection on the instance it has just created.|
| PostInitialization|The last stage. By default, nothing happens here.|

对于`ILogger`来说可以在`PreCreation`阶段插入一个`BuilderStrategy`来根据一定条件实例化一个日志类，比如是把日志输出到文件还是调试器的`Debug Output`，被创建的类名信息等等。

> Microsoft Docs上有一篇[Case study provided by Dan Piessens](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn178464(v=pandp.30))给出了基于`Unity Container`早期版本一个实现，用于把当前被创建的类名传递给`ILogger`，我把原文略去的一些细节补充之后放在了[github](https://github.com/eagleboost/DanPiessensUnityCaseStudy)上。

## 3. 实现

核心逻辑由继承自`BuilderStrategy`的`LoggerStrategy`完成，代码非常简单。值得注意的是对`ILogContext`的处理使用了`unsafe`代码从当前`BuilderContext`的`Parent`属性获取`ILogger`被注入为属性的类的实例。`unsafe`的原因是`Parent Context`的引用不知为什么被转换为`IntPtr`来传递，所以也只能用`unsafe`方式转换过后来访问。

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

    ////如果declaring type实现了ILogContext，则创建一个ContextLogger
    if (typeof(ILogContext).IsAssignableFrom(context.DeclaringType))
    {
      if (context.Existing == null && context.Parent != IntPtr.Zero)
      {
        unsafe
        {
          var pContext = Unsafe.AsRef<BuilderContext>(context.Parent.ToPointer());
          ////如果ILogger是被依赖注入到构造函数，那么pContext.Existing就是null
          if (pContext.Existing is ILogContext logContext)
          {
            ////如果ILogger是被依赖注入到属性，那么pContext.Existing就是已经创建的被注入到对象实例。我们创建ContextLogger并把该对象引用以及实际做事的ILogger传入
            result = new ContextLogger(logContext, result);
          }
        }
      }
    }

    return result;
  }
}
```

`LoggerManager.GetLogger`根据给定类型返回一个`ILogger`的实例，`LoggerManager`可以包含按类型缓存`ILogger`实例以及切换不同实现的逻辑。比如下面的代码演示了当调试器加载的情况下使用`TraceLogger`从而把所有日志输出到`Debug Output`。

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

`ContextLogger`则简单地把`ILogContext.Context`与日志信息拼接起来一并写入。

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

完整实现请移步[*github*](https://github.com/eagleboost/LoggerInjection)
