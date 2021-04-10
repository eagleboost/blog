---
layout:     post
title:      "EnabledStateBinding"
subtitle:   "WPF Best Practices"
date:       2020-06-11 22:17:00
author:     "eagleboost"
header-img: "img/post-bg-frog-woods.jpg"
header-mask:  0.3
catalog: true
tags:
    - IsEnabled
    - DataBinding
    - MVVM
    - M-V-VM
    - WPF高级编程
    - Advanced WPF
    - WPF Best Practices
    - WPF
    - .Net
---

### 1. 问题

`UI`编程中的一个常见场景是根据上下文把某些控件设置为不可用状态，比如这个并不恰当的例子。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/EnabledStateBinding.png)

在M-V-VM的语境下最简单的实现方法是在`ViewModel`中添加一个与所绑定的值属性对应的属性，比如`TextBox`的`Text`属性绑定到`ViewModel`的`Name`属性，那么`TextBox`的`IsEnabled`属性就绑定到`ViewModel`的`IsNameEnabled`属性，如下所示：

```xml
<Label Content="_Name" Target="{Binding ElementName=NameBox}"/>
<TextBox x:Name="NameBox" Text="{Binding Name}" IsEnabled="{Binding IsNameEnabled}"/>
```

有追求的开发者也许已经觉得`IsNameEnabled`属性有点累赘，但到目前为止没问题，实际上`99%`的代码都是这么做的，增加一个属性太正常不过。

但真实世界的系统比如客户关系管理系统`CRM`，财会系统，金融交易系统等等通常有较多输入和权限控制的系统，在一些更复杂的界面下，属性数量翻倍会头疼不说，复制粘贴容易出错也会带来维护上的问题。

从追求简洁和效率的角度出发，本文试图给出一个通用，简单，容易扩展的解决方案。


### 2. 分析

首先明确我们的目标是去掉`IsNameEnabled`这种属性。由于“`XXXX is enabled`”本身已经是一个约定（`contract`），所以只要能访问`XXXX`就能相应创建出`IsXXXXEnabled`。因此问题就转化为如何访问和存储`IsXXXXEnabled`的值。假设我们有这样一个接口：

```c#
public interface IItemEnabledStateStore
{
  ////当前可用的所有ItemEnabledState
  IReadOnlyCollection<ItemEnabledState> Items { get; }
  
  ////用于数据绑定
  ItemEnabledState this[string name] { get; }

  ////代码中设置属性可用/不可用状态
  void Update(string name, bool isEnabled, string reason = null);
}
```

同时如果存在一个自定义的`IsEnabledBinding`，就能够这样来用：

```xml
<TextBox Text="{Binding Name}" IsEnabled="{local:IsEnabledBinding Name}"/>
```

尽管`xaml`的代码并没有减少，但是`ViewModel`的属性少了一个。

容易发现一旦往这个方向发展，开发扩展功能也变得轻而易举，具体不再展开，只给出两个可能的例子：

* 当类似`TextBox`的编辑项很多时，可用写代码（如`Attached Behavior`）来动态生成所有绑定——比如遍历一遍根控件`Visual Tree`中的元素，为已经存在`Text`属性绑定的控件自动创建`IsEnabled`绑定。

* 把值绑定和`IsEnabled`绑定合二为一。如下，`BindingEx`在把`Text`属性绑定到`Name`的同时也为`IsEnabled`属性生成相应绑定。这种方式的问题在于获得方便性的同时牺牲了灵活性，扩展的功能不管用得到用不到都会包含在里面，如果通过属性的方式来`打开/关闭`某些扩展功能又会使得`XAML`复杂化，有时候并不灵活。

```xml
<TextBox Text="{local:BindingEx Name}"/>
```

### 3. ItemEnabledState

`ItemEnabledState`用于存放需要控制失效状态的项：

```c#
public class ItemEnabledState : NotifyPropertyChangedBase
{
  ////默认状态IsEnabled = true
  public static readonly ItemEnabledState DefaultState = new ItemEnabledState {IsEnabled = true};
    
  private bool _isEnabled;
  private string _reason;
  public readonly string Name;

  private ItemEnabledState()
  {
  }
  
  public ItemEnabledState(string name, bool isEnabled = false, string reason = null)
  {
    Name = name;
    IsEnabled = isEnabled;
    Reason = reason;
  }
  
  ////用于绑定到控件的IsEnabled属性
  public bool IsEnabled
  {
    get => _isEnabled;
    set => SetValue(ref _isEnabled, value);
  }
  
  ////用于绑定到控件的ToolTip属性
  public string Reason
  {
    get => _reason;
    set => SetValue(ref _reason, value);
  }

  public override string ToString()
  {
    return Name + " " + (_isEnabled ? "is enabled" : "is disabled") + (_reason != null ? ", " + _reason : null);
  }
}
```

### 4. ItemEnabledStateStore

只考虑在`GUI`线程使用的场景，`ItemEnabledStateStore`的一个简单实现如下：

