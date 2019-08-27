---
layout:     post
title:      "M-V-VM下实现数据项的选择（二）"
subtitle:   "WPF Best Practices"
date:       2019-08-25 20:59:00
author:     "eagleboost"
header-img: "img/post-bg-mountain-night.jpg"
header-mask:  0.3
catalog: true
tags:
    - 选择数据项
    - M-V-VM
    - MVVM
    - WPF高级编程
    - WPF Best Practices
    - WPF
---

### 1. ISelectionContainer接口

先解释一下[上一篇](https://eagleboost.com/2019/08/06/WPF-Best-Practices-M-V-VM%E4%B8%8B%E5%AE%9E%E7%8E%B0%E9%80%89%E6%8B%A9%E6%95%B0%E6%8D%AE%E9%A1%B9-%E4%B8%80/)结尾定义的`ISelectionContainer`接口。

该接口提供了支持数据项选择的所有功能。`SelectedItems`很简单，用于返回当前选中的所有数据项。`Clear/Select/Unselect`几个方法用于改变选中列表，比如当界面控件的`SelectionChanged`事件发生时可以调用这几个方法把数据传递到`ViewModel`。而`ItemsCleared/ItemsSelected/ItemsUnselected`事件则用于在`ViewModel`改变选中的数据项触发事件通知界面控件更新选中状态。

需要重点提到的是索引器。代码中常常需要检查某个数据项是否被选中，传统方式有：

+ 调用扩展方法`Contains`检查`SelectedItems`。效率通常不高，因为`SelectedItems`的具体实现一般没有对索引做优化。
  
+ 提供`IsSelected(object item)`方法。此方法最常见，但我们都知道方法调用对于数据绑定并不友好。就算我们通过一些蹩脚的方式在`XAML`里面调用了`IsSelected`方法，如果选中状态发生了改变需要更新界面还是会头疼。

使用索引器的好处在于`WPF`的数据绑定对索引器有原生支持，所以只要能想办法创建数据绑定，`ViewModel`触发的`PropertyChanged`事件就能被界面代码接收到，界面更新便顺理成章实现了。

> 作为可选项，也可以对`ISelectionContainer`做一些修改，比如把几个事件分离到一个单独的接口使得接口的作用更细化和明确，也可以添加`Command`以便界面上的按钮或者菜单调用。

```c#
  /// <summary>
  /// ISelectionContainer - 非泛型接口便于UI调用
  /// </summary>
  public interface ISelectionContainer
  {
    /// <summary>
    /// 返回当前选中的所有数据项，具体代码可以选择实现INotifyCollectionChanged接口
    /// </summary>
    IEnumerable SelectedItems { get; }

    /// <summary>
    /// 返回参数传入的item是否被选中。使用Indexer的目的在于方便数据绑定，如果用I是Selected(object item)之类的方法则无法支持绑定
    /// </summary>
    bool this[object item] { get; }

    /// <summary>
    /// 清除当前选中的列表，执行过后SelectedItems应该为空
    /// </summary>
    void Clear();

    /// <summary>
    /// 把items加入选中的列表
    /// </summary>
    void Select(ICollection items);

    /// <summary>
    /// 把items从选中的列表中删除
    /// </summary>
    void Unselect(ICollection items);

    event EventHandler ItemsCleared;

    event EventHandler<ItemsSelectedEventArgs> ItemsSelected;

    event EventHandler<ItemsUnselectedEventArgs> ItemsUnselected;
  }

  /// <summary>
  /// ISelectionContainer - 泛型接口用于实现组件
  /// </summary>
  /// <typeparam name="T"></typeparam>
  public interface ISelectionContainer<T>
  {
    IReadOnlyCollection<T> SelectedItems { get; }

    bool this[T item] { get; }

    void Clear();

    void Select(ICollection<T> items);

    void Unselect(ICollection<T> items);
  }
```

### 2. 实现

接口定义清楚具体实现很简单。我们需要三个版本，分别对应单选，互斥单选和多选。

+ SingleSelectionContainer\<T\>
  
初始化时可以为空，也可以包含一个已选中的数据项。当`Select`被调用时新的数据项就替换已有的数据项，但`Unselect`被调用时则把已有的数据项删除。

+ RadioSelectionContainer\<T\>

初始化时一般不为空，少数情况为空的情况可以用来表示一个`Nullable`。也可以包含一个已选中的数据项。当`Select`被调用时新的数据项就替换已有的数据项，但`Unselect`被调用时则什么都不做。

+ MultipleSelectionContainer\<T\>

初始化时根据实际情况可以包含0到n个数据项。当`Select`被调用时新的数据项被添加到列表中，`Unselect`被调用时则从列表中移除数据项。选择数据项顺序无关时可以选用`HashSet`作为内部数据项容器的实现，需要保持顺序时可以用别的数据结构。

这样一来暴露给`UI`的接口不变，只需要`ViewModel`给出不同版本就能实现不同方式的数据项选择。具体代码不再赘述，请移步[github](https://github.com/eagleboost/eagleboost/tree/master/eagleboost.core/Collections)。

### 3. 应用

先来一个最常见的例子，即前文说到`SelectedItems`在`View`和`ViewModel`之间双向传递。完成之后`XAML`里面的绑定可以这样写。

```xml
<ListBox Grid.Row="1" ItemsSource="{Binding Items}" SelectionMode="Extended" SelectedItem="{Binding SelectedItem}">
  <i:Interaction.Behaviors>
    <SelectionContainerListBoxSync SelectionContainer="{Binding MultipleSelectionContainer}"/>
  </i:Interaction.Behaviors>
</ListBox>
```

`ListBox`本身是一个`Selector`，因此我们先实现一个针对`Selector`的通用基类，用来在`Selector`的`SelectionChanged`事件触发的时候把数据项传递到`SelectionContainer`。

```c#
public class SelectionContainerSync<T> : Behavior<T> where T : Selector
{
  #region Dependency Properties
  #region SelectionContainer
  public static readonly DependencyProperty SelectionContainerProperty = DependencyProperty.Register(
    "SelectionContainer", typeof(ISelectionContainer), typeof(SelectionContainerSync<T>), new PropertyMetadata(OnSelectionContainerChanged));

  public ISelectionContainer SelectionContainer
  {
    get { return (ISelectionContainer) GetValue(SelectionContainerProperty); }
    set { SetValue(SelectionContainerProperty, value); }
  }

  private static void OnSelectionContainerChanged(DependencyObject obj, Dependency`PropertyChanged`EventArgs e)
  {
    ((SelectionContainerSync<T>)obj).OnSelectionContainerChanged();
  }
  #endregion SelectionContainer
  #endregion Dependency Properties

  #region Overrides
  protected override void OnAttached()
  {
    base.OnAttached();

    var selector = AssociatedObject;
    selector.SelectionChanged += HandleSelectionChanged;
  }

  protected override void OnDetaching()
  {
    var selector = AssociatedObject;
    if (selector != null)
    {
      selector.SelectionChanged -= HandleSelectionChanged;
    }

    base.OnDetaching();
  }
  #endregion Overrides

  #region Virtuals
  protected virtual void OnSelectionContainerChanged()
  {
  }

  protected virtual void OnSelectorSelectionChanged(SelectionChangedEventArgs e)
  {
    var c = SelectionContainer;
    foreach (var removed in e.RemovedItems)
    {
      c.Unselect(removed);
    }

    foreach (var added in e.AddedItems)
    {
      c.Select(added);
    }
  }
  #endregion Virtuals

  #region Event Handlers
  private void HandleSelectionChanged(object sender, SelectionChangedEventArgs e)
  {
    var c = SelectionContainer;
    if (c == null)
    {
      Detach();
      return;
    }

    OnSelectorSelectionChanged(e);
  }
  #endregion Event Handlers
}
```

`SelectionContainerListBoxSync`则继承`SelectionContainerSync`并实现为`ListBox`的特例，用来监听`SelectionContainer`的三个事件，在事件触发时把`ViewModel`的数据项选择传递给`ListBox`。

同理针对`DataGrid`的实现代码几乎一模一样，也可以用一些技巧来减少代码重复，本文不再赘述。

```c#
public class SelectionContainerListBoxSync : SelectionContainerSync<ListBox>
{
  #region Overrides
  protected override void OnAttached()
  {
    base.OnAttached();

    InitializeSelection();
  }

  protected override void OnDetaching()
  {
    UnhookSelectionContainer();

    base.OnDetaching();
  }

  protected override void OnSelectionContainerChanged()
  {
    var c = SelectionContainer;
    if (c != null)
    {
      InitializeSelection();
      HookSelectionContainer();
    }
  }

  protected override void OnSelectorSelectionChanged(SelectionChangedEventArgs e)
  {
    UnhookSelectionContainer();

    base.OnSelectorSelectionChanged(e);

    HookSelectionContainer();
  }
  #endregion Overrides

  #region Private Methods
  private void HookSelectionContainer()
  {
    var c = SelectionContainer;
    if (c != null)
    {
      c.ItemsSelected += HandleItemsSelected;
      c.ItemsUnselected += HandleItemsUnselected;
      c.ItemsCleared += HandleItemsCleared;
    }
  }

  private void UnhookSelectionContainer()
  {
    var c = SelectionContainer;
    if (c != null)
    {
      c.ItemsSelected -= HandleItemsSelected;
      c.ItemsUnselected -= HandleItemsUnselected;
      c.ItemsCleared -= HandleItemsCleared;
    }
  }

  private void InitializeSelection()
  {
    var listBox = AssociatedObject;
    var c = SelectionContainer;
    if (listBox == null || c == null)
    {
      return;
    }

    if (listBox.SelectedItems.Count == 0)
    {
      foreach (var i in c.SelectedItems)
      {
        listBox.SelectedItems.Add(i);
      }
    }
  }
  #endregion Private Methods

  #region Event Handlers
  private void HandleItemsCleared(object sender, EventArgs e)
  {
    var listBox = AssociatedObject;
    if (listBox != null)
    {
      listBox.SelectedItems.Clear();
    }
  }

  private void HandleItemsSelected(object sender, ItemsSelectedEventArgs e)
  {
    var listBox = AssociatedObject;
    if (listBox != null)
    {
      foreach (var i in e.Items)
      {
        listBox.SelectedItems.Add(i);
      }
    }
  }

  private void HandleItemsUnselected(object sender, ItemsUnselectedEventArgs e)
  {
    var listBox = AssociatedObject;
    if (listBox != null)
    {
      foreach (var i in e.Items)
      {
        listBox.SelectedItems.Remove(i);
      }
    }
  }
  #endregion Event Handlers
}
```

下一篇文章将演示几个更高级的使用场景，比如用`ListBox`来显示`CheckBox`和`RadioBox`，以及一个使用`SelectionContainer`来简化界面设计的例子。