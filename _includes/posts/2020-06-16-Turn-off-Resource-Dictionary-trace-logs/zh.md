## 1. 问题

在开发`WPF`应用的过程中，开发者常常会在调试器（`Visual Studio`，`Rider`等）的输出窗口看到下面的输出：

```c#
System.Windows.ResourceDictionary Warning: 9 : Resource not found; ResourceKey='xxxxxx'
```

一般来说这表明程序有问题，因为`WPF`基础库在运行时找不到某个资源，结果可能导致某些空间显示异常，开发者应该引起重视。

值得注意的是“`Resource not found`”这种警告只在使用[*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension)的情况下发生，一般来说同时也需要调试器存在，而生产环境并不受影响。使用[*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension)的目的是告知`WPF`把资源查询的操作延迟到该资源真正被访问的时候发生。

[*StaticResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/staticresource-markup-extension)则不同。使用[*StaticResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/staticresource-markup-extension)的情况下，在把`XAML`载入内存的时候`WPF`就会搜索资源系统，找到资源并设置给相应的对象。如果资源找不到就会抛出异常而不是输出警告，因为[*StaticResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/staticresource-markup-extension)是静态的，除非运行时显示修改，否则设置好之后在相应的界面生命周期中是不变的。

[*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension)的一个典型应用是用来实现支持换肤的程序。大多数时候换肤只不过是界面的颜色、字体、背景之类发生改变，这种情况下如果抛异常似乎有点过分，毕竟程序的功能不会受影响。[*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension)的设计思路大概也考虑到了这一点，所以输出警告更为合理。


尽管“`Resource not found`”警告需要引起开发者的注意，但当我们并不希望看到的时候，它们就成了输出窗口中的噪音。所以本文要讨论的问题就是怎样关掉这种警告。


## 2. 分析

