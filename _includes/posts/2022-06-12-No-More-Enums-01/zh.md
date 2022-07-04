## 1. 问题

枚举类型`Enum`作为编程语言中的常用数据类型，为有限数量情况下的整数提供了别名支持，使用很简单，但隐含的问题其实不少——使用不当带来的性能损失对于大型桌面应用来说有时候会是一笔不大不小（但可以避免）的开销。

+ 枚举类型转换到字符串

写`WPF`应用程序的程序员大都碰到过这个问题——把一个枚举类型的所有可能值显示在一个列表中供用户选择，比如`ComboBox`或`ListBox`。有时候把相应枚举值的名称显示出来就行，但会导致`ToString()`方法的调用。当值的名称对用户不友好的时候通常的做法是给每个值加上一个`Attribute`来定义名称，再通过反射取出每个值对应的名称用于显示。枚举类型的`ToString()`方法以及反射都是低效的操作，如果是小型应用程序或者只是为了显示倒也问题不大，毕竟不是频繁发生。

在需要频繁调用的上下文中使用枚举类型则需要注意。比如为每条订单记录日志的时候使用枚举类型但是忘了调用`ToString()`会导致装箱，即便调用`ToString()`避免了装箱但是枚举类型的`ToString()`还是有额外的开销（打开`Enum`源码可以看到），更遭的是如果希望在日志中记录名称那么反射又不可避免。

+ 字符串转换到枚举类型

有转换到字符串的需求自然也有从字符串解析得到相应枚举类型的需求。字符串与值名称相同的情况下可以直接调用`Enum.Parse()`。在`.net Framework`中该方法会创建临时字符串，`.net Core`使用了`Span<char>`避免了在堆上创建临时字符串但从源代码可以看出代价还是不小。如果使用了`Attribute`定义名称，那`Enum.Parse()`就不再适用，通常的做法是把名称缓存起来，比如这样：

```c#
public static class EnumParser<T> where T : struct, Enum
{
  private static readonly Dictionary<string, T> StringToEnum;

  static EnumParser()
  {
    StringToEnum = new Dictionary<string, T>(StringComparer.OrdinalIgnoreCase);
    foreach (T value in Enum.GetValues(typeof(T)))
    {
      //GetName根据Attribute取回相应的名称，细节不再赘述。
      var name = GetName(value);
      StringToEnum.Add(name, value);
    }
  }
  
  public static T Parse(string value)
  {
    return StringToEnum[value];
  }
  
  public static bool TryParse(string value, out T result)
  {
    return StringToEnum.TryGetValue(value, out result);
  }
}
```

+ 类型转换。把一个枚举类型`Status`转换为整数类型很简单：

```c#
var intValue = (int)Status;
```

但这只适用于显式转换的场景。注意到枚举类型`Enum`实现了`IConvertible`接口，所以在一些泛型代码中可以这样写：

```c#
var intValue = Status.ToInt32(null);

////泛型转换
public static class EnumConvert<TIn> where TIn: Enum, IConvertible
{
  public static int From(TIn value)
  {
    return (int)value.ToInt32(null);
  }
}
```

结果相同但是使用`IConvertible`接口产生的隐含装箱操作常常容易被忽略：

```c#
int IConvertible.ToInt32(IFormatProvider? provider)
{
  //GetValue返回值是object
  return Convert.ToInt32(GetValue());
}
```

## 2. 那就装箱吧

避免值类型被装箱的一种办法是把值类型装箱，很拗口，看个例子就明白了。比如布尔类型`Boolean`就两个可能值，那么事先把它们装箱，然后在需要的地方（不是所有地方）用它们来代替从而避免频繁装箱。`WPF`里创建布尔类型的依赖属性`DependencyProperty`时就大量使用了这种技术。

```c#
public static class BooleanBox
{
  public static readonly object True = true;
  public static readonly object False = false;
  
  public static object Box(bool value)
  {
    return value ? True : False;
  }
  
  public static object Box(bool? value)
  {
    return value.HasValue ? Box(value.Value) : null;
  }
}
```

枚举类型由于可能值的数量有限，也可以使用类似的方式，比如这个最简陋的实现：

```c#
public static class EnumBox<T> where T : struct, Enum
{
  private static readonly Dictionary<T, object> BoxedValues = new();

  static EnumBox()
  {
    foreach (var value in Enum.GetValues(typeof(T)))
    {
      BoxedValues.Add((T)value, value);
    }
  }
  
  public static object Box(T value)
  {
    return BoxedValues[value];
  }
}
```
 
## 3. 还有问题

好，到目前为止这几个问题得到了解决：

| 问题                          | 解决方法|
| --------                     |:----- |
| 避免装箱                      | 使用`EnumBox<T>`或类似实现|
| 从名称获得枚举值               | 使用`EnumParser<T>`或类似实现|
| 转换到整数                   | 不使用`IConvertible`就没有装箱|

但再仔细一想，`EnumBox`用装箱避免了装箱，但一方面字典访问虽然快但仍旧有开销，另一方面拆箱也不快，而且为了实际上就是个整数的类型搞出几个辅助类总感觉有点多余。

枚举类型的基本使用场景其实只有别名一个，但真实世界的使用场景则更多，比如用户呈现和序列化，这两者都不可避免的导致装箱拆箱操作。此外还有一些延伸的高级场景则难于用传统的枚举类型实现，比如下面的例子来自某个订单系统：