```c#
[DebuggerDisplay("{Items.Count}")]
public class ItemEnabledStateStore : NotifyPropertyChangedBase, IItemEnabledStateStore
{
  private readonly Dictionary<string, ItemEnabledState> _items = new Dictionary<string, ItemEnabledState>();

  public IReadOnlyCollection<ItemEnabledState> Items => _items.Values;

  public ItemEnabledState this[string name] => GetOrDefault(name);

  public void Update(string name, bool isEnabled, string reason = null)
  {
    var state = GetOrCreate(name);
    state.IsEnabled = isEnabled;
    state.Reason = reason;
    ////通知名为"Item[]"的属性变化以便于通过索引器绑定的所有属性收到更新通知
    RaisePropertyChanged("Item[]");
  }

  private ItemEnabledState GetOrDefault(string name)
  {
    return _items.TryGetValue(name, out var state) ? state : ItemEnabledState.DefaultState;
  }
  
  private ItemEnabledState GetOrCreate(string name)
  {
    if (_items.TryGetValue(name, out var state))
    {
      return state;
    }

    return _items[name] = new ItemEnabledState(name);
  }  
}
```

### 5. IsEnabledBinding

众所周知`WPF`的[*BindingBase.ProvideValue*](https://referencesource.microsoft.com/#PresentationFramework/src/Framework/System/Windows/Data/BindingBase.cs,af362b96f5085ec8)被标记为[*sealed*](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/sealed)，所以不能通过继承`BindingBase`来实现自定义的绑定。这里我们使用一个基于[*BindingDecoratorBase*](http://www.hardcodet.net/2008/04/wpf-custom-binding-class)的[*MarkupExtension*](https://docs.microsoft.com/en-us/dotnet/api/system.windows.markup.markupextension?view=netcore-3.1)来实现自定义的`IsEnabledBinding`，效果不变。

实现的关键在于创建类似于如下绑定，也就是发现一个`ItemEnabledStateStore`的实例，通过索引器传入需要绑定的数据项的`Id`。

```xml
<TextBox IsEnabled="{Binding ItemEnabledStateStore[Name].IsEnabled}"/>
```

发现`ItemEnabledStateStore`实例的方法很多，此处为简单起见假设根控件的`DataContext`，即`ViewModel`有一个名为`ItemEnabledStateStore`的属性，把其余的工作交给`Binding System`利用反射机制去完成。也可以检查`ViewModel`是否实现某个接口并访问相应属性来做出更确定性的判断，不再赘述。

```c#
public class IsEnabledBinding : BindingDecoratorBase
{
  public IsEnabledBinding(string id)
  {
    Id = id;
  }

  public string Id { get; set; }
  
  public override object ProvideValue(IServiceProvider provider)
  {
    ////绑定当前属性到ItemEnabledState的IsEnabled属性
    var pathPrefix = "ItemEnabledStateStore[" + Id + "].";
    Binding.Path = new PropertyPath(pathPrefix + "IsEnabled");
    
    if (TryGetTargetItems(provider, out var obj, out var dp))
    {
      var element = (FrameworkElement)obj;
      var toolTipBinding = new Binding(pathPrefix + "Reason");
      ////绑定到ToolTip到ItemEnabledState的Reason属性
      element.SetBinding(FrameworkElement.ToolTipProperty, toolTipBinding);
      ////设置控件不可用时也显示ToolTip
      ToolTipService.SetShowOnDisabled(element, true);
    }
    
    return base.ProvideValue(provider);
  }
}
```

### 6. 使用

假设`ViewModel`实现如下：

```c#
public class ViewModel
{
  public IItemEnabledStateStore ItemEnabledStateStore { get; } = new ItemEnabledStateStore();

  public string Name { get; set; }
  
  public string Address { get; set; }
}
```

在运行时通过上下文调用`ItemEnabledStateStore.Update`方法即可控制相应的控件的使能状态。

```c#
var vm = new ViewModel {Name = "eagleboost"};
vm.ItemEnabledStateStore.Update("Name", false, "Name is not allowed to change");
//6.0及以后的c#版本可以使用nameof表达式
vm.ItemEnabledStateStore.Update(nameof(vm.Name), false, "Name is not allowed to change");
```

在较低版本的`c#`中如果想避免直接使用字符串带来的潜在错误，也可以把`ItemEnabledStateStore`实现为泛型，添加一个基于`Linq`表达式的参数，就能够获得编译时的正确性保证。

```c#
public void Update(Expression<Func<T, object>> property, bool isEnabled, string reason = null)
{
  ......
}

public class ViewModel
{
  public ItemEnabledStateStore<ViewModel> ItemEnabledStateStore { get; } = new ItemEnabledStateStore<ViewModel>();
}

////使用表达式来获取Name属性的名称
vm.ItemEnabledStateStore.Update(o=>o.Name, false, "Name is not allowed to change");
```

上述所有的代码的完整实现和实例请移步[*github*](https://github.com/eagleboost/ItemEnabledStateApp)。
