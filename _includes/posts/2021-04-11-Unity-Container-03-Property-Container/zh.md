## 1. 问题

[前文](https://eagleboost.com/2021/04/06/Unity-Container-(2)-Implementing-the-INotifyPropertyChanged-Interface/)在解决自动实现`INotifyPropertyChanged`接口问题之外还提出了几个遗留问题：

1. 调试不方便
2. 属性过多时监听事件带来不必要的性能损失
3. 不易获取属性之前的值

这三个问题归纳起来其实是一个问题，那就是需要提供属性而不是对象层面的处理——如果把一个属性的`Backing field`升级为一个对象，这几个问题就将迎刃而解。

先定义如下接口来代替一个类型为`T`的属性的`Backing field`：

```c#
public interface IProperty<T>
{
  string Name { get; }

  T Value { get; set; }

  event EventHandler<ValueChangedArgs<T>> ValueChanged;
}
```

再生成代码把下面的`ViewModel`展开为`ViewModel_<PropertyContainer>_Impl`。其中`PropertyStore`负责根据属性名创建并缓存`IProperty<T>`的实例。

```c#
//原始类
public class ViewModel
{
  public virtual string Name { get; set; }
}
  
//动态生成的类
public class ViewModel_<PropertyContainer>_Impl : ViewModel
{
  private PropertyStore<ViewModel_<PropertyContainer>_Impl> _store;
  private IProperty<string> _name;

  public ViewModel_<PropertyContainer>_Impl()
  {
    _store = new PropertyStore<ViewModel_<PropertyContainer>_Impl>(this);
    _name = _store.Create<String>("Name");
  }
    
  public IPropertyStore PropertyStore => _store;
  
  public override string Name
  { 
    get => _name.Value; 
    set => _name.Value = value; 
  }
}
```

解决上面的几个问题现在我们可以这样：
1. 调试不方便——在`IProperty<T>`实现类`Value`属性的`Setter`里设置断点。
2. 属性过多时监听事件带来不必要的性能损失——根据属性名获得相应`IProperty<T>`的实例并直接监听其`ValueChanged`事件。
3. 不易获取属性之前的值——`ValueChanged`事件的`EventArgs`已经包含属性的之前值和当前值。

当然这样做的代价是内存开销变大，但首先来说没有银弹，不存在可以用来解决一切问题的方法，我们总是需要具体情况具体分析。这种方法适用的对象并不是那种有几十上百个属性同时还有成千上万甚至几十上百万个实例的类。这种方法更适用于编写`ViewModel`——即便是超大型应用程序也不大可能有成千上万个`ViewModel`，这种情况下额外的内存开销完全可以接受。

## 2. 实现

使用`Reflection.Emit`生成代码的过程比较繁复，仅把主干陈列如下，大体工作分为下面几步：

1. 创建一个类继承自`BaseType`并实现几个接口：
   + `IPropertyContainer`——用于扩展方法从对象获取其隐式生成的`IPropertyStore`实例。
   + `INotifyPropertyChanged`——生成的类需要实现该接口，保持兼容性。
   + `IPropertyChangeNotifiable`——用于当`IProperty<T>.Value`发生改变时也在类上触发`PropertyChanged`事件。

2. 实现`INotifyPropertyChanged`接口
   + 生成`PropertyChanged`事件及其私有成员变量。
   + 生成`NotifyPropertyChanged`方法来触发`PropertyChanged`事件。
  
3. 为`BaseType`所有`virtual`的公开自动属性生成代码：
   + 生成`IProperty<T>`私有成员变量并在构造函数里生成代码调用`PropertyStore<T>.CreateImplicit()`方法为该成员创建实例。
   + 为属性生成`Getter`来从`IProperty<T>.Value`读取数据。
   + 为属性生成`Setter`来把数据写入`IProperty<T>.Value`。


```c#
private static Type CreateImplType(Type baseType)
{
  var assemblyName = new AssemblyName {Name = "<PropertyContainer>_Assembly"};
  var assemblyBuilder = AssemblyBuilder.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.Run);
  var moduleBuilder = assemblyBuilder.DefineDynamicModule("<PropertyContainer>_Module");
  var typeName = GetImplTypeName(baseType);
  
  ////IPropertyContainerMarker = IPropertyContainer, IPropertyChangeNotifiable, INotifyPropertyChanged
  var interfaces = new[] {typeof(IPropertyContainerMarker)};
  var typeBuilder = moduleBuilder.DefineType(typeName, TypeAttributes.Public, baseType, interfaces);

  ////Emit the PropertyChanged event and return its field so we can use it to implement the NotifyPropertyChanged method
  var eventFieldBuilder = EmitPropertyChangedField(typeBuilder);
  EmitNotifyPropertyChanged(typeBuilder, eventFieldBuilder);

  ////Emit the constructor and return the ILGenerator, it will be used to emit codes to create the IProperty<T>
  ////backing fields in the ctor
  var ctorIl = EmitCtor(typeBuilder, baseType);
  
  ////Emit the PropertyStore<T> field and corresponding property getter
  //// _store = new PropertyStore<BaseType_<PropertyContainer>_Impl>(this);
  var storeFieldBuilder = EmitPropertyStore(typeBuilder, ctorIl);
  
  ////This would be used to Emit codes like below to create backing fields for each property
  //// _name = _store.CreateImplicit<String>(nameof(PropertyArgs));
  var createPropertyMethod = typeof(PropertyStore<>).GetMethod("CreateImplicit", BindingFlags.Instance | BindingFlags.Public);

  ////Prepare the properties need to be overridden and generate property getter and setter
  var targetProperties = GetTargetProperties(baseType);
  foreach (var property in targetProperties)
  {
    ////type of string
    var propertyType = property.PropertyType;
    var propertyName = property.Name;
    var fieldName = ToFieldName(propertyName);
    ////type of IProperty<string>
    var fieldType = typeof(IProperty<>).MakeGenericType(propertyType);
    var fieldBuilder = typeBuilder.DefineField(fieldName, fieldType, FieldAttributes.Private);

    var fieldInfo = new PropertyFieldInfo(propertyName, propertyType, fieldBuilder);
      
    var valueProperty = fieldType.GetProperty("Value");
    var setValueMethod = valueProperty.GetSetMethod();
    var getValueMethod = valueProperty.GetGetMethod();

    ////private IProperty<string> _name;
    EmitPropertyBackingField(ctorIl, storeFieldBuilder, createPropertyMethod, fieldInfo);
    ////get => _name.Value; 
    EmitPropertyGetter(typeBuilder, property, fieldBuilder, getValueMethod);
    ////set => _name.Value = value; 
    EmitPropertySetter(typeBuilder, property, fieldBuilder, setValueMethod);
  }
  
  ctorIl.Emit(OpCodes.Ret);
  
  return typeBuilder.CreateType();
}
```

与`Unity Container`的集成非常简单。首先添加一个`TypeMapping`阶段的的构造策略`PropertyContainerStrategy`。

```c#
public class UnityPropertyContainerExt : UnityContainerExtension
{
  protected override void Initialize()
  {
    Context.Strategies.Add(new PropertyContainerStrategy(Container), UnityBuildStage.TypeMapping);
  }
}
```

然后在`PreBuildUp`方法中检查是否需要为当前类型创建动态类。

```c#
public override void PreBuildUp(ref BuilderContext context)
{
  base.PreBuildUp(ref context);

  var type = context.Type;
  if (_container.IsRegistered(type))
  {
    ////已经为当前类型显示调用了Container.RegisterType()，直接返回
    return;
  }

  if (type.IsSubclassOf<IPropertyContainerMarker>())
  {
    ////IPropertyContainerMarker接口是PropertyContainerImpl为生成的代码添加的接口。该接口存在表示当前类型已经是PropertyContainerImpl生成的类型，无需继续
    return;
  }
  
  ////用PropertyContainerTypeKey<T>来确定是否需要为当前类生成代码
  if (_container.IsRegistered(typeof(PropertyContainerTypeKey<>).MakeGenericType(type)))
  {
    var implType = PropertyContainerImpl.GetImplType(type);
    context.Type = implType;
    _container.RegisterType(type, implType);
  }
}
```

> 此处`PreBuild`所演示的代码仅限于`Unity Container 4.0`及以后版本。检查某个类型是否已经注册的行为在早期版本并不相同。早期版本中一方面`IsRegistered`方法仅仅作为一个供测试用的扩展方法存在，不仅低效而且并不线程安全，需要通过编写自定义代码重新实现。另一方面调用一个非接口的类型对于`Unity Container`来说始终是已经注册的状态。

## 3. 使用

使用非常简单，示例及注释如下：

```c#
public void Test()
{
  var container = new UnityContainer();

  ////把扩展添加到Unity Container
  container.AddNewExtension<UnityPropertyContainerExt>();
  ////告诉Unity Container需要为ViewModel类生成代码。
  container.MarkPropertyContainer<ViewModel>();
  
  ////vm的类型会是一个继承自ViewModel，类型名为"ViewModel_<PropertyContainer>_Impl"的类
  var vm = container.Resolve<ViewModel>();
  ////仅监听Name属性的变化
  vm.HookChange(o => o.Name, HandleNameChanged);
  ////监听传统PropertyChanged事件
  vm.HookChange(HandlePropertyChanged);
}

private void HandleNameChanged(object sender, ValueChangedArgs<string> e)
{
  Log($"[Name Changed] {e}");
}

private void HandlePropertyChanged(object sender, PropertyChangedEventArgs e)
{
  Log($"[Property Changed] {e.PropertyName}");
}
```

同样的事件监听代码在`ViewModel`内部也可以工作。

```c#
public class ViewModel
{
  public virtual string Name { get; set; }
  
  public virtual int Age { get; set; }
  
  [InjectionMethod]
  public void Init()
  {
    this.HookChange(o => o.Name, HandleNameChanged);
    this.HookChange(HandlePropertyChanged);
  }

  private void HandleNameChanged(object sender, ValueChangedArgs<string> e)
  {
    Log($"[Name Changed] {e}");
  }
  
  private void HandlePropertyChanged(object sender, PropertyChangedEventArgs e)
  {
    Log($"[Property Changed] {e.PropertyName}");
  }
}
```

完整实现请移步[*github*](https://github.com/eagleboost/PropertyContainer)

## 4. 结语

自此我们基本解决了为一个类自动生成`INotifyPropertyChanged`接口的问题，并且引入`IProperty<T>`的概念消除了调试及性能问题。更多的应用可以在此基础上展开，比如：

1. 把所有属性的值改变过程写入日志。这在过去是一项常常会涉及反射的繁琐工作，现在只需要从生成的对象调用`IPropertyStore.GetProperties()`方法，监听每个属性的`ValueChanged`事件即可，这项工作很容易创建为一个可重用的组件。
2. 属性值校验。`WPF`在数据绑定层面提供了属性值校验的机制，`ViewModel`只需实现`IDataErrorInfo`或者`INotifyDataErrorInfo`接口(用于异步场合)。看似简单，但与`INotifyPropertyChanged`接口类似的问题同样存在：首先`IDataErrorInfo`接口的实现大同小异而繁琐。其次该接口要求`ViewModel`本身而不是其它对象来实现(通过绑定来隐式发现)，极易导致`ViewModel`代码混乱，重用也不容易。第三，虽然是小问题，但也存在一个属性变化触发无关属性被更新的问题。接下来我们将会讨论一种方法解决所有问题。
3. 脏数据跟踪。这也是有`UI`的应用程序常见的需求，`WPF`并未提供对脏数据跟踪的内建支持。一个常见的简单做法是让`ViewModel`暴露一个`IsDirty`属性，当其余任何属性发生变化时把`IsDirty`设置为`True`。这种方法有几个缺点，一是`ViewModel`需要包含又一个可选的额外机制，容易导致代码混乱。二是`IsDirty`状态的维护会是个问题，因为谁可以把它设置为`True`。三是缺乏真正的脏数据跟踪，比如用户把一个属性的值从`A`改成`B`再改回`A`，虽然`PropertyChanged`事件触发了几次，但`IsDirty`应该是`False`而不是`True`。

基于`IProperty`的概念，我们将能够轻松编写可重用并且可插拔的组件来解决上述几种应用场景的问题。