| 场景        |描述|
| --------   |:----- |
| **查找与过滤**    | 在表格中某一列显示订单发起的国家。国家代码从定义来说是一个枚举类型，用户需要能够在列表头输入`CN`或者`China`来查找和过滤中国的订单。|
| **排序**    | 枚举类型的数据定义的顺序并不一定是显示的顺序。还是以国家为例，国家代码可以用一个整数代替，但排序的时候按照用户所见的字符串（比如国家名称）排列再正常不过。|
| **多种显示** | 还是以国家为例，国家代码可以用一个整数代替，但根据实际情况可能需要显示为`国家名称`/`缩写`/`国家代码`/`国旗`等。|
| **动态加载** | 尽管国家从定义上来说是个枚举类型，但是其数量较多，那么更好的做法应该是把数据存放在配置文件或者数据库中，在运行时动态加载到内存。|


于是会发现创建一个专门的数据类型来代替枚举是更好的选择——假设我们把这个新的枚举类型定义为一个类（而不是值类型），那么上述问题都很容易找到相应的解决办法。

## 4. TypeSafeEnum

基类定义了`Id`和`Name`。字符串类型的`Id`是唯一标识，与传统枚举类型基于整数不同，字符串具有更广泛的适用性，一来很多时候唯一标识并非数字，二来在需要序列化的场景下字符串可以一步到位。

```c#
public abstract class TypeSafeEnum
{
  protected TypeSafeEnum(string id, string name)
  {
    Id = id;
    Name = name;
  }
  
  public readonly string Id;
  
  public readonly string Name;
  
  public override string ToString()
  {
    return Name;
  }
}
```

接下来是泛型基类的定义，其核心功能由内嵌类`EnumCache`实现。

```c#
public abstract partial class TypeSafeEnum<T> : TypeSafeEnum where T : TypeSafeEnum<T>
{
  protected TypeSafeEnum(string name) : this(name, name)
  {
  }

  protected TypeSafeEnum(string id, string name) : base(id, name)
  {
    EnumCache.Add(id, (T)this);
  }
  
  ////根据Id或Name找到相应的实例T，失败时抛出异常
  public static T Parse(string id)
  {
    return EnumCache.Find(id);
  }

  ////根据Id或Name找到相应的实例T，失败时返回False
  public static bool TryParse(string id, out T result)
  {
    return EnumCache.TryFind(id, out result);
  }
  
  ////返回该类型T的所有实例
  public static IReadOnlyCollection<T> AllItems => EnumCache.Items;
}
```

核心功能`EnumCache`的简单实现如下。`EnumCache`一方面创建了一个字典（字典不是唯一选择，枚举值数量少的时候可以使用数组或列表达到更小的内存占用和更快的存取速度）用于保存`Id`和`Name`到相应实例的映射，另一方面其静态构造函数触发了类型T的静态构造函数以使得类型T的所有已定义的静态实例在`EnumCache`的任何方法被调用之前创建好。


```c#
public abstract partial class TypeSafeEnum<T> where T : TypeSafeEnum<T>
{
  private static class EnumCache
  {
    private static readonly Dictionary<string, T> Map = new(StringComparer.OrdinalIgnoreCase);
    private static readonly List<T> Values = new();

    static EnumCache()
    {
      ////触发类型T的静态构造函数并创建所有已定义的静态实例
      RuntimeHelpers.RunClassConstructor(typeof(T).TypeHandle);
    }
    
    public static IReadOnlyCollection<T> Items => Values;
      
    public static T Find(string id)
    {
      try
      {
        return Map[id];
      }
      catch (Exception e)
      {
        throw new ArgumentException($"Cannot parse {id} for '{typeof(T)}'");
      }
    }

    public static bool TryFind(string id, out T item)
    {
      item = FindCore(id);
      return item != null;
    }
    
    public static void Add(string id, T item)
    {
      Map.Add(id, item);
      Map.Add(item.Name, item);
      Values.Add(item);
    }

    private static T FindCore(string id)
    {
      return Map.TryGetValue(id, out var itemById) ? itemById : null;
    }
  }
}
```

有了泛型基类，以`Status`为例可以定义出`TypeSafeStatus`如下。

```c#
public sealed class TypeSafeStatus : TypeSafeEnum<TypeSafeStatus>
{
  public static readonly TypeSafeStatus New = new ("0", "New");
  public static readonly TypeSafeStatus Open = new ("1", "Open");
  public static readonly TypeSafeStatus Cancelled = new ("2", "Cancelled");
  
  private TypeSafeStatus(string id, string name) : base(id, name)
  {
  }
}
```

相应的单元测试如下：

```c#
[Test]
public void Test_01_ParseById()
{
  Assert.That(TypeSafeStatus.Parse("0"), Is.EqualTo(TypeSafeStatus.New));
  Assert.That(TypeSafeStatus.Parse("1"), Is.EqualTo(TypeSafeStatus.Open));
  Assert.That(TypeSafeStatus.Parse("2"), Is.EqualTo(TypeSafeStatus.Cancelled));
}

[Test]
public void Test_02_ParseByName()
{
  Assert.That(TypeSafeStatus.Parse("New"), Is.EqualTo(TypeSafeStatus.New));
  Assert.That(TypeSafeStatus.Parse("Open"), Is.EqualTo(TypeSafeStatus.Open));
  Assert.That(TypeSafeStatus.Parse("Cancelled"), Is.EqualTo(TypeSafeStatus.Cancelled));
}
```
