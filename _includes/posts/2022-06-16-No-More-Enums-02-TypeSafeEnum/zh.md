## 1. 性能测试

下面分别对从字符串解析，转换到字符串和访问所有元素三个方面对传统枚举类型，`EnumBox`和`EnumParser`辅助类和`TypeSafeEnum`三者作性能比较。

+ 从字符串解析 - 可以看出辅助类和`TypeSafeEnum`性能相当，毕竟查询操作远快于`Enum.Parse`。
  
```c#
public class Benchmark_Parse
{
  [GlobalSetup]
  public void Setup()
  {
    ////触发静态构造函数完成热身，下略
    RuntimeHelpers.RunClassConstructor(typeof(TypeSafeStatus).TypeHandle);
    RuntimeHelpers.RunClassConstructor(typeof(EnumParser<Status>).TypeHandle);
  }
  
  [Benchmark(Baseline = true)]
  [Arguments("New")]
  [Arguments("Open")]
  [Arguments("Cancelled")]
  public void Enum_Parse(string status)
  {
    Enum.Parse(typeof(Status), status);
  }
    
  [Benchmark]
  [Arguments("New")]
  [Arguments("Open")]
  [Arguments("Cancelled")]
  public void EnumBox_Parse(string status)
  {
    EnumParser<Status>.Parse(status);
  }
    
  [Benchmark]
  [Arguments("New")]
  [Arguments("Open")]
  [Arguments("Cancelled")]
  public void TypeSafeEnum_Parse(string status)
  {
    TypeSafeStatus.Parse(status);
  } 
}
```

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Benchmark-TypeSafeEnum-Parse.png)

+ 转换到字符串 - 辅助类只是省去了重复装箱，但`Enum.ToString()`本身很慢。`TypeSafeEnum`直接访问属性从而大幅胜出。
  
```c#
public class Benchmark_ToString
{
  [Benchmark(Baseline = true)]
  public void Enum_ToString()
  {
    var str = Status.New.ToString();
  }
    
  [Benchmark]
  public void EnumBox_ToString()
  {
    var str = EnumBox<Status>.Box(Status.New).ToString();
  }
    
  [Benchmark]
  public void TypeSafeEnum_ToString()
  {
    var str = TypeSafeStatus.New.ToString();
  } 
}
```

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Benchmark-TypeSafeEnum-ToString.png)

+ 访问所有元素 - `TypeSafeEnum`同样是直接访问属性大幅胜出。
  
```c#
public class Benchmark_All
{
  [Benchmark(Baseline = true)]
  public void Enum_All()
  {
    var items = Enum.GetValues<Status>();
  }
    
  [Benchmark]
  public void EnumBox_All()
  {
    var items = EnumParser<Status>.AllItems;
  }
    
  [Benchmark]
  public void TypeSafeEnum_All()
  {
    var items = TypeSafeStatus.AllItems;
  } 
}
```

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/Benchmark-TypeSafeEnum-All.png)


## 2. 使用场景举例

普通使用场景与传统枚举类型类似，如条件判断语句，如果需要在`switch-case`的场景中使用，略为改写`OrderStatus`类即可：

```c#
switch (status.Id)
{
  case OrderStatusId.New:
    ......
    break;
  case OrderStatusId.Open:
    ......
    break;
}

public sealed class OrderStatus : TypeSafeEnum<OrderStatus>
{
  public static readonly OrderStatus New = new (OrderStatusId.New, "New");
  public static readonly OrderStatus Open = new (OrderStatusId.Open, "Open");
  public static readonly OrderStatus Cancelled = new (OrderStatusId.Open, "Cancelled");
  
  private OrderStatus(string id, string name) : base(id, name)
  {
  }
}

public static class OrderStatusId
{
  public const string New = "0";
  public const string Open = "1";
  public const string Cancelled = "1";
}
```

更为有用也远比传统枚举类型高效的是在大量数据需要解析的场景。

举个例子，在一个股票交易系统中，每条交易订单都包含大量的属性，比如交易的价格，数量等，也包含交易状态`OrderStatus`，是买还是卖`Side`等等`枚举类型`。当这些交易订单保持在网络服务中时仅需保存各个属性对应的值而不是显示名称，比如保存`0`表示买单，`1`表示卖单等。当订单系统从网络服务读取这些数据时，得到的结果可能是这样：

```c#
55=IBM;11=636730640278898634;15=USD;38=7000;40=1;54=1;39=0;10000=UserId_123
```

对应的意义是：
| Tag/Value                          |意义|
| --------                     |:----- |
| 55=IBM               | 股票代码是IBM|
| 11=636730640278898634               | 订单唯一标识|
| 15=USD                   | 交易货币是美元|
| 38=7000                   | 交易数量是7000股|
| 40=1                   | 订单类型是当日有效|
| 54=1                   | 卖单|
| 39=0                   | 订单状态|
| 10000=UserId_123                  | 交易员标识UserId_123|

而从上述字符串解析后在内存中我们希望看到的是这样的类：

```c#
public class Order
{
  public Symbol Symbol { get; set; } //IBM
  public string OrderId { get; set; } //636730640278898634
  public Currency Currency { get; set; } //USD
  public double Quantity { get; set; } //7000
  public OrderType OrderType { get; set; } //Day
  public Side Side { get; set; } //Buy
  public OrderStatus OrderStatus { get; set; } //Open
  public Trader Trader { get; set; } //Michael Jordan (UserId_123)
}
```

如果把这里的`Currency`、`OrderType`等类型定义为传统枚举类型，那么解析的过程将大量调用非常耗时的`Enum.Parse`导致性能问题。

对于`TypeSafeEnum`则非常高效。我们可以为相应的类型定义`TypeConverter`用于数据类型的转换。`OrderStatusConverter`最终调用了`TypeSafeEnum<T>.Parse`方法，远比`Enum.Parse`高效。

```c#
[TypeConverter(typeof(OrderStatusConverter))]
public sealed class OrderStatus : TypeSafeEnum<OrderStatus>
{
  ......
}

public class OrderStatusConverter : TypeConverter
{
  public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
  {
    return sourceType == typeof(string);
  }

  public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
  {
    var str = (string)value;
    return OrderStatus.Parse(str);
  }
}

////示例代码，比如这里的属性是OrderStatus，stringValue是"0"，那么解析后value的值就是OrderStatus.New

if (typeof(TypeSafeEnum).IsAssignableFrom(p.PropertyType))
{
  var converter = TypeDescriptor.GetConverter(p.PropertyType);
  var value = converter.ConvertFromString(stringValue);
  ////把value设置给相应属性
  ......
}
```

## 3. 结论

相比传统枚举类型，`TypeSafeEnum`的设计所牺牲的内存带来的回报是巨大的。因而在实际工作中需要显示的场景，都可以用`TypeSafeEnum`来代替。

细心的读者可能也注意到这两篇博客并未提及枚举类型作为`Flag`使用的场景——尤其是需要快速标志位测试的时候，`TypeSafeEnum`同样增加一个属性来代表标志位达到类似的效果，具体实现不再赘述。

相关代码请移步[*github*](https://github.com/eagleboost/TypeSafeEnum)