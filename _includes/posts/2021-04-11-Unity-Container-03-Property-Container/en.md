## 1. The Problem

In the previous [post](https://eagleboost.com/2021/04/06/Unity-Container-(2)-Implementing-the-INotifyPropertyChanged-Interface/) we talked about `Reflection.Emit` being the best option to implicitly implement the `INotifyPropertyChanged` interface, with several points that can be improved：

1. Not debug friendly.
2. Impact performance when properties are many.
3. Not easy to get previous value of a property.

These 3 problems are essentially one problem that we need some extra property level logic. Problem would be solved if the property's `Backing field` is replaced with something like an `IProperty` object, as we can see blow:

Let's start with defining `IProperty<T>` as the replacement of `backing field` of a property of type T:

```c#
public interface IProperty<T>
{
  string Name { get; }

  T Value { get; set; }

  event EventHandler<ValueChangedArgs<T>> ValueChanged;
}
```

Then given `ViewModel` we emit equivalent IL codes to generate a `ViewModel_<PropertyContainer>_Impl` class. The `PropertyStore` class would be responsible for creating and caching instances of `IProperty<T>`.

```c#
//Original class
public class ViewModel
{
  public virtual string Name { get; set; }
}
  
//Generated class
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

Now we can easily fix the 3 problems listed above:

1. Not debug friendly - Set a break point in the setter of the `Value` property of an `IProperty<T>` implementation class.
2. Impact performance when properties are many - directly listen to the `ValueChanged` event of `IProperty<T>`
3. Not easy to get previous value of a property - `EventArgs` of the `ValueChanged` already contains previous and current value of the property.

Since a `T` now becomes a `IProperty<T>`, so it uses more memory, but it's not a big concern in our scenario. This approach apparently is not designed for objects that might have thousands or even millions of instances in the memory. It's more suitable for writing classes like `ViewModels` that even in a large scale application total number of `ViewModels` would not be crazy large.

## 2. Implementations

The whole process of using `Reflection.Emit` to generate codes are pretty standard and tedious, we only list the high level steps:

1. Create a new class and make it derive from `BaseType` and implement these interfaces:
   + `IPropertyContainer` - used in the extension methods to retrieve the instance of `IPropertyStore` created implicitly.
   + `INotifyPropertyChanged` - the must have interface。
   + `IPropertyChangeNotifiable` - used to raise `PropertyChanged` event on the generated class when `IProperty<T>.Value` is changed.

2. Implement the `INotifyPropertyChanged` interface
   + Emit the `PropertyChanged` event and its backing field.
   + Emit the `NotifyPropertyChanged` method used to raise the `PropertyChanged` event.
  
3. Emit codes for all public `virtual` properties of the `BaseType`:
   + Emit private field of `IProperty<T>` and call `PropertyStore<T>.CreateImplicit()` to create instance for the field in the constructor.
   + Emit `Getter` for the property to read value from `IProperty<T>.Value`.
   + Emit `Setter` for the property to set value to `IProperty<T>.Value`.


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

Integration with `Unity Container` is also simple. We create a `PropertyContainerStrategy` and add it to the `TypeMapping` stage.

```c#
public class UnityPropertyContainerExt : UnityContainerExtension
{
  protected override void Initialize()
  {
    Context.Strategies.Add(new PropertyContainerStrategy(Container), UnityBuildStage.TypeMapping);
  }
}
```

then check if the dynamic creation process is needed for the current type in the `PreBuildUp` method.

```c#
public override void PreBuildUp(ref BuilderContext context)
{
  base.PreBuildUp(ref context);

  var type = context.Type;
  if (_container.IsRegistered(type))
  {
    ////The current type is already registered by calling Container.RegisterType(), do nothing.
    return;
  }

  if (type.IsSubclassOf<IPropertyContainerMarker>())
  {
    ////The IPropertyContainerMarker interface is added dynamically to the generated class by PropertyContainerImpl, it's not supposed to be used directly by any class, so presence of the interface means the current type is already generated by PropertyContainerImpl, so do nothing
    return;
  }
  
  ////Use PropertyContainerTypeKey<T> to check if we need to generate the class for the current type
  if (_container.IsRegistered(typeof(PropertyContainerTypeKey<>).MakeGenericType(type)))
  {
    var implType = PropertyContainerImpl.GetImplType(type);
    context.Type = implType;
    _container.RegisterType(type, implType);
  }
}
```

> Please note that the above `PreBuild` method works only in `Unity Container 4.0` and above. The behavior of determine is a type is registered is different in earlier versions of `Unity Container`. On one hand, the `IsRegistered` method is just an extension method that is not production ready, it's extremely slow and not thread safe. On the other hand, a non-interface type is always registered from `Unity Container` point of view.

## 3. Usages

Using it is also straightforward:

```c#
public void Test()
{
  var container = new UnityContainer();

  ////Add the extension to the Unity Container
  container.AddNewExtension<UnityPropertyContainerExt>();
  ////Tell Unity Container to generate the class for ViewModel.
  container.MarkPropertyContainer<ViewModel>();
  
  ////Type of vm will be a class named 'ViewModel_<PropertyContainer>_Impl' and derived from ViewModel
  var vm = container.Resolve<ViewModel>();
  ////Only listen to the Name property's changes
  vm.HookChange(o => o.Name, HandleNameChanged);
  ////Listen to the traditional PropertyChanged event
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

Similar event listeners also work inside the `ViewModel` class.

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

Please visit [*github*](https://github.com/eagleboost/PropertyContainer) for complete implementations.


## 4. Applications

Now we can already generate codes to automatically implement the `INotifyPropertyChanged` interface for classes, and fixed some problems by introducing `IProperty<T>`. This approach not only simplifies the process of writing `ViewModel` classes, it can also change the way some other functions were done before and more applications can be developed based on it:

For example, there're cases we want to log all property changes. This job usually requires reflection to access property values when the. Now we just need to call `IPropertyStore.GetProperties()` and listen to `ValueChanged` event of all properties, then access property value directly.

The second example is `input validation`. `WPF` already supports validation at Data Binding level. Validation rules can be configured in `XAML`, or in `M-V-VM` world `ViewModels` can also implement either `IDataErrorInfo` or `INotifyDataErrorInfo` interface (for asynchronous scenarios) and do the validation at data level. The concept is simple but like the `INotifyPropertyChanged` interface, similar problems still exist:

1. The whole process of implementing `IDataErrorInfo` is pretty standard but not simple. 
2. The `IDataErrorInfo` interface has to be implemented by the `ViewModel` itself (Binding Source), which makes the `ViewModel` fat and mess, also not easy to reuse.
3. When `IDataErrorInfo` is used, data binding is applied to to indexer, which means we'll need to raise `PropertyChanged` event with an event args named `Item[]` and cause refresh on all of the properties. It's a small problem but not necessary.

Another example made easy is also pretty common in `UI` applications - `dirty tracking`. `WPF` does not have built-in support for dirty tracking. One common approach used by developers is exposing a `IsDirty` property on the `ViewModel` and set it to true when the `PropertyChanged` event is triggered. This approach also has some obvious problems:

1. Dirty tracking is an optional function but the implementation could easily mess with business logics.
2. Maintaining the state of `IsDirty` could be troublesome as anyone can set the property to true.
3. Lack of real dirty tracking. Assume user change one property from `A` to `B` then change it back to `A`, although the `PropertyChanged` event is triggered multiple times but `IsDirty` should be `False` instead of `True`。

Based on the `IProperty` idea, next we'll discuss reusable solutions to fix all of the problems.