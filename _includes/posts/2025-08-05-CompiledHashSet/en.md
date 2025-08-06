### Compiled HashSet

&emsp;&emsp;In the previous blog post "[Compiled Dictionary](https://eagleboost.com/2025/07/28/CompiledDictionary/)" a basic implementation was provided, but one scenario was not addressed. For example, when the dictionary's key is of string type and case-insensitive comparison is desired, we pass a `StringComparer.OrdinalIgnoreCase` in the dictionary's constructor. Its `GetHashCode` method returns the same value for different case forms of the same string.

&emsp;&emsp;Therefore, the code for obtaining the hash value needs to be modified accordingly:


```c#
////Call TKey.GetHashCode()
Call(keyParameter, typeof(object).GetMethod(nameof(GetHashCode))!);

////Call IEqualityComparer<TKey>.GetHashCode(TKey)
Call(Constant(_comparer), typeof(IEqualityComparer<TKey>).GetMethod(nameof(GetHashCode)), keyParameter);
```

&emsp;&emsp;The core part of the new implementation is as follows:

```c#
private SwitchExpression CreateSwitchBody(Expression keyParameter, Expression defaultCase, Func<TValue, Expression> switchCaseBodyFunc)
{
  // Expression that gets the key's hash code using the comparer
  var keyGetHashCodeCall = Call(Constant(_comparer), GetHashCodeMethod, keyParameter);
  
  return Switch(
    keyGetHashCodeCall, // switch condition
    defaultCase, // default case
    null, // use default comparer
    _inner // switch cases
      .GroupBy(p => _comparer.GetHashCode(p.Key))
      .Select(g =>
      {
        if (g.Count() == 1)
        {
          return CreateSwitchCase(g.Key, g.Single().Value);
        }

        return SwitchCase(
          Switch(
            keyParameter, // switch on the actual key
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

&emsp;&emsp;Since `Dictionary` can be optimized this way, the same method naturally applies to `HashSet`. The code is as follows. It is almost identical to `CompiledDictionary`, with the only difference being that `HashSet` only has value, requiring only the implementation of a `ContainsKey` method.

```c#
private void CompileContains()
{
  var itemParameter = Parameter(typeof(T));
  var defaultCase = ReturnValue(false);
  var body = CreateSwitchBody(itemParameter, defaultCase, _ => ReturnValue(true));
  var lambda = Lambda<Func<T, bool>>(body, itemParameter);
  _contains = lambda.Compile();
  return;
  
  Expression ReturnValue(bool found) => Constant(found, typeof(bool));
}

private SwitchExpression CreateSwitchBody(Expression itemParameter, Expression defaultCase, Func<T, Expression> switchCaseBodyFunc)
{
  // Expression that gets the item's hash code using the comparer
  var keyGetHashCodeCall = Call(Constant(_comparer), GetHashCodeMethod, itemParameter);
  
  return Switch(
    keyGetHashCodeCall, // switch condition
    defaultCase, // default case
    null, // use default comparer
    _inner // switch cases
      .GroupBy(p => _comparer.GetHashCode(p))
      .Select(g =>
      {
        if (g.Count() == 1)
        {
          return CreateSwitchCase(g.Key, g.Single());
        }

        return SwitchCase(
          Switch(
            itemParameter, // switch on the actual key
            defaultCase,
            null,
            g.Select(p => CreateSwitchCase(p, p))
          ),
          Constant(g.Key)
        );
      })
  );

  SwitchCase CreateSwitchCase(object key, T value) => SwitchCase(switchCaseBodyFunc(value), Constant(key));
}
```

The benchmark test results in .NET Framework are as follows:
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledHashSetNet481.png)


The test results in .NET 9 are as follows:
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledHashSetNet9.png)


&emsp;&emsp;Similarly, the performance improvement is more significant in .NET Framework, but there is still a notable improvement in .NET 9.

|Count|.NET Framework Speed Increase|.NET 9 Speed Increase|
|----|--|--|
|6|64.6%|44.1%|
|500|48.9%|32.2%|
|2000|37.2%|16.4%|

&emsp;&emsp;For the specific implementation, please visit[github](https://github.com/eagleboost/CompiledDictionaryApp)