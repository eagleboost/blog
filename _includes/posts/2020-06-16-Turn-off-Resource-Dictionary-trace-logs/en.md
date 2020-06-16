## 1. The problem

When working with `WPF` applications, it's common to see logs like below in the `Output` window of the debugger (`Visual Studio`, `Rider`, etc).

```c#
System.Windows.ResourceDictionary Warning: 9 : Resource not found; ResourceKey='xxxxxx'
```

Usually developers should pay attention because it means certain resource cannot be found by the `WPF` infrastructure and it may result in some controls in the UI not being displayed properly.

This '`Resource not found`' trace log only happens to the resources marked by the [*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension) markup extension and when the debugger is attached. With [*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension) the resource is not resolved until runtime when the expression is evaluated. 

[*StaticResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/staticresource-markup-extension) markup extension is different. It causes resolving of the resource happen during loading of the `XAML` , when a resource cannot be resolved, exceptions would be thrown. It's static, set once and never change (unless explicitly).

One typical use case of the [*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension) is implementing theme aware applications. Most of the time switching theme only turns the `GUI` into different look and feel by changing foreground, background, font etc, so throwing exception may be too much of noises. This possibly could be one of the design considerations of [*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension).

Even though developers should pay attention but in some cases they becomes noises in the output when we don't want to see them, and the problem is now how can we turn it off.


## 2. Analysis

First of all, there is no official documented way to turn it off. Below is some answers from Microsoft support team can be found on [*Stack Overflow*](https://stackoverflow.com/questions/10822003/strange-resource-dictionary-warnings-appear-in-output-window-even-when-the-wpf-t):

> Thanks for the update. I was afraid of that, since my testing found similar results. It seems there is some internal WPF tracing code which does not adhere to the specified settings. In the meantime, We don’t have any suggestions other than finding the Resource Dictionary (or the relevant type) and correcting the issues that the trace output is warning about.

> if a debugger was attached, there will always be some WPF tracing emitted regardless of the settings specified in the IDE (or in the app.config). Unfortunately, the output you are receiving appears to fall into this category. Regrettably, there is no way to turn off all the WPF trace output from being emitted

> We could certainly file a feature request for the product for this to be considered in a future release, but otherwise I don’t see a way for you to avoid the issue in the current release.

So most likely we have to hack it some how, and let's find out how.

Because the log starts with `System.Windows.ResourceDictionary` so we first take a look at the source code of the [*ResourceDictionary*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/ResourceDictionary.cs,08d992ee7e0a1516) class. It's obvious to see the tracing is done by `TraceResourceDictionary` because of the usages like this:

```c#
if (TraceResourceDictionary.IsEnabled)
{
  TraceResourceDictionary.Trace(...);
}
```

However, `TraceResourceDictionary` is an internal class and its source codes are not available on [*Reference Source*](https://referencesource.microsoft.com/). There're plenty of free disassemble tools out there, my favorite one is the awesome free .net Decompiler and Assembly Browser [*dotPeek*](https://www.jetbrains.com/decompiler/)

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

Most of the details are omitted, notice that this static class has two properties `IsEnabled` and `IsEnabledOverride`. `IsEnabled` used in many places but `IsEnabledOverride` is only used by the `FrameworkElement.FindResourceInternal`, and it happen to be logging trace messages of `ResourceNotFound`:

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

Look further into `AvTrace.IsEnabledOverride`, it simply does null reference check against a private field `_traceSource`, which may be created given whether the `ManagedTracing` flag is enabled in the `Windows Registry`, or the is the debugger attached, or a flag that indicating explicit refresh is true.

To get what we want to turn the warning trace off, without considering refresh (uncommon) and `ManagedTracing` being disabled by default in the `Windows Registry`, the solution is to set `_traceSource` to null so that `TraceResourceDictionary.IsEnabledOverride` would return false and trace logs would be skipped.

>`TraceResourceDictionary.IsEnabled` is used for other cases and beyond the scope so we skip discussing it in this article.


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

## 3. Implementations

Coding is fairly straightforward with Reflection calls all the way to clear out `_traceSource`. In terms of timing, since `TraceResourceDictionary` is accessed pretty early during the initialization of a WPF application so we can call `SkipResourceNotFound.Install()` right in the `Application.OnStartup` override method.

```c#
////GetNonPublicStaticField and GetNonPublicField are extensions methods that does what their name suggests
public static class SkipResourceNotFound
{
  public static void Install()
  {
    try
    {
      var typeName = "MS.Internal.TraceResourceDictionary";
      ////Get the type of MS.Internal.TraceResourceDictionary
      var traceResDict = FindType(typeName);
      ////Get the static private field of _avTrace
      var avTrace = traceResDict.GetNonPublicStaticField("_avTrace").GetValue(null);
      var avTraceType = avTrace.GetType();
      ////Get private field of _traceSource and set it to null
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

## References
+ [*StaticResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/staticresource-markup-extension)
+ [*DynamicResource*](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/dynamicresource-markup-extension)
+ [*Stack Overflow*](https://stackoverflow.com/questions/10822003/strange-resource-dictionary-warnings-appear-in-output-window-even-when-the-wpf-t)
+ [*ResourceDictionary*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/ResourceDictionary.cs,08d992ee7e0a1516)