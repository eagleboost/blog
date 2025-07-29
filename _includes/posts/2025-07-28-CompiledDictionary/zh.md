### Compiled Dictionary

&emsp;&emsp;最近读到一篇很棒的博客[Compiling a dictionary into a switch expression](https://tyrrrz.me/blog/expression-trees#compiling-a-dictionary-into-a-switch-expression)，作者提供了一个提高`Dictionary`查询速度的思路。简单来说是把字典里面每个`Key`的哈希值算出来，生成一个`switch`语句直接返回相应的值，这样就省去了字典查询过程中一些不必要的开销从而提高效率。伪代码如下：

```c#
public TValue Lookup(TKey key) => key.GetHashCode() switch
{
  // 没有哈希冲突，1对1返回响应值
  9254 => value1,
  -101 => value2,

  // 哈希值冲突，进而比较每个值本身
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

&emsp;&emsp;实测下来效果非常好，尤其是在`.Net Framework`中，速度比字典直接查询快了一倍还多。我们知道从`.Net Core`开始微软在性能上下了不少功夫，`.Net 9`中字典查询已经比`.Net Framework`快了不少，但应用上这个方法后性能仍旧有提升。下面是当字典中分别有`2000`和`4000`对键值时的性能对比：

`.Net Framework`测试结果
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryNet481.png)

`.Net9`测试结果
![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryNet9.png)

&emsp;&emsp;`Dictionary`的查询操作有三个：

|方法|是否抛异常|是否取值|测试名|
|----|--------|-------|---|
|`this[]`|是|是|Standard/Compiled dictionary|
|`TryGetValue`|否|是|Standard/Compiled TryGetValue|
|`ContainsKey`|否|否|Standard/Compiled ContainsKey|

&emsp;&emsp;这三个方法的逻辑其实是统一的，不同之处在于：

+ 默认行为不同，即查询失败时的操作不同。`this[]`会抛异常而且另外两个不会
+ 查询成功的行为不同。`this[]`直接返回值，`TryGetValue`不仅返回值还返回查询成功与否，`ContainsKey`则只返回查询成功与否

&emsp;&emsp;原博客只用`this[]`举例，我把代码重构了一下，让这三个方法共用同一套逻辑，如上性能测试的结果所示，性能提升幅度是一致的：

```c#
private SwitchExpression CreateSwitchBody(Expression keyParameter, Expression defaultCase, Func<TValue, Expression> switchCaseBodyFunc)
{
  // 计算Key哈希值的表达式
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

&emsp;&emsp;三个方法的实现都调用了`CreateSwitchBody`方法：


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
### 扩展思考

&emsp;&emsp;代码写完后我又想起一个著名的说法：当只有少数几个值的时候，直接用数组保存并轮训数组其实比字典更快。因此有时候我们会实现一些复杂的逻辑来根据数据量动态切换选择数组或者字典来存储。数组本身更省内存，如果更快的话何乐而不为？不过有了本文的方法，情况不一样了。

&emsp;&emsp;下面`.Net Framework`中数据量分别为`1-7`的情况下查询性能的对比。可以看到`5`个数据及以下确实是数组更快一些，超过`5`个时字典查询胜出。但是`Compiled Dictionary`性能则始终碾压另外两个。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryArrayLookupNet481.png)

&emsp;&emsp;`.Net9`中数组查询在`8`个及以下比字典快，但`Compiled Dictionary`仍然更快。

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/CompiledDictionaryArrayLookupNet9.png)

&emsp;&emsp;考虑到大多数情况下字典数据塞完之后几乎都是查询操作，所以无需频繁重新生成表达式，在这种情况下使用`Compiled Dictionary`可以带来就地性能提升，也用不着根据数据量动态切换来选择数组或者字典了，算得上一举两得。

&emsp;&emsp;具体实现请移步[github](https://github.com/eagleboost/CompiledDictionaryApp)