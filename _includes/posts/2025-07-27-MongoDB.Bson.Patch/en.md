### Issue  

&emsp;&emsp;Our project has been using `MongoDB` to store information such as user settings. For many years, we relied on MongoDB's official `.NET Driver`, which involved creating a client instance with a username, password, and server address to establish a direct connection to the database. Over time, the codebase incorporated numerous custom `BSON Converters` to work with `MongoDB`'s `BsonSerializer` for serialization.  

&emsp;&emsp;Earlier this year, India market regulations demanded that usernames and passwords must not appear in the code. To comply, we had to modify the implementation. We introduced a `Node.js` middleware to relay messages between `MongoDB` and the clients, with the client sending an `Access Token` for `Kerberos` authentication instead.  

&emsp;&emsp;The data returned by the `Node.js` middleware to the client is in `JSON` format. Since rewriting all the previously configured `Converters` used with the `.NET Driver` would have been too costly, I made minor adjustments to reuse the existing serialization logic. After running smoothly in the test environment for some time, the changes were deployed to `Pilot`, where an issue surfaced.  

&emsp;&emsp;To simplify the problem: Suppose we have a class with a `double` property stored in `MongoDB`:  

```c#  
private class Model  
{  
  public double Value { get; set; }  
}  
```  
The `Value` property is almost always an integer, so the data in the database looks like this:  

```json  
{  
   "Value" : 100,  
}  
```  
&emsp;&emsp;The error observed in the logs was an `OverflowException`:  

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
&emsp;&emsp;The issue is reproducible. Upon examining the user data, it became clear that the `Value` in the database was an extremely large number. A simple test confirmed this:  

```c#  
const string json = """  
                    {  
                      "Value" : 12345678901234567890,  
                    }  
                    """;  
Assert.Throws<OverflowException>(() => BsonSerializer.Deserialize<Model>(json));  
```  
&emsp;&emsp;Previously, with a direct `MongoDB` connection, there was no issue—despite the large number, it was within the representable range of a `double`, and deserialization succeeded. My hypothesis is that in the direct connection mode, the server returns binary `BSON` data with type information, so `BsonSerializer` uses `BsonBinaryReader` instead of `JsonReader`. As a result, large numbers are parsed as `double` based on the type metadata in the binary stream.  

&emsp;&emsp;However, `JsonReader` processes `JSON` strings without type information and must infer the data type while parsing. When encountering a number like `12345678901234567890` without a decimal point, it assumes it's an integer. Since this number exceeds `Int64.MaxValue`, the following line calls `Int64.Parse` and throws an exception:  

```c#  
var type = JsonTokenType.Int64;  
...... // Character-by-character parsing to determine the type  
var str = buffer.GetSubstring(start, buffer.Position - start);  
if (type == JsonTokenType.Double) // type is Int64  
{  
  var value = JsonConvert.ToDouble(str);  
  return new DoubleJsonToken(str, value);  
}  
else  
{  
  var value = JsonConvert.ToInt64(str); // Throws OverflowException  
```  

### Solution  

&emsp;&emsp;Now that the problem is understood, how do we fix it?  

&emsp;&emsp;The large number is technically invalid for the use case—likely stored due to a bug (since there was no validation for unknown data ranges). A seemingly simple solution is to manually correct the invalid data in the database. However, such data may be widespread and scattered, making manual fixes impractical and hard to verify. Moreover, this would only be a temporary fix—future occurrences would still trigger exceptions. A code-level solution is necessary.  

&emsp;&emsp;Since the issue lies in an internal class `MongoDB.Bson.IO.JsonScanner`, custom `Converters` cannot intercept the parsing logic (the error occurs before `Converter` execution). Reviewing the `JsonReader` code revealed no extensibility points for custom logic. A conventional fix would require reimplementing `JsonReader` entirely to modify `JsonScanner`'s behavior, which is too costly. The latest `MongoDB Driver` also retains this logic, suggesting either no one has reported it or the official stance is that it's not a `bug` (or not worth fixing, as it stems from user error). Thus, runtime patching (`patching`) is the only viable option.  

&emsp;&emsp;The chosen approach uses [Harmony](https://github.com/pardeike/Harmony). I write a method `IsBadInt64String` to check if a given string can be safely converted to `Int64`. Then modify the `IL` instructions of `JsonScanner.GetNumberToken` to insert a call to `IsBadInt64String`, effectively changing the logic to:  

```c#  
// Even if type is not Double, treat as Double if str cannot be safely converted to Int64  
if (type == JsonTokenType.Double || IsBadInt64String(str))  
{  
  var value = JsonConvert.ToDouble(str);  
  return new DoubleJsonToken(str, value);  
}  
```  

&emsp;&emsp;The issue is now resolved. Of course, we should also investigate how such large numbers were stored in the first place—perhaps adding validation against `Int64.MaxValue`.  

&emsp;&emsp;The implementation details are omitted here, but the core reference code is below. For the full implementation, visit [GitHub](https://github.com/eagleboost/BsonInt64DoubleFix).  

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