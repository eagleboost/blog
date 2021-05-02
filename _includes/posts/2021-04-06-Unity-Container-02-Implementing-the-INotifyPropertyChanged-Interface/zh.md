## 1. 问题

如果说面向接口编程的历史上接口的著名程度有一个排名，`INotifyPropertyChanged`排前三应该没有太大争议。`INotifyPropertyChanged`高度抽象化了对象的“属性发生改变”这一行为，在应用程序尤其是包含`UI`的应用程序中可以说无处不在。

然而正因为其无处不在，开发人员不得不耗费大量时间来编写重复的代码。就算是只包含一个属性的`ViewModel`也至少需要包含下面的代码或等价的逻辑，其中以`Property Setter`的编写尤其繁琐。

> 在`.Net 4.0`以后的版本中已经可以通过`[CallerMemberName]`来省去对属性名的重复。在早期版本中每个调用`NotifyPropertyChanged`的地方还需要显式传递属性名。

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

聪明的开发人员想出了各种办法来简化`ViewModel`的编写过程：

1. 把基本的实现放到一个基类中，子类只需要实现属性的`Getter`和`Setter`。但不能解决基类不能更改的情况。
2. 把`Property Setter`的代码提取到一个泛型方法比如`SetValue`中，用`EqualityComparer<T>.Equals()`来代替值的比较，这种方法可以把`Setter`简化到一行代码。
3. 自动生成代码。

由于`INotifyPropertyChanged`接口的实现以及触发属性变化的过程对每个类来说都是类似的逻辑，自动生成代码如果能够实现显然会带来简化的效果。

## 2. InterceptionBehavior

比较简单的一种方法是使用`Unity Container`中的`Interception`。Microsoft docs 上`Unity`系列文章里有一篇[Implement INotifyPropertyChanged Example](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff660851(v=pandp.20)?redirectedfrom=MSDN#implement-inotifypropertychanged-example)给出了一种实现。假设`ViewModel`及其接口定义如下：

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

可以通过`InterceptionBehavior`来截获对属性的`Setter`以及添加删除`PropertyChanged`事件订阅的调用来隐式实现`INotifyPropertyChanged`接口，具体实现可以参考[这里](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff660851(v=pandp.20)?redirectedfrom=MSDN#implement-inotifypropertychanged-example)。

```c#
var container = new UnityContainer();
container.AddNewExtension<Interception>();

var interceptor = new Interceptor<InterfaceInterceptor>();
var behavior = new InterceptionBehavior<NotifyPropertyChangedBehavior>();
container.RegisterType<IViewModel, ViewModel>(interceptor, behavior);
```

这种方法基本可行，但存在不小的缺陷：

1. `Unity Container`实际上创建了一个实现相同接口的类来封装现有类，所以接口`IViewModel`是必须的，否则会抛出"`Type is not interceptable`"的异常。
2. 由于事件是只能在一个类的内部调用的特殊委托，`NotifyPropertyChangedBehavior`实际上自己定义了一个`PropertyChanged`事件来替代被截获类的`PropertyChanged`事件，因此监听属性改变只对外部代码有效，任何在类的内部监听属性改变的尝试都是徒劳的，因为被监听类自己的`PropertyChanged`事件根本没有被用到。

## 3. Reflection.Emit

终极的解决方案是动态生成代码。一种方法是在运行时通过`Reflection.Emit`动态生成代码，另一种是在`MSBuild`的`post process`阶段根据`Attribute`之类的配置生成代码并修改`Assembly`。[PostSharp](https://www.postsharp.Net/)之类`AOP`库使用的就是第二种方法。两种方法归根到底都是操作`IL`代码，本文只讨论在运行时生成代码的方式。实际上在上述`Interception`方法里`Unity Container`也使用`Reflection.Emit`动态生成了一个`Proxy`类，只不过其目的是为了允许开发人员添加自定义代码。

> `PostSharp`对实现`INotifyPropertyChanged`接口也提供了[支持](https://doc.postsharp.Net/inotifypropertychanged)，不过`PostSharp`并非免费，其实现也与我们将要提到的方法大同小异。

为了完美地自动实现`INotifyPropertyChanged`接口，最好的办法是自己生成代码：

1. 无需定义接口。
2. 可在类的外部和内部监听`PropertyChanged`事件。

这篇文章[Dynamically generating types to implement INotifyPropertyChanged](https://grahammurray.wordpress.com/2010/04/13/dynamically-generating-types-to-implement-inotifypropertychanged/)就给出了一个相对完整并可用的实现——动态生成一个类来继承现有类，所以无需接口，唯一的要求是把属性标记为`virtual`以便动态生成代码来重载`Property Setter`。

我把整理了代码并与`Unity Container`简单整合了一下放到了[github](https://github.com/eagleboost/NotifyPropertyChangedImpl)——通过一个`IAutoNotifyPropertyChanged`接口(也可以用别的方法)来标记一个类，一旦检测到该接口的存在就创建动态一个类来实现`INotifyPropertyChanged`接口并重载所有属性的`Setter`。

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

## 4. 还有问题

虽然动态生成代码的方法帮助我们节省了不少重复劳动，但还有改进的空间：

首先代码调试不方便。动态生成的类`Emit`了`IL`代码来重载属性的`Setter`，所以没有源代码也无法直接添加断点进行调试，只能通过添加事件处理函数的方式来绕道而行。

其次，前面说过`INotifyPropertyChanged`接口高度抽象了属性发生改变的行为，这一高度抽象一定程度上也带来了两个问题：

1. 性能损失——假如我们只希望监听某几个属性的改变，只能在`PropertyChanged`事件处理函数里检查属性名是否匹配。如果被监听的对象属性众多那么事件处理函数会因为无关属性的改变被不必要地调用很多次，极端情况会带来性能问题。
2. 缺乏便利性——有时候我们需要在事件触发时知道属性的当前值和之前的值。原生的做法是让对象实现另一个`INotifyPropertyChanging`接口，然后监听其`PropertyChanging`事件来记录属性之前的值。这可行但不是好的代码。

因此我们需要一套更好的实现来改进这几个缺陷……