组里从去年到今年因为各种原因接连走了几个人，最近几个月在密集地面试，希望能把空位填上。候选人来源不一，质量也良莠不齐。总体上来自投行的比来自别的行业的候选人平均水平还是高出一截，前几天一个来自大摩的候选人（下称`S`）甚至试图给我上一堂课。

细节略去，面试结束后我照例问对方有什么问题想问。一般说来大多数人都围绕职位细节，成长空间等等提问，这位目测应该比我年长一些的`S`的问题则是“面试中你提到你们的项目使用了依赖注入，我想问是构造函数注入还是属性注入”。第一次遇到类似的问题，老实说挺让我惊喜的。我告诉他都有，以为他想探讨一下各自的优劣，于是说我个人更多使用属性注入，代码更清爽，因为构造函数注入会导致在有继承的情况下子类不得不包含带参数的构造函数来调用基类构造函数，如果依赖增加，参数也会增加，在完成同样工作的情况下构造函数注入没有给我带来任何好处。

`S`接着说，嗯如果构造函数参数太多那通常说明设计有问题了，我表示同意。他接着说你听过`Mediator`模式吗？以前学习设计模式的时候我有些印象，但是因为项目中很少用细节也不记得了。他说相比依赖注入我更喜欢用`Mediator`模式，可以一举解决依赖过多的问题。

谈话从这里开始变得有意思了。原来在他的项目中为了解决多个依赖的问题——时间关系并未深入问为什么这是个并不是问题的问题，估计他碰到的情况是代码过多使用了构造函数依赖导致很头痛——使用了`Mediator`模式，具体来说用了一个叫做`MediatR`的类库，把能想到的依赖全都包装到`Mediator`里面去，从而`ViewModel`只需要依赖`Mediator`即可。代码从此变得清爽，世界仿佛也宁静了。

虽然没有使用过`Mediator`，但听他的叙述我立即发现了其中的问题——如果把依赖都包装进`Mediator`，那么一方面`ViewModel`的依赖是什么变得不清晰，另一方面这个`Mediator`会背负太多的包袱。

我举了一个例子，一个`ViewModel`需要显示消息框和输入框，因此有两个依赖，那么编写测试代码的时候看到`Dependency`并`mock`这两个接口即可。

```c#
public class UserManager
{
  [Dependency]
  public IMessageBox MessageBox { get; set; }

  [Dependency]
  public IInputBox InputBox { get; set; }
}
```

如果使用`Mediator`模式，那么在调用的时候就是这样：

```c#
public class UserManagerWithMediator
{
  [Dependency]
  public IMediator Mediator { get; set; }

  public void ShowMessageBox()
  {
    ...
    Mediator.Send(new MessageBoxData(...));
    ...
  }

  public void ShowInputBox()
  {
    ...
    Mediator.Send(new InputBoxData(...));
    ...
  }
}
```

对`Mediator`来说有几种可能：

1. `Send`方法接受一个`object`类型的参数。这肯定是糟糕的设计。
   
2. `Send`方法接受某个接口作为参数，比如`IMediatorContext`。看起来比`object`稍微合理一点，但其实更烂，因为一方面参数必须要实现该接口才行，另一方面由于`Mediator`的概念过于宽泛，该接口最终只能是一个不包含任何成员的空接口。我在[M-V-VM视图交互简述](https://eagleboost.com/2021/11/02/Interactions-between-the-View-and-ViewModel/)中提到过的`IDialogService`就是一个可类比的坏设计，不过`IDialogService`稍微好一点，因为至少（看上去）跟对话框相关的东西可以抽象到一个接口中。
   
3. `Send`方法是泛型的，即`Send<T>(...)`。看起来更高级一点，但从设计角度来说并没有实质性的提升。

实际上但无论哪种方式，`Mediator`都会不可避免地会把不相关的东西放在一起，从而变成一个大杂烩。我在面试中常问的一个问题是如何在不违反`M-V-VM`模式的前提下从`ViewModel`中打开一个对话框并获取用户输入。目前为止最好的面试者能给出的解决方案是`Prism`的[InteractionRequest](https://prismlibrary.com/docs/wpf/legacy/Advanced-MVVM.html)，可用但不完美。绝大多数面试者则只能达到`IDialogService`的程度（差不多也是人人都能想到的程度）——用一个类来根据输入参数的类型选择适当的模板来创建对话框，这个类最终会变成一个大杂烩。

说大杂烩指的是实现的问题，此外还有测试，由于依赖不明确，针对`UserManagerWithMediator`这样的类写测试的话还有额外的工作要做。

`S`最后还表示`Mediator`实在是好，他特地用了“自豪”一词来强调他选择这个模式的英明，并表示如果有机会一起工作他会详细演示给我看是怎么用的，因为我似乎没太听懂其好处所在。

我并没有跟他再纠缠下去。结束面试后重温了一下`Mediator`模式，以及他提到的[MediatR](https://github.com/jbogard/MediatR)类库。毫不意外地也顺手搜到不少类似“为什么不要是使用Mediator模式”的文章，比如这几篇

+ [Is Mediator/MediatR still cool?](https://alex-klaus.com/Mediator/)
+ [You Probably Don't Need to Worry About MediatR](https://jimmybogard.com/you-probably-dont-need-to-worry-about-mediatr/)
+ [You probably don't need MediatR](http://arialdomartini.github.io/mediatr)
+ [No, MediatR Didn't Run Over My Dog](https://scotthannen.org/blog/2020/06/20/mediatr-didnt-run-over-dog.html)

从定义上说，`Mediator`模式通过一个类封装一组对象之间的交互，其目标是避免这些对象之间相互引用从而实现松耦合。该定义来自[`C# Mediator`](https://www.dofactory.com/net/`Mediator`-design-pattern)。

>The `Mediator` design pattern defines an `object` that encapsulates how a set of `object`s interact. `Mediator` promotes loose coupling by keeping `object`s from referring to each other explicitly, and it lets you vary their interaction independently.

常见的例子是聊天室，用户`A`通过聊天室(也就是`Mediator`)与用户`B`互相发送消息而不用相互直接关联。`Mediator`模式的好坏不谈，但显而易见它要解决的问题并不是“依赖注入过多”或者“构造函数注入参数过多”，它只是用来解决某一类问题的工具而已。而候选人`S`显然是把它滥用了来解决不该它解决的问题。

也浏览了一下[MediatR](https://github.com/jbogard/MediatR/blob/master/src/MediatR/Mediator.cs)的核心代码，用的是上面说的第三种泛型参数的方法，根据`Send<T>(...)`传过来的`T`的类型创建相应的类（因此`T`不能是接口）并缓存到字典里面供下次调用，不论是接口规范的设计还是实现细节都是很业余的水平，但不妨碍其在[github](https://github.com/jbogard/MediatR)上得到`8.5k`星，可见一般程序员并不太关心或者不太能理解什么是好的设计。

又想起来我在前段时间写的一个`Android`小程序还真用过与`MediatR`类似的东西，那就是`Xamrin`自带的[MessageCenter](https://learn.microsoft.com/en-us/xamarin/xamarin-forms/app-fundamentals/messaging-center)，不同之处是代码质量高了不少。