### Compiled Dictionary  

&emsp;&emsp;I recently came across an excellent blog post [Compiling a dictionary into a switch expression](https://tyrrrz.me/blog/expression-trees#compiling-a-dictionary-into-a-switch-expression), where the author proposed an idea to improve the lookup speed of a `Dictionary`. In short, the approach involves precomputing the hash values of each `Key` in the dictionary and generating a `switch` statement to directly return the corresponding value. This eliminates some of the overhead in the dictionary lookup process, thereby improving efficiency. Here’s a simplified version of the pseudocode:  

```c#
public TValue Lookup(TKey key) => key.GetHashCode() switch
{
  // No hash collision, 1-to-1 mapping  
  9254 => value1,
  -101 => value2,

  // Hash collision, further comparison of the actual keys  
  777 => key switch
  {
      key3 => value3,
      key4 => value4
  },

  // ...

  // Not found  
  _ => throw new KeyNotFoundException(key.ToString())
};
```  

&emsp;&emsp;The actual test results were impressive, especially in `.Net Framework`, where the compiled dictionary was more than twice as fast as a standard dictionary lookup. We know that Microsoft has made significant performance improvements since `.Net Core`, and in `.Net 9`, dictionary lookups are already much faster than in `.Net Framework`. However, even in `.Net 9`, this method still provides a noticeable performance boost. Below are the benchmark results for dictionaries containing `2000` and `4000` key-value pairs, respectively:  

.Net Framework Test Results
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryNet481.png)  

.Net 9 Test Results
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryNet9.png)  

&emsp;&emsp;A `Dictionary` supports three main lookup operations:  

| Method | Throws Exception? | Retrieves Value? | Benchmark Name |  
|--------|------------------|------------------|----------------|  
| `this[]` | Yes | Yes | Standard/Compiled dictionary |  
| `TryGetValue` | No | Yes | Standard/Compiled TryGetValue |  
| `ContainsKey` | No | No | Standard/Compiled ContainsKey |  

&emsp;&emsp;The logic for these three methods is fundamentally the same, with the following differences:  

+ **Default behavior** (when the key is not found):  
  - `this[]` throws an exception, while the other two do not.  
+ **Behavior on successful lookup**:  
  - `this[]` directly returns the value.  
  - `TryGetValue` returns both the value and a boolean indicating success.  
  - `ContainsKey` only returns a boolean indicating whether the key exists.  

&emsp;&emsp;The original blog post only demonstrated the `this[]` approach. I refactored the code to let all three methods share the same underlying logic. As shown in the benchmark results, the performance improvements are consistent:  

```c#
private SwitchExpression CreateSwitchBody(Expression keyParameter, Expression defaultCase, Func<TValue, Expression> switchCaseBodyFunc)
{
  // Expression to compute the hash code of the key  
  var keyGetHashCodeCall = Call(keyParameter, typeof(object).GetMethod(nameof(GetHashCode))!);
  
  return Switch(keyGetHashCodeCall, defaultCase, null,
    inner
      .GroupBy(p => p.Key.GetHashCode())
      .Select(g =>
      {
        if (g.Count() == 1)
        {
          return CreateSwitchCase(g.Key, g.Single().Value);
        }

        return SwitchCase(
          Switch(
            keyParameter,
            defaultCase,
            null,
            g.Select(p => CreateSwitchCase(p.Key, p.Value))
          ),
          Constant(g.Key)
        );
      })
  );

  SwitchCase CreateSwitchCase(object key, TValue value) => SwitchCase(switchCaseBodyFunc(value), Constant(key));
}
```  

&emsp;&emsp;All three methods call `CreateSwitchBody`:  

```c#
private void CompileLookup()
{
  var keyParameter = Parameter(typeof(TKey));
  var defaultCase = ThrowException();
  var body = CreateSwitchBody(keyParameter, defaultCase, v => Constant(v, TypeValue));
  var lambda = Lambda<Func<TKey, TValue>>(body, keyParameter);
  var str = lambda.ToReadableString();
  _lookup = lambda.Compile();
  return;

  UnaryExpression ThrowException()
  {
    var keyToStringCall = Call(keyParameter, typeof(object).GetMethod(nameof(ToString))!);
    var exceptionCtor = typeof(KeyNotFoundException).GetConstructor([typeof(string)]);
    var unaryExpression = Throw(New(exceptionCtor!, keyToStringCall), TypeValue);
    return unaryExpression;
  }
}

private void CompileTryGetValue()
{
  var keyParameter = Parameter(typeof(TKey));
  var valueParameter = Parameter(TypeValue.MakeByRefType());
  var defaultCase = ReturnValueBlock(false, default!);
  var body = CreateSwitchBody(keyParameter, defaultCase, v => ReturnValueBlock(true, v));
  var lambda = Lambda<TryGetValueDelegate>(body, keyParameter, valueParameter);
  _tryGetValue = lambda.Compile();
  return;

  BlockExpression ReturnValueBlock(bool found, TValue value)
  {
    return Block(Assign(valueParameter, Constant(value, TypeValue)), Constant(found));
  }
}

private void CompileContainsKey()
{
  var keyParameter = Parameter(typeof(TKey));
  var defaultCase = ReturnValue(false);
  var body = CreateSwitchBody(keyParameter, defaultCase, _ => ReturnValue(true));
  var lambda = Lambda<Func<TKey, bool>>(body, keyParameter);
  _containsKey = lambda.Compile();
  return;

  Expression ReturnValue(bool found) => Constant(found, typeof(bool));
}
```  

### Further Thoughts  

&emsp;&emsp;After implementing this, I recalled a well-known optimization tip: when dealing with a small number of values, iterating over an array is actually faster than using a dictionary. This is why some implementations dynamically switch between arrays and dictionaries based on the data size—arrays are more memory-efficient and, in some cases, faster. However, with the compiled dictionary approach, the situation changes.  

&emsp;&emsp;Below are the lookup performance comparisons in `.Net Framework` for datasets ranging from `1` to `7` entries. The results show that for `5` or fewer entries, array iteration is indeed faster, but beyond that, the standard dictionary wins. However, the **compiled dictionary consistently outperforms both**.  

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryArrayLookupNet481.png)  

&emsp;&emsp;In `.Net 9`, array iteration remains faster for up to `8` entries, but the **compiled dictionary is still the fastest in all cases**.  

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryArrayLookupNet9.png)  

&emsp;&emsp;Since dictionaries are typically populated once and then repeatedly queried, the overhead of compiling the switch expression is negligible. This means we can achieve immediate performance gains without the need for dynamic switching between arrays and dictionaries—a win-win scenario.

For the full implementation, visit [GitHub](https://github.com/eagleboost/CompiledDictionaryApp).  