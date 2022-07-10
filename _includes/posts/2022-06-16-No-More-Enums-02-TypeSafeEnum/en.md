## 1. Benchmark

Let's create some benchmark test codes to compare `traditional Enum`, `EnumBox`/`EnumParser` helper class and `TypeSafeEnum` from 3 aspect: `parsing from string`, `converting to string` and `accessing all values`:

+ Parsing from string - The helper class `EnumBox`/`EnumParser` and `TypeSafeEnum` have similar performance, after all dictionary lookup is way faster than `Enum.Parse`.
  
```c#
public class Benchmark_Parse
{
  [GlobalSetup]
  public void Setup()
  {
    ////Trigger static constructors to warm up
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

+ Converting to string - Helper classes only avoids repeated boxing, but `Enum.ToString()` is still a slow operation. `TypeSafeEnum` is the fastest as it directly access the `Name` property.
  
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

+ Accessing all possible values - `TypeSafeEnum` wins again for direct accessing of the AllItems property.
  
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


## 2. Use cases

Assume there's a `OrderStatus` class, using it in `if` statement is straightforward, for `switch-case` statement, we just need to make minor changes to the `OrderStatus` class:

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

////Used in switch-case statement
public static class OrderStatusId
{
  public const string New = "0";
  public const string Open = "1";
  public const string Cancelled = "1";
}
```

It's more powerful and efficient in scenarios that huge amount of data parsing is needed.

For instance, in a stock trading system, each order contains a lot of properties, apart from basic information symbol, price, quantity, it also has `Enum` types like order status, side etc. when the orders are stored in the database or some other network services, usually we only save the corresponding value of the properties instead of name, like `0` for Buy, `1` for Sale instead of `Buy`, `Sell`. When the orders are loaded to client side, we might get something like this:

```c#
55=IBM;11=636730640278898634;15=USD;38=7000;40=1;54=1;39=0;10000=UserId_123
```

The meanings of the Tag/Values are:

| Tag/Value                          |Meaning|
| --------                     |:----- |
| 55=IBM               | The symbol is IBM|
| 11=636730640278898634               | Unique identity of the order|
| 15=USD                   | The order is traded in USD|
| 38=7000                   | The order quantity is 7000 shares|
| 40=1                   | It's a day order|
| 54=1                   | It's a Buy order|
| 39=0                   | The order is in Open status|
| 10000=UserId_123                  | The id of the trader is UserId_123|

Once parsed we'd want to see something like this in the memory:

```c#
public class Order
{
  public Symbol Symbol { get; set; } //IBM

  public string OrderId { get; set; } //636730640278898634

  public Currency Currency { get; set; } //USD

  public double Quantity { get; set; } //7000 shares

  public OrderType OrderType { get; set; } //Day order

  public Side Side { get; set; } //Buy

  public OrderStatus OrderStatus { get; set; } //Open

  public Trader Trader { get; set; } //Michael Jordan (UserId_123)
}
```

If the `Currency`, `OrderType` etc are defined as traditional `Enum` type, then massive calls to the slow `Enum.Parse` may potentially cause performance issues.

`TypeSafeEnum` can solve the problem efficiently. We can add `TypeConverter` to it for data conversion, in the `OrderStatus` example `OrderStatusConverter` calls the fast `TypeSafeEnum<T>.Parse` method eventually.

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

////Demo code, if the property here is OrderStatus, stringValue is "0", then after parsing, value would be OrderStatus.New

if (typeof(TypeSafeEnum).IsAssignableFrom(p.PropertyType))
{
  var converter = TypeDescriptor.GetConverter(p.PropertyType);
  var value = converter.ConvertFromString(stringValue);
  ////Set value to the property p
  ......
}
```

## 3. Conclusion

`TypeSafeEnum` isn't free, apparently it uses a lot more memory (compare to an integer) than traditional `Enum`, but the return of investment is enormous, indeed any `Enum` related to display can be replaced with `TypeSafeEnum`.

You may have noticed that we didn't discuss the case that `Enum` is used as `Flag` for fast flag test, we can simply add a Flag property to `TypeSafeEnum` to achieve similar result.

Please visit [*github*] for the demo project (https://github.com/eagleboost/TypeSafeEnum)