## 1. The problem

The `INotifyPropertyChanged` interface is probably one of the most famous interfaces in the interface oriented programing history. It has been used extensively almost everywhere, especially `UI` applications.

Because it's so popular, developers have to spend time to keep writing similar codes for properties. A typical `ViewModel` needs to contain at least below codes or equivalent logic, even when it has only one property, and as we can see that the property setter is the most tedious piece among all these codes.

> Developers can use `[CallerMemberName]` to save the explicit passing of the property name, but it only works in `.Net 4.0` and later versions.

```c#
public class ViewModel : INotifyPropertyChanged
{
  private string _name;
  
  public string Name
  {
    get => _name;
    set
    {
      if (_name != value)
      {
        _name = value;
        NotifyPropertyChanged();
      }
    }
  }

  public event PropertyChangedEventHandler PropertyChanged;

  private void NotifyPropertyChanged([CallerMemberName] string name = null)
  {
    PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
  }
}
```

So various ways have been created to simplify the process of writing a `ViewModel`:

1. Put the common codes into a base class such that the child classes can focus on property `Getter` and `Setter`. But it does not work for the case that base class cannot be changed.
2. Extract `Property Setter` into some generic method like `SetValue` and use `EqualityComparer<T>.Equals()` to do the value difference check. This can help simplify the property setter code to one line.
3. Dynamically generate codes.

Since the pattern to implement the `INotifyPropertyChanged` interface and raise event in the property setter is pretty common, so if things come true obviously auto-generated codes can simplify the efforts of writing classes like `ViewModel`.

## 2. InterceptionBehavior

The `Interception` in `Unity Container` is one of the ways to simplify the codes. One of the `Unity` series articles on Microsoft docs [Implement INotifyPropertyChanged Example](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff660851(v=pandp.20)?redirectedfrom=MSDN#implement-inotifypropertychanged-example) provides a demo implementation. Assume `ViewModel` and its interface are like this：

```c#
public interface IViewModel
{
  string Name { get; set; }
  
  int Age { get; set; }
}

public class ViewModel : IViewModel
{
  public string Name { get; set; }
  
  public int Age { get; set; }
}
```

We can use [`InterceptionBehavior`](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff660851(v=pandp.20)?redirectedfrom=MSDN#implement-inotifypropertychanged-example) to intercept the calls to property setters and add/remove to the PropertyChanged event to implicitly implement the `INotifyPropertyChanged` interface.

```c#
var container = new UnityContainer();
container.AddNewExtension<Interception>();

var interceptor = new Interceptor<InterfaceInterceptor>();
var behavior = new InterceptionBehavior<NotifyPropertyChangedBehavior>();
container.RegisterType<IViewModel, ViewModel>(interceptor, behavior);
```

It works, but has problems：

1. `Unity Container` actually creates a new class dynamically to implement the same interface and wrap the original be intercepted class, so the `IViewModel` interface is required, otherwise `Unity Container` would throw exceptions of "`Type is not interceptable`".
2. Because event is a special delegate that can only be called from inside of a class, `NotifyPropertyChangedBehavior` actually provides its own `PropertyChanged` event to support add/remove of the event listeners. So listen to the property changes only works from outside of the class, event listeners attached from inside the class won't work because the event is not used at all.

## 3. Reflection.Emit

The ultimate solution is to dynamically generate codes. One way is to generate codes at run time via `Reflection.Emit`, another way is to run some MSBuild post process tasks to generate codes based configurations like attribute and edit the assemblies. `AOP` libraries like [PostSharp](https://www.postsharp.Net/) work in this way. In the end these two approaches are both manipulating IL, but we'll only talk about the first approach in this blog. For the `InterceptionBehavior` above, `Unity Container` internally also creates a `Proxy` class via `Reflection.Emit`, just that its purpose is to expose something to allow developers put custom logic in the behaviors.

> `PostSharp` also provides [support](https://doc.postsharp.Net/inotifypropertychanged) to implement `INotifyPropertyChanged`, but `PostSharp` is not free and its approach is similar to what we're going to talk about.

So write our own codes is the best way to properly generate implementations for the `INotifyPropertyChanged` interface.

1. No need to define interface for the source class.
2. Allow attaching `PropertyChanged` event listeners from inside and outside of the class.

This article [Dynamically generating types to implement INotifyPropertyChanged](https://grahammurray.wordpress.com/2010/04/13/dynamically-generating-types-to-implement-inotifypropertychanged/) gives a relatively complete implementation. It creates a dynamic class inheriting from the existing class (so no interface is needed), the existing class just need to mark the properties as `virtual` so they can be overridden.

I have created a project to integrate the codes with `Unity Container` and put on [github](https://github.com/eagleboost/NotifyPropertyChangedImpl). As long as a class implement the `IAutoNotifyPropertyChanged` interface (any other way is also fine), when being resolved by the Container, a dynamic class would be created with `INotifyPropertyChanged` interface implemented.

```c#
public class NotifyPropertyChangedImplStrategy : BuilderStrategy
{
  public override void PreBuildUp(ref BuilderContext context)
  {
    base.PreBuildUp(ref context);

    if (context.Type.IsSubclassOf<IAutoNotifyPropertyChanged>())
    {
      var implType = NotifyPropertyChangedImpl.GetType(context.Type);
      context.Type = implType; 
    }
  }
}
```

## 4. There're still problems

Although the `Reflection.Emit` approach can save us repeated works, but there're still rooms to improve:

First of all, it's not debug friendly. Since the property setter is emitted `IL` codes, so there're no source codes to set break points in the setter, there're workarounds but not convenient.

Second, the `INotifyPropertyChanged` interface is a very high level abstraction, so it might cause problems if not used properly:

1. Performance impact. Say if we're only interested in changes of several properties, we need to match the property name in the event handler. If the object has many properties then the event handler would be called many times because of irrelevant property changes, it can cause performance problems in extreme cases.
2. Inconvenience. There're cases we want to know the previous and current value of the property, one common way is letting the object implement the `INotifyPropertyChanging` interface, then we subscribe to it and store previous value of the properties. It works but apparently not quality coding.

There should be better solutions to fix these problems.