### 问题

&emsp;&emsp;项目中一直使用`MongoDB`来保存诸如用户设置等信息。以前用了挺多年的官方`.net Driver`，也就是通过用户名、密码以及服务器地址创建客户端实例来直联数据库。这些年下来代码中也引入了不少自定义的`BSON Converter`配合`MongoDB`的`BsonSerializer`处理序列化。

&emsp;&emsp;今年初某个自认为领先东大`20`年的大国市场要求代码中不能出现用户名和密码，于是被要求修改代码以便合规。我们引入了一个`node.js`中间件，负责在`MongoDB`和客户端之间转发消息，客户端则发送`Access Token`通过`Kerberos`进行身份认证。

&emsp;&emsp;客户端从和`node.js`中间件取回的数据是`JSON`，之前使用`Driver`的时候引入的各种配置以及`Converter`扔掉重写代价太大，我只做了一些小改动来重用已有的所有代码处理序列化。代码在测试环境跑了不少时间没问题后进了`Pilot`，然后一个问题冒了出来。

&emsp;&emsp;把问题简化一下大致是这样：假设有下面这样一个包含`double`类型属性的类被保存到`MongoDB`。

```c#
private class Model
{
  public double Value { get; set; }
}
```
这个属性`Value`的值几乎总是整数，在数据库中看到是类似这样：

```json
{
   "Value" : 100,
}
```
&emsp;&emsp;报错后在日志中看到是`OverflowException`：

```
System.OverflowException: Value was either too large or too small for an Int64.
   at System.Number.ThrowOverflowException[TInteger]()
   at System.Int64.Parse(String s)
   at MongoDB.Bson.IO.JsonConvert.ToInt64(String value)
   at MongoDB.Bson.IO.JsonScanner.GetNumberToken(JsonBuffer buffer, Int32 firstChar)
   at MongoDB.Bson.IO.JsonScanner.GetNextToken(JsonBuffer buffer)
   at MongoDB.Bson.IO.JsonReader.PopToken()
   at MongoDB.Bson.IO.JsonReader.ReadBsonType()
   at MongoDB.Bson.Serialization.BsonClassMapSerializer`1.DeserializeClass(BsonDeserializationContext context)
   at MongoDB.Bson.Serialization.BsonClassMapSerializer`1.Deserialize(BsonDeserializationContext context, BsonDeserializationArgs args)
   at MongoDB.Bson.Serialization.IBsonSerializerExtensions.Deserialize[TValue](IBsonSerializer`1 serializer, BsonDeserializationContext context)
   at MongoDB.Bson.Serialization.BsonSerializer.Deserialize[TNominalType](IBsonReader bsonReader, Action`1 configurator)
   at MongoDB.Bson.Serialization.BsonSerializer.Deserialize[TNominalType](String json, Action`1 configurator)
```
&emsp;&emsp;可以稳定重现，把用户数据拿过来一跑就发现数据库中`Value`的值是一个很大的数，写一行简单的测试代码即可验证。

```c#
const string json = """
                    {
                      "Value" : 12345678901234567890,
                    }
                    """;
