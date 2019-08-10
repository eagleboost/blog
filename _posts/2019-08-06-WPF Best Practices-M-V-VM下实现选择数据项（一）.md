---
layout:     post
title:      "M-V-VM下实现数据项的选择（一）"
subtitle:   "WPF Best Practices"
date:       2019-08-06 21:59:00
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

### 问题

用户界面由数据驱动是`M-V-VM`编程模式最佳实践的指标之一，其中数据项的选择同步尤为重要。数据项选择指的是`ViewModel`中的数据项通过数据绑定显示在界面上供用户选择，常见的场景包括互斥单选，单选及多选，选中的数据项在`ViewModel`和`View`之间应实现双向传递。

在`M-V-VM`的范畴内实现数据项的选择的方法很多。单选最简单，实现方法没有争议。但要正确地实现互斥单选和多选却不容易。本文通过对各种常见实现方式的缺陷分析，给出一种适用于所有场景的统一方式，该方式还可以扩展到更复杂的场景下使用。

### 基本场景

上述三种场景举例如下：

| 场景|描述|界面呈现|
| ----|---|----|
|单选|有0-N个数据项，任何时刻有0-1个被选中|通常使用从`Selector`继承下来的控件<br>比如`ComboBox`或设置为单选模式的`ListBox`，`ListView`，`DataGrid`等|
|互斥单选|有1-N个数据项，任何时刻都有一个被选中|互斥行为常见于`RadioButton`。<br>数据项数量不多时`RadioButton`可以在XAML中写死<br>数量多的时候可以通过`ItemsControl`来实现，需要重写ItemContainerStyle/DataTemplate|
|多选|有0-N个数据项，任何时刻可以有0-N个被选中|常见于设置为多选模式的`ListBox`，List`View`，`DataGrid`等|


+ 单选
  
`Selector`控件本身已经提供一个依赖属`SelectedItem`性用于数据绑定，`ViewModel`只需实现`INotifyPropertyChanged`接口并给出一个可读写的属性，比如SelectedPerson，并设置双向绑定即可，不再赘述。

```xml
<ComboBox SelectedItem="{Binding SelectedPerson, Mode=TwoWay}" ItemsSource="{Binding Persons}" />
<ListBox SelectedItem="{Binding SelectedPerson, Mode=TwoWay}" ItemsSource="{Binding Persons}" />
```

+ 互斥单选

`RadioButton`是一个特殊的`ToggleButton`，除了看起来不一样之外还加上了一个限制条件——`IsChecked`属性可以通过点击设置为`True`但不能通过*点击*自身设置为False，要设置`IsChecked`属性为False只能通过代码，数据绑定，或者点击设定了相同GroupName的另一个`RadioButton`来实现，也就是互斥行为。其实`RadioButton`本就是为互斥单选而设计。