首先需要明确的是在官方文档中找不到已知方法来关掉这种警告。在[*Stack Overflow*](https://stackoverflow.com/questions/10822003/strange-resource-dictionary-warnings-appear-in-output-window-even-when-the-wpf-t)上的一个问题的回答中可以找到以下回答来自微软技术支持部门的回答：

> Thanks for the update. I was afraid of that, since my testing found similar results. It seems there is some internal WPF tracing code which does not adhere to the specified settings. In the meantime, We don’t have any suggestions other than finding the Resource Dictionary (or the relevant type) and correcting the issues that the trace output is warning about.

> if a debugger was attached, there will always be some WPF tracing emitted regardless of the settings specified in the IDE (or in the app.config). Unfortunately, the output you are receiving appears to fall into this category. Regrettably, there is no way to turn off all the WPF trace output from being emitted

> We could certainly file a feature request for the product for this to be considered in a future release, but otherwise I don’t see a way for you to avoid the issue in the current release.

所以我们只能靠自力更生，看下面的分析：

注意到输出以"`System.Windows.ResourceDictionary`"开头，所以先来看看[*ResourceDictionary*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/ResourceDictionary.cs,08d992ee7e0a1516)的源代码。很容易发现很多类似于下面的调用，显然真正的日志输出操作是由`TraceResourceDictionary`来完成的：

```c#
if (TraceResourceDictionary.IsEnabled)
{
  TraceResourceDictionary.Trace(...);
}
```

`TraceResourceDictionary`是在[*Reference Source*](https://referencesource.microsoft.com/)上没有源代码的内部类，我们可以用反编译工具来查看其代码。类似工具很多，我最常用的是来自[*JetBrains*](https://www.jetbrains.com)的免费工具[*dotPeek*](https://www.jetbrains.com/decompiler/)：

```c#
internal static class TraceResourceDictionary
{
  private static AvTrace _avTrace = new AvTrace((GetTraceSourceDelegate) (...);
  
  public static bool IsEnabled
  {
    get
    {
      if (_avTrace != null)
        return _avTrace.IsEnabled;
      return false;
    }
  }

  public static bool IsEnabledOverride
  {
    get
    {
      return _avTrace.IsEnabledOverride;
    }
  }
}
```

上面的代码略去了大多数实现细节，重点是两个属性`IsEnabled`和`IsEnabledOverride`. `IsEnabled`在很多地方被调用到，但是`IsEnabledOverride`的使用者只有一处——`FrameworkElement.FindResourceInternal`，碰巧跟`ResourceNotFound`相关:

```c#
internal static object FindResourceInternal(...)
{
  ...
  if (TraceResourceDictionary.IsEnabledOverride && !isImplicitStyleLookup)
  {
    if (fe != null && fe.IsLoaded || fce != null && fce.IsLoaded)
      TraceResourceDictionary.Trace(TraceEventType.Warning, TraceResourceDictionary.ResourceNotFound, resourceKey);
    else if (TraceResourceDictionary.IsEnabled)
      TraceResourceDictionary.TraceActivityItem(TraceResourceDictionary.ResourceNotFound, resourceKey);
  }
  ...
}
```

再进一步可以看到`AvTrace.IsEnabledOverride`属性只是简单查询`_traceSource`字段是否为空，`_traceSource`在某些情况下会被创建，比如`Windows`注册表中打开了`ManagedTracing`开关，或者当前处在调试器环境中，或者一个标志显示刷新的静态变量为真。

要达到关闭“`Resource not found`”警告的，在不考虑显示刷新（刷新并不常见）以及`ManagedTracing`默认关闭的情况下，只需要想办法把`_traceSource`设置为空即可。

> `TraceResourceDictionary.IsEnabled`的用途超出本文讨论的范围，此处略去。


```c#
internal class AvTrace
{
  private TraceSource _traceSource;

  public bool IsEnabledOverride
  {
    get
    {
      return this._traceSource != null;
    }
  }

  private void Initialize()
  {
    if (AvTrace.ShouldCreateTraceSources())
    {
      this._traceSource = this._getTraceSourceDelegate();
      this._isEnabled = AvTrace.IsWpfTracingEnabledInRegistry() || AvTrace._hasBeenRefreshed || this._enabledByDebugger;
    }
    else
    {
      this._clearTraceSourceDelegate();
      this._traceSource = (TraceSource) null;
      this._isEnabled = false;
    }
  }

  private static bool ShouldCreateTraceSources()
  {
    return AvTrace.IsWpfTracingEnabledInRegistry() || AvTrace.IsDebuggerAttached() || AvTrace._hasBeenRefreshed;
  }
```

## 3. 实现

解决方案清楚之后实现则顺理成章——使用反射找到类型及字段，清除即可。至于调用时机，由于`TraceResourceDictionary`在`WPF`初始化的过程中很早就被访问，所以只需要重载`Application.OnStartup`并在其中调用`SkipResourceNotFound.Install()`。

```c#
////GetNonPublicStaticField和GetNonPublicField是两个扩展方法，其功能如其名称所示
public static class SkipResourceNotFound
{
  public static void Install()
  {
    try
    {
      var typeName = "MS.Internal.TraceResourceDictionary";
      ////通过类名找到相应的类 MS.Internal.TraceResourceDictionary
      var traceResDict = FindType(typeName);
      ////取得_avTrace字段
      var avTrace = traceResDict.GetNonPublicStaticField("_avTrace").GetValue(null);
      var avTraceType = avTrace.GetType();
      ////取得_avTrace._traceSource字段并设置为null
      var traceSourceField = avTraceType.GetNonPublicField("_traceSource");
      traceSourceField.SetValue(avTrace, null);
    }
    catch
    {
    }
  }

  private static Type FindType(string typeName)
  {
    Type type = null;
    foreach (var assembly in AppDomain.CurrentDomain.GetAssemblies())
    {
      type = Type.GetType(typeName + "." + assembly.GetName());
      if (type != null)
      {
        return type;
      }
    }

    return null;
  }
}
```

## 参考资料
+ [*StaticResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/staticresource-markup-extension)
+ [*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension)
+ [*Stack Overflow*](https://stackoverflow.com/questions/10822003/strange-resource-dictionary-warnings-appear-in-output-window-even-when-the-wpf-t)
+ [*ResourceDictionary*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/ResourceDictionary.cs,08d992ee7e0a1516)