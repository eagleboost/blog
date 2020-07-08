## 1. The problem

`ToolTip` binding is one of the common problems of `WPF` applications. For example, the binding source has a `Int32` property `Number`, its value can be positive or negative, based on the scenario sometimes we want to display the values in different forms:

| Value     | Display  |
| --------| ----- |
| 1000    | 1000  |
| -1000   | (1000)|
| 0       | Zero  |

`Binding` has built-in support for [*StringFormat*](https://docs.microsoft.com/en-us/dotnet/api/system.windows.data.bindingbase.stringformat?view=netcore-3.1), along with [*Custom numeric format strings*](https://docs.microsoft.com/en-us/dotnet/standard/base-types/custom-numeric-format-strings#the--section-separator) anyone can easily come up with this:

```xml
<TextBlock Text="{Binding Number, StringFormat={}{0:##;(##);Zero}}"/>
```

The problem is that the same piece of code does not work for `ToolTip`. No errors show up at either compile time or run time, the only thing is based on the result we can tell `StringFormat` is ignored somehow.

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/ToolTipBinding_Bad.png)


## 2. Analysis

It's actually an old problem - the [*FrameworkElement.ToolTip*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/FrameworkElement.cs,dfa9b59a70d7968c) property is of type `object`, which allows developers put anything in it. When it comes to display, it's the [*ToolTip*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/Controls/ToolTip.cs,3b75d3030dfed404) control does the real job . The `ToolTip` control is a `ContentControl`, so we need to use [*ContentStringFormat*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/Controls/ContentControl.cs,9ac210145afd84c6) to make it work:

```xml
<TextBlock Text="{Binding Number, StringFormat={}{0:##;(##);Zero}}">
  <TextBlock.ToolTip>
    <ToolTip Content="{Binding Number}" ContentStringFormat="{}{0:##;(##);Zero}"/>
  </TextBlock.ToolTip>
</TextBlock>
```

`ToolTip` is flexible enough to display complex user interface, but `99.99%` of the time we just want to display some strings. It seems to me not worth making `XAML` complicated for such a tiny task. We know that sometimes `Microsoft` does tricks to make developers' life easier, in my pinion this could have been one of those tricks, but they didn't do it.

Luckily that `WPF` is super flexible, so let's do it on our own.

## 3. Implementations

There're more than one choices we can use, in this post we use custom binding approach to achieve the most intuitive usage, it's again based on the [*BindingDecoratorBase*](http://www.hardcodet.net/2008/04/wpf-custom-binding-class) used in my previous article [*EnabledStateBinding*](https://eagleboost.com/2020/06/11/EnabledStateBinding/):


```c#
public sealed class ToolTipBinding : BindingDecoratorBase
{
  public ToolTipBinding()
  {
  }
  
  public ToolTipBinding(string path)
  {
    Path = new PropertyPath(path);
  }
  
  public override object ProvideValue(IServiceProvider provider)
  {
    if (StringFormat == null)
    {
      throw new ArgumentException("StringFormat is not set", nameof(StringFormat));
    }

    var converter = Binding.Converter;
    var newConverter = new ToolTipConverter {Converter = converter, Format = StringFormat};
    Binding.Converter = newConverter;
    
    return Binding.ProvideValue(provider);
  }
  
  private class ToolTipConverter : IValueConverter
  {
    public IValueConverter Converter { get; set; }
    
    public string Format { get; set; }

    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
      var v = value;
      var cvt = Converter;
      if (cvt != null)
      {
        ////If there's a Converter comes with the original binding, use it first
        v = cvt.Convert(value, targetType, parameter, culture);
      }
      
      return string.Format(Format, v);
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
      throw new NotImplementedException();
    }
  }
```

Here's the usage:

```xml
<TextBlock Text="{Binding Number, StringFormat={}{0:##;(##);Zero}}" 
           ToolTip="{local:ToolTipBinding Number, StringFormat={}{0:##;(##);Zero}}"/>
```

We can further optimize the code based on specific cases. For example, the `AutoBinding` (implementation is omitted) below can automatically generate binding for the `ToolTip` property based on the binding of the `Text` property.

```xml
<TextBlock Text="{local:AutoBinding Number, StringFormat={}{0:##;(##);Zero}}"/>
```

## References
+ [*EnabledStateBinding*](https://eagleboost.com/2020/06/11/EnabledStateBinding)
+ [*BindingDecoratorBase*](http://www.hardcodet.net/2008/04/wpf-custom-binding-class)