使用Converter是[最常见的实现方法](https://www.wpftutorial.net/`RadioButton`.html)，即把`RadioButton`.`IsChecked`属性绑定到`ViewModel`对应的属性上，为每个`RadioButton`设置一个`IsChecked`为`True`时对应的值，Converter通过比较该值并转换以达到传递数据的效果。

```xml
<!-- 这里的OptionA, OptionB, OptionC是所有可能的值，Converter通过比较Option是否等于当前的ConverterParameter来觉得IsChecked是否为True -->
<RadioButton Content="A" GroupName="Options" 
             IsChecked="{Binding Path=Option, Mode=TwoWay, Converter={StaticResource OptionConverter}, ConverterParameter=OptionA}" />
<RadioButton Content="B" GroupName="Options" 
             IsChecked="{Binding Path=Option, Mode=TwoWay, Converter={StaticResource OptionConverter}, ConverterParameter=OptionB}" />
<RadioButton Content="C" GroupName="Options" 
             IsChecked="{Binding Path=Option, Mode=TwoWay, Converter={StaticResource OptionConverter}, ConverterParameter=OptionC}" />
```

互斥单选的场景下另一个常见问题（往严重了说是错误）是使用枚举类型，枚举类型不仅带来性能问题（装箱拆箱以及类型解析），也使得实现更为复杂（每个值需要显示为说明性文字）。

> `Enum`的完美替代方案本身是个非常有价值的话题，WPF Best Practices系列会用单独的一篇文章来讨论。

上面的方式一般说来问题不大，3到4个可选项还能接受，但是如果可选项变多则会比较讨厌，重复多条`RadioButton`的绑定让聪明的程序员看起来很傻不说，还容易出错。容易想到，可以用一个`ListBox`来自动生成多个条目，但如果在Google上搜索一下“wpf radiobutton binding”，排名最靠前的几个结果没有一个能称得上优化的。

+ 多选

多选的使用场景很多，比如在一个用`DataGrid`显示的订单系统里，选择多个订单并对每条订单执行相似的逻辑。我们需要能够通过代码在`ViewModel`中设置被选中的数据项，这些数据项在`DataGrid`中能呈现为选中的状态。而用户通过鼠标或键盘在`DataGrid`中选择的条目能够在`ViewModel`中反应出来，然后在执行相似逻辑的时候访问到这些被选择的数据项。

最常见的实现有两种方式。一是给数据项对应的对象添加`IsSelected`属性。因为`ListBox`Item也好，`DataGridRow`也好，这些`ItemsControl`为显示数据项所生成的界面对象都有`IsSelected`属性，只需要设置一下双向绑定即可。

```xml
<Style TargetType="{x:Type ListBoxItem}">
  <Setter Property="IsSelected" Value="{Binding IsSelected, Mode=TwoWay}" />
</Style>
```
如果你在Google上搜索“wpf listbox selecteditems binding”，[排名第一](https://stackoverflow.com/questions/11142976/how-to-support-listbox-selecteditems-binding-with-mvvm-in-a-navigable-applicatio)的结果来自[StackOverflow](https://stackoverflow.com)，以及[排名第二](https://www.markwithall.com/programming/2017/05/14/accessing-wpf-listbox-selecteditems-using-mvvm.html)的结果都与上述方式如出一辙。

看起来简单，运行一下也能用，遗憾的是这种方法有两个非常致命的缺陷。第一，需要数据项对象实现`IsSelected`属性并不可行。代码是你自己写的倒也罢了，如果来自第三方怎么办？更直接点，我只想显示几个字符串让用户选，为此专门实现一个类来包装字符串总觉得哪里不对。

好了，就算都是自己的代码，也给数据项对象加了`IsSelected`属性，考虑一下这个场景：在一个交易系统中有成千上万条订单，界面上有多个表格给用户显示订单用，用户甚至可以创建多个表格，每个表格设置不同的过滤条件以缩小范围，比如：

1) 表格一显示所有未成交的订单
   
2) 表格二显示所有来自客户A的订单

显然，表格一和表格二中的订单可能存在交集。假设一条订单在内存中只有一个实例（实际上一个设计良好的系统通常如此），那么当用户在表格一选中一条订单时，该订单的`IsSelected`属性会被设置为`True`，如果该订单在表格二中也存在，那么根据数据绑定该订单在表格二中也会呈现选中状态。大多数情况下在用户看来这会是一个错误（有意为之除外）。

也许你会说在不同的表格里为每个订单生成不同的实例不就行了？这样做带来的问题更多。

在股票交易系统这样的复杂系统中，订单数量通常非常庞大，多个实例会造成不必要的内存浪费，极端情况下用户开了N个表格同一个订单就有N个拷贝。内存浪费尚在其次，内存管理的挑战更多。比如订单上关联的股票市场数据收到来自服务商数据更新的时候，需要把更新也推送到每一个拷贝上，相应的界面也需要更新，这就需要维护从原始数据项到每个拷贝的关系，一不留神就会造成内存泄漏。

