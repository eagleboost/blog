## 问题
一般来说[TabControl](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol?view=netframework-4.8)有两种用法:
1. 在XAML里直接把[TabItem](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabitem?view=netframework-4.8)添加到`TabControl.Items`，再把要显示的控件作为TabItem的内容。

```html
<TabControl>
  <TabControl.Items>
    <TabItem Header="Tab 1">
      <TextBlock Text="Tab Item Content 1"/>
    </TabItem>
      <TabItem Header="Tab 2">
    <TextBlock Text="Tab Item Content 2"/>
    </TabItem>
  </TabControl.Items>
</TabControl>
```
2. 在[M-V-VM](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/enterprise-application-patterns/mvvm)的编程模式下则通常把TabControl的[ItemsSource](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol.itemssource?view=netframework-4.8)属性绑定到数据源，再用[ContentTemplate](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.contenttemplate?view=netframework-4.8)及[ContentTemplateSelector](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.contenttemplateselector?view=netframework-4.8)根据数据项生成UI上的控件，不同的数据项（可以是类型不同，也可以是属性不同）可以映射到不同的模板，最终生成的控件也可以不同。

```html
<TabControl ItemsSource="{Binding DataSource}">
  <TabControl.ItemTemplate>
    <DataTemplate>
     <TextBlock Text="{Binding Header}"/>
    </DataTemplate>
  </TabControl.ItemTemplate>
  <TabControl.ContentTemplate>
    <DataTemplate>
      <TextBlock Text="{Binding Content}"/>
    </DataTemplate>
  </TabControl.ContentTemplate>  
</TabControl>
```

虽然看起来差不多，然而就TabControl的设计来说，这两种方式存在一个最大的不同:
1. 在XAML里直接签加控件作为TabItem的[内容](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.contentcontrol.content?view=netframework-4.8)以后，控件会在所属的TabItem第一次激活时被创建并且缓存在所属TabItem的Content属性里。当同一个TabItem再次被激活的时候，它Content属性的内容就会被直接用来替换[TabControl.SelectedContent](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.selectedcontent?view=netframework-4.8)，从而立刻显示出来。
2. 在使用[ContentTemplate](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.contenttemplate?view=netframework-4.8)/[ContentTemplateSelector](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.contenttemplateselector?view=netframework-4.8)的情况下, TabItem的[内容](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.contentcontrol.content?view=netframework-4.8)不再是控件而变成了数据项，[TabControl.SelectedContent](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.selectedcontent?view=netframework-4.8)也首先会是数据项，TabControl会在[SelectedContent](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.selectedcontent?view=netframework-4.8)属性每次改变的时候根据[ContentTemplate](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.contenttemplate?view=netframework-4.8)/[ContentTemplateSelector](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol.contenttemplateselector?view=netframework-4.8)重新按模板生成控件。模板简单的时候这种方式问题不大，但如果模板很复杂，每次重新创建控件都花费较长时间的话用户体验就会大打折扣。

事实上`#2`在真实开发场景中用得做多。[M-V-VM](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/enterprise-application-patterns/mvvm)作为开发模式的首选，通过绑定加上数据映射来生成界面再自然不过。遗憾的是TabControl并没有提供在使用模板的情况下缓存控件的机制。

