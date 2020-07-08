## 1. 问题

`ToolTip`的绑定也是`WPF`应用程序开发的常见问题之一。比如数据绑定的源对象有一个`Int32`类型的属性`Number`，数值可能为正数也可能为负数，根据不同场景我们有时候希望显示为不同的形式，比如：

| 数值     | 显示  |
| --------| ----- |
| 1000    | 1000  |
| -1000   | (1000)|
| 0       | Zero  |

借助`Binding`对于[*StringFormat*](https://docs.microsoft.com/en-us/dotnet/api/system.windows.data.bindingbase.stringformat?view=netcore-3.1)的内建支持，配合[*Custom numeric format strings*](https://docs.microsoft.com/en-us/dotnet/standard/base-types/custom-numeric-format-strings#the--section-separator)可以轻松写出如下实现：

```xml
<TextBlock Text="{Binding Number, StringFormat={}{0:##;(##);Zero}}"/>
```

问题是把同样的代码放到`ToolTip`行不通，不论是编译时还是运行时都不会报错，但从结果可以看出`StringFormat`被忽略了。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/ToolTipBinding_Bad.png)


## 2. 分析

这实际上是个老生常谈的问题——[*FrameworkElement.ToolTip*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/FrameworkElement.cs,dfa9b59a70d7968c)类型是`object`，可以放任何类型的数据，而真正显示是由[*ToolTip*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/Controls/ToolTip.cs,3b75d3030dfed404)完成的。而`ToolTip`是个`ContentControl`，要用[*ContentStringFormat*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/Controls/ContentControl.cs,9ac210145afd84c6)才能起作用，如下所示：

```xml
<TextBlock Text="{Binding Number, StringFormat={}{0:##;(##);Zero}}">
  <TextBlock.ToolTip>
    <ToolTip Content="{Binding Number}" ContentStringFormat="{}{0:##;(##);Zero}"/>
  </TextBlock.ToolTip>
</TextBlock>
```

`ToolTip`很强大可以显示复杂信息不假，但`99.99%`的时候我们只是要显示一串字符而已，为了这芝麻大点事把`XAML`搞复杂有点本末倒置。在`WPF`中有时候微软会有一些小`trick`为开发者提供便利，在绑定上为`ToolTip`提供`StringFormat`支持应该是可以提供的便利，然而并没有。

好在`WPF`极度灵活，自己实现一下并非难事。

## 3. 实现

解决办法不少，本文采用自定义绑定的方法以达到最直观的使用形式，同样基于我之前的文章[*EnabledStateBinding*](https://eagleboost.com/2020/06/11/EnabledStateBinding/)中用到的[*BindingDecoratorBase*](http://www.hardcodet.net/2008/04/wpf-custom-binding-class)。


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
        ////如果原先的绑定有Converter，先调用Converter把数据转换一下
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

使用方式如下：

```xml
<TextBlock Text="{Binding Number, StringFormat={}{0:##;(##);Zero}}" 
           ToolTip="{local:ToolTipBinding Number, StringFormat={}{0:##;(##);Zero}}"/>
```

根据实际情况，还可以进一步简化。比如下面的`AutoBinding`（实现略去）在为`TextBlock`的`Text`属性创建绑定的同时也自动为`ToolTip`生成相关绑定。

```xml
<TextBlock Text="{local:AutoBinding Number, StringFormat={}{0:##;(##);Zero}}"/>
```

## 参考资料
+ [*EnabledStateBinding*](https://eagleboost.com/2020/06/11/EnabledStateBinding)
+ [*BindingDecoratorBase*](http://www.hardcodet.net/2008/04/wpf-custom-binding-class)