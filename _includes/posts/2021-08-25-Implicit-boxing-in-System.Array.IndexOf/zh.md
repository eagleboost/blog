## 1. 问题

上周在修复项目里一个性能问题时偶然发现一段使用了
`System.Array.IndexOf<T>()`的代码在`profiler`下显示有装箱操作，那段代码非常简单没有问题，那么`.net framework`的代码就成了首要嫌疑犯。

## 2. 分析

尽管`System.Array.IndexOf<T>()`看起来是泛型方法，但打开[源代码](https://referencesource.microsoft.com/#mscorlib/system/array.cs,35ebfdcba4f0e942)就会发现具体工作实际上是通过`EqualityComparer<T>.IndexOf()`来完成的。

```c#
public static int IndexOf<T>(T[] array, T value, int startIndex, int count) 
{
  ......
  return EqualityComparer<T>.Default.IndexOf(array, value, startIndex, count);
}
```

由于我们的数据类型是`char`，`EqualityComparer<char>.Default`会返回一个`GenericEqualityComparer<char>`的实例。

```c#
public abstract class EqualityComparer<T> : IEqualityComparer, IEqualityComparer<T>
{
  private static EqualityComparer<T> CreateComparer() 
  {
    ......
    // If T implements IEquatable<T> return a GenericEqualityComparer<T>
    if (typeof(IEquatable<T>).IsAssignableFrom(t)) {
        return an instance of GenericEqualityComparer<T>;
    }

    ......
    // Otherwise return an ObjectEqualityComparer<T>
    return new ObjectEqualityComparer<T>();
}
```

问题在于`GenericEqualityComparer<T>`在比较的时候会测试数据是否为空，当数据类型是值类型的时候装箱就发生了，如下所示：

```c#
internal class GenericEqualityComparer<T>: EqualityComparer<T> where T: IEquatable<T>
{
  internal override int IndexOf(T[] array, T value, int startIndex, int count) 
  {
    int endIndex = startIndex + count;
    if (value == null)  ////Boxing
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] == null) return i;
      }
    }
    else 
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] != null && array[i].Equals(value)) return i;
      }
    }
    return -1;
  }
```

从反编译后的IL代码也能看到`box`指令的身影：

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/System.Array.IndexOf.png)

网上搜索了一下类似的问题，找到一篇[Collections Without the Boxing](https://www.jacksondunstan.com/articles/5148)。文章描述的是一个`struct`由于没有实现`IEquatable<T>`接口导致`ObjectEqualityComparer<T>`被创建，作者阅读源代码后实现了`IEquatable<T>`接口使得`GenericEqualityComparer<T>`被创建，并认为问题解决了。

显然该作者并未意识到装箱操作的存在，更有趣的是如果我们打开[`ObjectEqualityComparer<T>`](
https://referencesource.microsoft.com/#mscorlib/system/collections/generic/equalitycomparer.cs,ac282b3e1817bb9b)的源代码会发现其`IndexOf`方法的实现跟`GenericEqualityComparer<T>`是一模一样的，也就是说他刚从坑里面爬出来转身又自己跳进去了还不自知。

```c#
internal class ObjectEqualityComparer<T>: EqualityComparer<T>
{
  internal override int IndexOf(T[] array, T value, int startIndex, int count) 
  {
    int endIndex = startIndex + count;
    if (value == null) ////Boxing
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] == null) return i;
      }
    }
    else 
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] != null && array[i].Equals(value)) return i;
      }
    }
    return -1;
  }
```

## 3. 解决方案

`EqualityComparer<T>.IndexOf()`的默认实现其实就已经我们所需要的。不过`EqualityComparer<T>`是个虚基类没法直接用，所以要么创建一个子类（但同时也需要实现其它`IndexOf`用不着的方法），要么直接复制这段代码吧。

```c#
public abstract class EqualityComparer<T> : IEqualityComparer, IEqualityComparer<T>
{
  internal virtual int IndexOf(T[] array, T value, int startIndex, int count) 
  {
    int endIndex = startIndex + count;
    for (int i = startIndex; i < endIndex; i++) 
    {
      if (Equals(array[i], value)) return i;
    }
    return -1;
  }
```

好在同样的问题在`.net Core`中不复存在，毕竟为了提升性能代码经过了大幅重写。