解决的办法不是没有。常见方案一是用第三方控件，比如著名开发商Telerik的[RadTabControl](https://docs.telerik.com/devtools/wpf/controls/radtabcontrol/howto/how-to-keep-content)就提供了一个`IsContentPreserved` 属性来实现这种行为。通过继承来改写TabControl的行为也是一个方案，实现不难，但问题是对于已有的那些从TabControl继承的控件没法重用（继承WPF的控件本身不是好的实践，我不推荐，但实践中常常发生）。

本文以下内容给出一种简单并且易于重用的方式来解决这个问题。

## 分析

假如用[Snoop](https://github.com/snoopwpf/snoopwpf)来查看第一种方式生成的[Visual Tree](https://docs.microsoft.com/en-us/dotnet/framework/wpf/advanced/trees-in-wpf)，你会发现当控件加载完成以后每个TabItem的Content属性就已经是XAML里写的控件了，但查看第二种方式的话TabItem的Content则是数据项，模板只会套用在当前激活的那个TabItem上。

很直观的想法是，在第二种方式下，如果能预先根据数据项相应的模板生成控件并设置给TabItem.Content，并使得TabControl跳过套用模板的过程直接显示控件，就能达到目的。事实也是如此。

当然使用方式也应该足够简单，能用在任何TabControl上，如下所示：

```html
<TabControl ItemsSource="{Binding ...}" TabContentPreservation.IsContentPreserved="True">
  ...
</TabControl>
```

## 解决方案
先写一个叫[`TabContentPreservation`](https://github.com/eagleboost/eagleboost/blob/master/eagleboost.presentation/Controls/TabContentPreservation.cs)的类，它只需要包含一个附加属性`IsContentPreserved`，在该附加属性的值变化回调函数里我们再创建`TabContentManager`类的一个实例来做具体的事。

**<u>注</u>**: 为简化讨论以下代码有删减，完整实现请移步[github](https://github.com/eagleboost/eagleboost/blob/master/eagleboost.presentation/Controls/TabContentPreservation.cs)。

```c#
private static void OnIsContentPreservedChanged(DependencyObject obj, DependencyPropertyChangedEventArgs e)
{
  var tc = obj as TabControl;
  if ((bool)e.NewValue)
  {
    var manager = new TabContentManager(tc);
    manager.Start();
    SetTabContentManager(tc, manager);
  }
}
```

`TabContentManager`首先侦听TabControl的`DataContextChanged`事件来做一些准备工作。

```c#
void HandleTabDataContextChanged(object sender, DependencyPropertyChangedEventArgs e)
{
  var tc = (TabControl)sender;
  var coll = (INotifyCollectionChanged)tc.Items;
  if (e.NewValue != null)
  {
    if (tc.Items.Count > 0)
    {
      ////Items属性已经有内容，也就是说TabItem.Content已经在XAML里有控件，同学你用错场景了
      var firstTab = tc.Items[0] as DependencyObject;
      if (firstTab != null)
      {
        throw new InvalidOperationException(string.Format("Content type of {0} is already preserved", tc.Items[0]));
      }
    }

    ////把TabControl的ContentTemplate保存下来并清除原来的引用，这样一来TabControl就跳过套用模板的过程。
    _contentTemplate = tc.ContentTemplate;
    tc.ContentTemplate = null;

    ////侦听SelectionChanged并对每个被激活的TabItem创建内容控件。
    tc.SelectionChanged += HandleTabSelectionChanged;

    ////也需要侦听CollectionChanged事件，当TabControl.Items中数据项发生删除/清除的时候清除数据项相应的缓存。
    coll.CollectionChanged += HandleDataItemCollectionChanged;
  }
}
```

尽管`TabControl.Items`中可能有多个数据项，但控件的创建可以延迟到当TabItem第一次被激活的时候。这样的话如果某些TabItem从来没有激活则无需耗费内存。

```c#
void HandleTabSelectionChanged(object sender, SelectionChangedEventArgs e)
{
  var tc = (TabControl)sender;

  if (e.AddedItems.Count > 0)
  {
    var dataItem = e.AddedItems[0];
    var tabItem = (TabItem)tc.ItemContainerGenerator.ContainerFromItem(dataItem);
    if (tabItem != null)
    {
      ////如果相应控件不存在则创建一次
      var contentControl = GetRealizedContent(dataItem);
      if (contentControl == null)
      {
        ////模仿TabControl的行为。TabControl每次重用同一个ContentPresenter来套用模板，我们为每个TabItem创建一个ContentControl来套用模板
        var template = _contentTemplate;
        if (template != null)
        {
          contentControl = new ContentControl
          {
            DataContext = dataItem,
            ContentTemplate = template,
            ContentTemplateSelector = tc.ContentTemplateSelector
          };

          contentControl.SetBinding(ContentControl.ContentProperty, new Binding());
          
          SetRealizedContent(dataItem, contentControl);
          ////延迟到TabControl完成其余操作后把TabItem.Content更新为缓存好的控件
          tc.Dispatcher.BeginInvoke((Action)(() => tabItem.Content = contentControl));
        }
      }
    }
  }
}
```
`CollectionChanged`事件处理很简单，当数据项发生删除或清除的时候把缓存的控件也清除，具体实现请移步[github](https://github.com/eagleboost/eagleboost/blob/master/eagleboost.presentation/Controls/TabContentPreservation.cs)。

## 结论

虽然[TabControl](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.tabcontrol?view=netframework-4.8)并未直接支持此行为，但也容易理解，毕竟大多数使用场景已经涵盖，更专业和更复杂的场景很大概率会使用第三方组件库。而第三方组件库提供的额外功能中大都支持在使用模板的情况下缓存TabItem的内容。好在`WPF`框架设计得足够灵活，使得我们很容易就能写出本文讨论的方法来解决相关问题。