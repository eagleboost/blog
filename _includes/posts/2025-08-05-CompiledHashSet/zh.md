### Compiled HashSet

&emsp;&emsp;上一篇博客[Compiled Dictionary](https://eagleboost.com/2025/07/28/CompiledDictionary/)给出了一个基本实现，但其实遗留了一种情况没有处理。比如字典的键值是字符串类型，希望键值对大小写不敏感的时候我们会在字典的构造函数传入一个`StringComparer.OrdinalIgnoreCase`，它的`GetHashCode`方法会对同一个字符串的不同大小写形式返回相同的值。

&emsp;&emsp;因此获取哈希值的代码需要做出相应修改：

```c#
////Call TKey.GetHashCode()
Call(keyParameter, typeof(object).GetMethod(nameof(GetHashCode))!);

////Call IEqualityComparer<TKey>.GetHashCode(TKey)
Call(Constant(_comparer), typeof(IEqualityComparer<TKey>).GetMethod(nameof(GetHashCode)), keyParameter);
```

&emsp;&emsp;核心部分新的实现如下：

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

&emsp;&emsp;既然`Dictionary`可以如此优化，那么同样的方法自然可以应用到`HashSet`，代码如下。与`CompiledDictionary`如出一辙，不同之处仅在于`HashSet`是键值一体，只有一个`ContainsKey`方法需要实现。

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

`.Net Framework`测试结果：
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledHashSetNet481.png)


`.Net9`测试结果：
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledHashSetNet9.png)


&emsp;&emsp;同样在`.Net Framework`中速度提升幅度更大，但`.Net 9`中也有不小的提升。

|数量|.Net Framework速度提升|.Net 9速度提升|
|----|--|--|
|6|64.6%|44.1%|
|500|48.9%|32.2%|
|2000|37.2%|16.4%|

&emsp;&emsp;具体实现请移步[github](https://github.com/eagleboost/CompiledDictionaryApp)