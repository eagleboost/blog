## 1. The problem

As one of the basic data types in a lot of programming language, `Enum` provides `Alias` capability for a set of integers with limited number of values, it's very easy to use but with some potential problems when it's not used properly. Such problems could cause performance issues especially in large scan desktop applications.

+ Converting Enum to string

One of the scenarios `WPF` developers would meet more or less is to display all possible values of certain type of `Enum` in a list control for user to choose, usually `ListBox` or `ComboBox`. Sometimes displaying the name of the value itself is enough, which would results in the calls to the `ToString()` method, but it's not always the case especially when the names of the `Enum` are not user friendly. The common solution is adding an `Attribute` to define display name for each of the value, then retrieve the display name from the `Attribute` for display. Reflection is not efficient operation, the `ToString()` call is also slow for `Enum`. It's not big deal for small size application when such case does not happen frequently.

However, extra attentions should be paid when `ToString()` or reflection is used very frequently. In an order management system for example, boxing would happen if we forget to call `ToString()` for `Enum` when writing order information to log, and even we do call `ToString()` to avoid boxing, the `ToString()` call itself is still very slow. Proof can be found in the source code of [`Enum` (.net Framework)](https://referencesource.microsoft.com/#mscorlib/system/enum.cs,e91b5f6f66834f75) or [`Enum` (.net Core)](https://source.dot.net/#System.Private.CoreLib/Enum.cs,b0c7d7f5b106cc60). Even worse if we want to write `Display Name` of the value to log then reflection would be needed.

+ Parsing string to Enum

When there's a need to convert `Enum` to string, there's also a need to convert string back to `Enum`, aka the `Parse` operation. When the string is the name of the value we can call `Enum.Parse()`, The `.net Framework` version of this method allocates temporary strings，the `.net Core` version, although uses `Span<char>` to avoid allocating temporary string on the Heap, but the cost is still not trivial according to the [source code](https://source.dot.net/#System.Private.CoreLib/Enum.cs,1cf2275e046f44e8). `Enum.Parse()` would not work if `Attribute` is used to define custom display name, common approach is to cache the names for reuse, like this:

```c#
public static class EnumParser<T> where T : struct, Enum
{
  private static readonly Dictionary<string, T> StringToEnum;

  static EnumParser()
  {
    StringToEnum = new Dictionary<string, T>(StringComparer.OrdinalIgnoreCase);
    foreach (T value in Enum.GetValues(typeof(T)))
    {
      //GetName returns the display name of the value from the Attribute, implementation details is ignored here
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

+ Type casting. Say we have a `Enum` type of `Status`, we can convert it to integer this way:

```c#
var intValue = (int)Status;
```

But it works only for explicit type casting. Notice that `Enum` type also implements `IConvertible`, so sometimes we write general code this way:

```c#
var intValue = Status.ToInt32(null);

////Generic convert
public static class EnumConvert<TIn> where TIn: Enum, IConvertible
{
  public static int From(TIn value)
  {
    return (int)value.ToInt32(null);
  }
}
```

The result is the same comparing to explicit direct casting but one fact usually being neglected by developers is the implicit boxing in the `IConvertible` implementations:

```c#
int IConvertible.ToInt32(IFormatProvider? provider)
{
  //Return type of the GetValue() method is System.Object
  return Convert.ToInt32(GetValue());
}
```

## 2. Box the Enum values

One way to avoid boxing operation is to box the values. For example `Boolean` has only two possible values, so we can box them and reuse the boxed values when needed (not everywhere) to avoid boxing. This approach has been used widely in `WPF` when creating `Boolean` type `Dependency Properties`.

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

Similar to `Boolean`, `Enum` also has limited number of possible values, so we can also box them and reuse later. Below is a simple implementation for demo purpose:

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
 
## 3. There're still problems

So far we have fixed these problems:

| Problem                          | Fix|
| --------                     |:----- |
| Avoid Boxing                      | Use`EnumBox<T>` or similar implementations|
| Parse Enum from string               | Use`EnumParser<T>` or similar implementations|
| Convert to integer                   | Avoid using `IConvertible`|

However, although boxing can be avoided by using `EnumBox<T>`, but on one hand dictionary lookup still has cost, on the other hand unboxing is not fast, and it feels strange to create helper classes like `EnumBox<T>` and `EnumParser<T>` just for a type that's basically an integer.

The basic scenario or the design purpose of `Enum` is actually just being alias, but there're far more use cases in real world especially in modern applications today. Except for display and serialization which cannot bypass boxing/unboxing, there're other advanced scenarios hard to implement by using the traditional `Enum`. Below examples are from some order management system:

| Scenario        |Description|
| --------   |:----- |
| **Searching and Filtering**    | One column in the grid is the country the order originated. The country code is an `Enum` by definition but users might want to be able to enter `CN` or `China` in the column header to search/filter orders from China.|
| **Sorting**    | The order of values defined in an `Enum` may not be the order they're defined. Take country as an example, the country code can just an an integer but the column usually should be sorted by the string presented to user (country short name or full name).|
| **Special Displaying** | Take country example again, we might want to show full name/short name/ISO code/flag in different places.|
| **Dynamic Loading** | Although country is `Enum` by definition, but since it has many values and barely used in `if` statement or `switch-case` statment, so instead of defining all values as static members of the Country class, its better store the values in some configuration file or data base and dynamically load them into memory at runtime.|

To implement all features listed above, creating a dedicated data type instead of using `Enum` maybe a better choice.

## 4. TypeSafeEnum

This base class `TypeSafeEnum` defines `Id` and `Name`, `Id` is the unique identity of a value of `TypeSafeEnum`, we choose string instead of integer as its data type because firstly most of the time identities are strings, secondly in the serialization case we could use string directly without conversion between string and integers back and forth.

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

Then we can define the generic class `TypeSafeEnum<T>`, its core functionalities are implemented by a nested class `EnumCache`.

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
  
  ////Returns corresponding instance of T based on Id or Name, it would throw exception when no value can be found.
  public static T Parse(string id)
  {
    return EnumCache.Find(id);
  }

  ////Returns corresponding instance of T based on Id or Name，return false when not found
  public static bool TryParse(string id, out T result)
  {
    return EnumCache.TryFind(id, out result);
  }
  
  ////Returns all possible values of the type T
  public static IReadOnlyCollection<T> AllItems => EnumCache.Items;
}
```

Below is a simple implementation of the core function `EnumCache`. Two things worth mention are first it created a dictionary (or an array/list when number of values is small for better lookup performance and smaller memory usage) to store the mapping from `Id` and `Name` to the corresponding value, second its static constructor triggers the static constructor of the type `T` so that all of the defined static instances of the type `T` would be created before any other method of `EnumCache` is called.


```c#
public abstract partial class TypeSafeEnum<T> where T : TypeSafeEnum<T>
{
  private static class EnumCache
  {
    private static readonly Dictionary<string, T> Map = new(StringComparer.OrdinalIgnoreCase);
    private static readonly List<T> Values = new();

    static EnumCache()
    {
      ////Trigger the static constructor of type T and create all instances of T
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

Then we can define a new type based on the generic base class:

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

Some unit test codes:

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