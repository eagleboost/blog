---
layout:     post
title:      "可疑的ArgumentException"
date:       2024-10-13
author:     "eagleboost"
header-img: "img/post-bg-color-feather.jpg"
catalog: true
tags:
    - ArgumentException
    - INotifyCollectionChanged
    - ObservableCollection
    - API设计
    - .Net Framework
---

&emsp;&emsp;项目做得久了，千奇百怪的问题都会遇到。大多数问题一看就知道怎么回事，有些问题则很有迷惑性，甚至让人百思不得其解。

&emsp;&emsp;最近用户报告了一个错误，分析日志后发现往一个`ObservableCollection<T>`里面插入数据的时候抛出了`System.ArgumentException`，类似下面这样。看起来是内部出现了`InvalidCast`，问题是这里的`xxx`确实是`yyy`类型，不应该出现无效类型转换的异常。

```c#
System.ArgumentException: The value "xxx" is not of type "yyy" and cannot be used in this generic collection. (Parameter 'value')
```

&emsp;&emsp;如果`xxx`与`yyy`类型真的不兼容，那么用下面的代码很容易重现这个异常：

```c#
public interface IContract
{
}

public class Implementation : IContract
{
}

IList list = new ObservableCollection<IContract>(); //使用非泛型接口来调用
list.Add(new object()); //抛出System.ArgumentException
```

&emsp;&emsp;`ObservableCollection<T>`的非泛型`Add`方法来自`Collection<T>`，看起来完全没问题，那么问题出在哪里呢？

```c#
int IList.Add(object value)
{
  if (this.items.IsReadOnly)
    ThrowHelper.ThrowNotSupportedException(ExceptionResource.NotSupported_ReadOnlyCollection);
  ThrowHelper.IfNullAndNullsAreIllegalThenThrow<T>(value, ExceptionArgument.value);
  try
  {
    this.Add((T) value);
  }
  catch (InvalidCastException ex)
  {
    ThrowHelper.ThrowWrongValueTypeArgumentException(value, typeof (T));
  }
  return this.Count - 1;
}
```

&emsp;&emsp;好在生产环境出现的这个问题很容易重现，调试后发现有点意思。我们知道`ObservableCollection<T>`实现了`INotifyCollectionChanged`接口，在增删改数据的时候会触发事件，所以如果代码事件处理代码恰好抛出了`InvalidCastException`被上面的`IList.Add`方法捕捉到就会抛出令人迷惑的`ArgumentException`，比如下面的代码：

```c#
var coll = new ObservableCollection<IContract>();
coll.CollectionChanged += (s, e) => throw new InvalidCastException();
IList list = coll;
list.Add(new Implementation());
```

&emsp;&emsp;`Collection<T>`的这段`IList.Add`方法与`List<T>`的实现几乎完全一样。对`List<T>`来说没问题，但`Collection<T>`中存在可重载的方法，所以代码需要改进一下：即先进行类型转换，再调用`this.Add`。我会把代码这样写：

```c#
int IList.Add(object value)
{
  ...
  this.Add(CastValue());
  
  return this.Count - 1;

  T CastValue()
  {
    try
    {
      return (T) value;
    }
    catch (InvalidCastException ex)
    {
      ThrowHelper.ThrowWrongValueTypeArgumentException(value, typeof (T));
    }
  }
}
```
&emsp;&emsp;这个问题只在`.NetFramework`中出现，`.Net 6.0`以上版本已经有了[新的实现
](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Collections/ObjectModel/Collection.cs#L281)，与上面的代码类似。


```c#
int IList.Add(object? value)
{
    if (items.IsReadOnly)
    {
        ThrowHelper.ThrowNotSupportedException(ExceptionResource.NotSupported_ReadOnlyCollection);
    }
    ThrowHelper.IfNullAndNullsAreIllegalThenThrow<T>(value, ExceptionArgument.value);

    T? item = default;

    try
    {
        item = (T)value!;
    }
    catch (InvalidCastException)
    {
        ThrowHelper.ThrowWrongValueTypeArgumentException(value, typeof(T));
    }

    Add(item);

    return this.Count - 1;
}
```