到目前为止我们还停留在讨论的层面，真正实现起来还会发现代码有臭味。原始对象和拷贝总该类型相同吧？如果类型相同那么原始数据上的`IsSelected`属性谁来用呢？似乎没有任何地方会用。如果对表格中显示的数据项给予新的类型，只有该类型才有`IsSelected`属性，看似解决了前一个问题，但这会造成其它麻烦。如果新类型从原始类型继承（假设原始类型允许继承），那么在使用中不可避免需要类型检查。如果不从原始类型继承又会有维护的麻烦，比如原始类型新添了一个属性，相应的在新类型里也要添加，如此这般不一而足……你也许已经看出来了，问题出在“给数据项添加`IsSelected`属性”这个想法上。

让我们停下来仔细想想什么是“选择数据项”。说一个数据项是否被选中，是指该数据项是否存在于某个指定的集合里。根据上下文的不同这样的集合可以有多个。比如上述“未成交的订单”中被用户选中的订单列表，就是这种集合的一个例子，一个数据项在这个列表中存在我们就说该数据项被选中，不存在就不被选中，从而同一个数据项可以同时在多个类似列表中存在，也就是同时在多个场景下被选中。所以给数据项添加`IsSelected`属性从根本上说是错误的。“被选中”并不是一个对象的属性，好比你被不同的好友拉进了多个聊天群，只是你的Id被加入到了聊天群的用户列表里，并不是你自己身上有“属于聊天群A”，“属于聊天群B”这样的属性。

更常见并且“可用”的是另一种实现。支持多选的`Selector`都有一个`SelectedItems`属性，通常情况下被注册为一个只读属性，如果尝试设置绑定会收到错误，这是合理的设置。`SelectedItems`作为一个集合，我们并不希望每次里面包含的数据项发生变化时就发出`PropertyChanged`通知`Selector`说`SelectedItems`变了，同时也不希望每次`Selector`中选中的数据项发生变化就生成一个新的集合传给`ViewModel`，况且`View`并不知道`ViewModel`需要什么类型的集合。

在`SelectedItem`s属性为只读的前提下，容易想到通过代码监听`ViewModel`和`View`两端所对应的`SelectedItems`集合的改变（`INotifyCollectionChanged`接口），在一方发生变化时把数据同步到另一方，从而实现类似单选绑定那样的双向数据传输。

下面是两个典型的例子，具体实现不再赘述。本文着重讨论这种方法存在的问题。

+ [WPF ListBox SelectedItems TwoWay binding](https://www.tyrrrz.me/Blog/WPF-ListBox-SelectedItems-TwoWay-binding)
+ [How to databind to selecteditems](http://blog.functionalfun.net/2009/02/how-to-databind-to-selecteditems.html)

首先这种方式从单选方式自然延申而来，也就局限于`ViewModel`提供一个实现了`INotifyCollectionChanged`接口的`SelectedItems`属性，否则无法监听改变。问题在于`INotifyCollectionChanged`接口本身并不是为“选择数据项”这种行为而设计，人们只是把它借过来勉强实现了选择数据项的同步。如果场景稍加变化就不适用了，比如在一个`ListBox`中某些特定的数据项显示不同的颜色，再比如在一个`TreeView`中某些节点需要显示不同的样式（更多例子后续会谈到）。

其次这种方式不够通用。实际上不论单选，互斥单选还是多选，本质上并没有不同，他们应当能够用统一的方式来实现。假如还沿用这种方式，单选模式下`INotifyCollectionChanged`会显得多余，而对于多选模式而言，假如我们希望`ViewModel`中在每一个数据项被选中或者取消选中的时候得到通知，使用`INotifyCollectionChanged`接口又会比较麻烦。

换言之，`INotifyCollectionChanged`作为响应列表改变而抽象化的接口，被人们借过来勉强实现了数据项多选的双向传输，但只能满足最基本的需求，不够用。

我们需要为“选择数据项”定义一套专用的统一接口，姑且称之为`ISelectionContainer`，接下来首先会讨论为该接口实现一组可重用的组件，把单选，互斥单选及多选通过统一的方式来实现，最后再推广到几个普通方法不容易实现的复杂场景。