Assert.Throws<OverflowException>(() => BsonSerializer.Deserialize<Model>(json));
```
&emsp;&emsp;以前直联`MongoDB`没有问题，数虽大，但仍在`double`可以表示的范围内，反序列化是成功的。我推测直联`MongoDB`方式下服务器返回的二进制数据流`BSON`带有格式信息，对二进制数据`BsonSerializer`会调用`BsonBinaryReader`而不是`JsonReader`来读取数据，这样一来大数在数据流中会根据类型消息被解析成`double`。

&emsp;&emsp;而`JsonReader`读取的`Json`字符串没有格式信息，只能边读数据边判断数据类型，读到逗号结束时上面的大数因为没有包含任何浮点数表示形式被认为是整数，但其表示的整数超出了`Int64.MaxValue`，于是下面代码的最后一行调用`Int64.Parse`抛了异常。

```c#
var type = JsonTokenType.Int64;
......//逐字读取并判断数据类型
var str = buffer.GetSubstring(start, buffer.Position - start);
if (type == JsonTokenType.Double) //type是Int64
{
  var value = JsonConvert.ToDouble(str);
  return new DoubleJsonToken(str, value);
}
else
{
  var value = JsonConvert.ToInt64(str); //抛出OverflowException
```

### 解决办法

&emsp;&emsp;问题差不多弄清楚，怎么解决呢？

&emsp;&emsp;这个大数其实对相关使用场景来说并不合法，也许是某个`bug`导致存进去了（因其数据范围未知没有`validation`）。一个看起来简单的办法是直接操作数据库改正非法数据，但类似的数据可能很多并且分散在不同的地方，所以修改数据难度其实非常大，而且很难验证是否改对；其次是治标不治本，以后再出现这样的问题还是会抛异常，问题仍然存在。还是得从代码层面解决问题。

&emsp;&emsp;由于问题出在一个内部类`MongoDB.Bson.IO.JsonScanner`里面，没法通过自定义的`Converter`接管数据解析（还没有执行到调用`converter`的那一步）。大致看了`JsonReader`的代码，也没发现可供插入自定义逻辑的空间，要从常规方式动手的话得从头到尾实现一个`JsonReader`才能触摸到`JsonScanner`那段抛异常的代码，代价过高。最新的`MongoDB Driver`里这段逻辑也没有变化，可见要么没人报告过要么官方认为这不是一个`bug`（毕竟是用户错误）或者不值得处理。那么就只能`patch`代码了。

&emsp;&emsp;我选择的方案是用[Harmony](https://github.com/pardeike/Harmony)。写一个方法`IsBadInt64String`来判断给定字符串是否可以转换为`Int64`，在运行时修改`JsonScanner.GetNumberToken`的`IL`指令插入对`IsBadInt64String`的调用，把上面的代码改成逻辑上等价于下面这样：

```c#
//即使type不是Double，如果str不能安全转换为Int64仍然按Double处理
if (type == JsonTokenType.Double || IsBadInt64String(str))
{
  var value = JsonConvert.ToDouble(str);
  return new DoubleJsonToken(str, value);
}
```

&emsp;&emsp;问题算是完美解决，当然还得找找那么大的数是怎么来的，至少加个针对`Int64.MaxValue`的验证吧……

&emsp;&emsp;具体过程不赘述，核心参考代码如下，具体实现请移步[github](https://github.com/eagleboost/BsonInt64DoubleFix)。

```c#
private static void AddIsBadInt64StringCheck(CodeMatcher codeMatcher, object subString, Label int64Label, ILGenerator il)
{
/*
  IL_0014: ldloc.0      // 'type'
  IL_0015: ldc.i4.s     10 // 0x0a  //JsonTokenType.Double
  IL_0017: beq.s        IL_0021     //isNotDoubleLabel
  IL_0019: ldloc.s, 7   // substring
  IL_001a: call         bool BsonInt64DoubleFix.JsonScannerFix/Impl::IsBadInt64String(string)
  IL_001f: br.s         IL_0022     //isBadInt64Label
  IL_0021: ldc.i4.1                 //isNotDoubleLabel
  IL_0022: stloc.2      // V_2      //isBadInt64Label

  IL_0023: ldloc.2      // V_2
  IL_0024: brfalse.s    IL_0039     //int64Label
*/      
  var isNotDoubleLabel = il.DefineLabel();
  var isBadInt64Label = il.DefineLabel();

  codeMatcher.InsertAndAdvance(
    Code(Beq_S, isNotDoubleLabel),
    Code(Ldloc_S, subString),
    Code(Call, typeof(Impl).GetMethod(nameof(IsBadInt64String), Static | NonPublic)),
    Code(Br_S, isBadInt64Label),
    Code(Ldc_I4_1).WithLabels(isNotDoubleLabel),
    Code(Stloc_2).WithLabels(isBadInt64Label),
    Code(Ldloc_2),
    Code(Brfalse_S, int64Label)
  );
}
```