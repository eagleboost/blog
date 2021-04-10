---
layout:     post
title:      "M-V-VM下实现数据项的选择（三）"
subtitle:   "WPF Best Practices"
date:       2020-04-19 16:48:00
author:     "eagleboost"
header-img: "img/post-bg-desktop.jpg"
header-mask:  0.3
catalog: true
tags:
    - 选择数据项
    - M-V-VM
    - MVVM
    - WPF高级编程
    - Advanced WPF
    - WPF Best Practices
    - WPF
    - .Net
---

### 1. RadioSelectionContainer应用

互斥单选是该系列[第一篇](https://eagleboost.com/2019/08/06/WPF-Best-Practices-M-V-VM%E4%B8%8B%E5%AE%9E%E7%8E%B0%E9%80%89%E6%8B%A9%E6%95%B0%E6%8D%AE%E9%A1%B9-%E4%B8%80/)提到的痛点之一，通常基于`RadioButton`的做法代码略显累赘。使用[上一篇](https://eagleboost.com/2019/08/25/WPF-Best-Practices-M-V-VM%E4%B8%8B%E5%AE%9E%E7%8E%B0%E9%80%89%E6%8B%A9%E6%95%B0%E6%8D%AE%E9%A1%B9-%E4%BA%8C/)提到的`RadioSelectionContainer`则可以非常优雅地实现。

假设有如下例子，`ListBox`根据`ItemsSource`生成多个互斥的`RadioButton`，上方的`TextBlock`显示当前选中的项。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/RadioSelectionContainer.png)

对应的`XAML`我们希望是这样：

```xml
<ListBox ItemsSource="{Binding Items}" ItemTemplate="{StaticResource RadioBoxTemplate}"/>
```

`ItemTemplate`如下。注意其中的`ContentControl`也可以直接用`RadioBoxContentTemplate`中的内容来替代，只需要略微调整`DataBinding`的路径。由于代码中标注了`DataTemplate`的`DataType`，使用`ContentControl`可以使得编辑器的`IntelliSense`能够提示相应类型的成员属性而不是显示找不到成员的波浪下划线。

```xml
<DataTemplate x:Key="RadioBoxTemplate" DataType="{x:Type sample:DataItem}">
  <Grid>
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="Auto"/>
      <ColumnDefinition Width="5"/>
      <ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
    <ContentControl Content="{Binding DataContext, RelativeSource={RelativeSource AncestorType=sample:SelectionContainerView}}" ContentTemplate="{StaticResource RadioBoxContentTemplate}" Tag="{Binding}"/>
    <TextBlock Grid.Column="2" Text="{Binding}"/>
  </Grid>
</DataTemplate>

<DataTemplate x:Key="RadioBoxContentTemplate" DataType="{x:Type sample:SelectionContainerViewModel}">
  <RadioButton>
    <i:Interaction.Behaviors>
      <SelectionContainerToggleButton SelectionContainer="{Binding RadioSelectionContainer}" DataItem="{Binding Tag, RelativeSource={RelativeSource AncestorType=ContentControl, AncestorLevel=2}}"/>
    </i:Interaction.Behaviors>
  </RadioButton>
</DataTemplate>
```

`SelectionContainerToggleButton`是一个面向`ToggleButton`的`Attached Behavior`，其功能是当`ToggleButton`的`Checked`和`Unchecked`事件发生时调用`SelectionContainer.Select`和`SelectionContainer.Unselect`方法来更新项目的选中状态，具体实现请移步[github](https://github.com/eagleboost/eagleboost/tree/master/eagleboost.presentation/Behaviors/SelectionContainer)。

也许看上去还是不简单，但这样的代码只需要写一次，重用时只需要涉及与`ListBox`相关的`XAML`。

### 2. MultipleSelectionContainer应用

跟上面的例子类似，多选可以用完全一样的方法来实现。代码如下，注意不通之处在于用`CheckBox`而不是`RadioButton`，还有在实际项目中`MultipleSelectionContainer`与前面的`RadioBoxContentTemplate`通常是完全一样的，此处为方便演示用了不同的`Binding Path`。

```xml
<ListBox ItemsSource="{Binding Items}" ItemTemplate="{StaticResource MultipleCheckBoxTemplate}"/>

<DataTemplate x:Key="MultipleCheckBoxTemplate" DataType="{x:Type sample:DataItem}">
  <Grid>
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="Auto"/>
      <ColumnDefinition Width="5"/>
      <ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
    <ContentControl Content="{Binding DataContext, RelativeSource={RelativeSource AncestorType=sample:SelectionContainerView}}" ContentTemplate="{StaticResource MultipleCheckBoxContentTemplate}" Tag="{Binding}"/>
    <TextBlock Grid.Column="2" Text="{Binding}"/>
  </Grid>
</DataTemplate>

<DataTemplate x:Key="MultipleCheckBoxContentTemplate" DataType="{x:Type sample:SelectionContainerViewModel}">
  <CheckBox>
    <i:Interaction.Behaviors>
      <SelectionContainerToggleButton SelectionContainer="{Binding MultipleSelectionContainer}" DataItem="{Binding Tag, RelativeSource={RelativeSource AncestorType=ContentControl, AncestorLevel=2}}"/>
    </i:Interaction.Behaviors>
  </CheckBox>
</DataTemplate>
```

### 3. 复杂应用

不论是桌面还是`Web`应用，一个常见的场景是有一系列待选项，其中一个可以被设置为默认。这里有两个要点：一是需要以某种方式告诉用户哪一项是默认项，二是要让用户能方便地设置任何一项为默认项。

第一点一般来说没有疑问，不外乎把默认项高亮。粗体也好，不同颜色也好，行末加上“默认”也好，各取所需。第二点则有多种实现或偏好。一种做法是在列表之外放一个按钮，点击它就能把当前选中的项设为默认值。这种方法最简单但也最偷懒，缺点是存在排版的问题，不论按钮放在何处，所在的行或列都会被留空而浪费掉。另一种方法是把设置默认值的按钮加到每一行的末尾。这解决了列表以外的空间浪费，但每行都有一个按钮显得过于丑陋。

下面给出使用`RadioSelectionContainer`的一种相对优化的方法以实现空间利用的最大化和空间浪费的最小化。

+ 在被设置为默认的项的末尾右对齐处显示一个“√”符号
  
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/RadioSelectionContainer_MouseOver1.png)

+ 当鼠标移动到任何一个非默认项时，在末尾右对齐处显示一个按钮，点击该按钮则把当前项设为默认。注意到由于互斥性的存在，对于已经被设置为默认的项当鼠标经过时无需改变。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/RadioSelectionContainer_MouseOver2.png)


具体实现跟前面的两个例子类似：

```xml
<!--HorizontalContentAlignment设置为Stretch以便“√”符号和按钮对齐到最右-->
<ListBox Grid.Row="2" ItemsSource="{Binding Items}" ItemTemplate="{StaticResource MouseOverRadioBoxTemplate}" HorizontalContentAlignment="Stretch" />

<DataTemplate x:Key="MouseOverRadioBoxTemplate" DataType="{x:Type sample:DataItem}">
  <Grid>
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="*"/>
      <ColumnDefinition Width="5"/>
      <ColumnDefinition Width="Auto"/>
    </Grid.ColumnDefinitions>
    <TextBlock Text="{Binding}"/>
    <ContentControl Grid.Column="2" Content="{Binding DataContext, RelativeSource={RelativeSource AncestorType=sample:SelectionContainerView}}" ContentTemplate="{StaticResource SelectCommandTemplate}" Tag="{Binding}"/>
  </Grid>
</DataTemplate>

<DataTemplate x:Key="SelectCommandTemplate" DataType="{x:Type sample:SelectionContainerViewModel}">
  <Grid>
    <CheckMark x:Name="SelectionBox" Height="14" Width="14" ToolTip="This is the default item">
      <i:Interaction.Behaviors>
        <SelectionContainerToggleButtonState SelectionContainer="{Binding RadioSelectionContainer}" DataItem="{Binding Tag, RelativeSource={RelativeSource AncestorType=ContentControl, AncestorLevel=2}}"/>
      </i:Interaction.Behaviors>
    </CheckMark>
    <buttons:AutoInvalidateButton 
      x:Name="Button" Height="16" Width="16" Padding="0,-1,0,0" Content="🟊"
      Command="{Binding RadioSelectionContainer.SelectCommand}" ToolTip="Set as default" CommandParameter="{Binding Tag, RelativeSource={RelativeSource AncestorType=ContentControl}}" Visibility="Collapsed"/>
  </Grid>
  <DataTemplate.Triggers>
    <MultiDataTrigger>
      <MultiDataTrigger.Conditions>
        <Condition Binding="{Binding IsMouseOver, RelativeSource={RelativeSource AncestorType=ListBoxItem}}" Value="True"/>
        <Condition Binding="{Binding IsChecked, ElementName=SelectionBox}" Value="False"/>
      </MultiDataTrigger.Conditions>
      <MultiDataTrigger.Setters>
        <Setter TargetName="Button" Property="Visibility" Value="Visible"/>
      </MultiDataTrigger.Setters>
    </MultiDataTrigger>
  </DataTemplate.Triggers>
</DataTemplate>
```

其中`CheckMark`是继承自`CheckBox`的只读控件，其`IsChecked`属性为真时显示“√”符号。`SelectionContainerToggleButtonState`负责把`CheckMark`的`IsChecked`属性绑定到`SelectionContainer`的索引器`[]`上——索引器的应用说明如下，摘录自[上一篇](https://eagleboost.com/2019/08/25/WPF-Best-Practices-M-V-VM%E4%B8%8B%E5%AE%9E%E7%8E%B0%E9%80%89%E6%8B%A9%E6%95%B0%E6%8D%AE%E9%A1%B9-%E4%BA%8C/)。

```c#
  public interface ISelectionContainer
  {
    /// <summary>
    /// 返回参数传入的item是否被选中。使用Indexer的目的在于方便数据绑定，如果用I是Selected(object item)之类的方法则无法支持绑定
    /// </summary>
    bool this[object item] { get; }
```

### 4. 结语

通过对`ISelectionContainer`接口的抽象，我们实现了单选，互斥单选，以及多选几种选择行为的统一，不仅易于单元测试和重用，也遵循了M-V-VM的最佳实践。

上述所有的代码的完整实现和实例请移步[github](https://github.com/eagleboost/eagleboost/tree/master/eagleboost.sampleapp/SelectionContainerSample)。
