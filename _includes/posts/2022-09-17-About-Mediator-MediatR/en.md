I've been interviewing candidates in the past several months. The candidates are from different sources, both internal and external. In general candidates from Investment Banking industry is better than those from other backgrounds. One of the candidates from Morgan Stanley I interviewed last week even try to teach me a lesson.

Before closing up the interview, as usual I asked the candidate if he has any questions for me. Most of the candidates in this case ask questions related to roles and responsibilities, career development etc. However, the question this candidate (let's call him `S`) asked was "during the interview you mentioned that your project uses `Dependency Injection`, may I ask are the dependencies injected via constructor parameters or properties". It's quite a unique question and it surprised me. I told him we use both forms, and personally I prefer property injection because it's simpler. Constructor injection would require child class to include a constructor to call the base constructor, if inheritance is used, and more parameters need to be added if more dependencies are needed. Given that they generate almost the same thing, constructor injection gives me more troubles than property injection.

`S` said, yes, too many constructor parameters usually means the design has problems. I agree with that. He continued with "have you heard about the Meditator pattern"? I only have some vague memories about it as I barely use it. He said compare to dependency injection he'd prefer the `Mediator` pattern and it solves the problem of too many dependencies.

After more talks I got to know that in order to solve the problem of too many dependencies in his project——to save time I didn't ask why it's a problem, but it should be because they use constructor dependencies a lot so it becomes headaches——he switch to the `Mediator` pattern, to be more specific, he used a library called [MediatR](https://github.com/jbogard/MediatR), and put all dependencies any ViewModel can possibly have into the `Mediator`, so the ViewModels only depend on the `Mediator` and the codes are also simpler.

Although I don't use `Mediator` in the current project, I quickly sniffed the bad smell of this design from his words. Apparently, on one hand, the dependencies of the ViewModel are not clear anymore, on the other hand, the `Mediator` may end up holding too many stuff that are totally unrelated.

I gave him an example, consider a ViewModel needs to show message box and input box, so there're two dependencies, when writing tests, we mock them when we see the properties.

```c#
public class UserManager
{
  [Dependency]
  public IMessageBox MessageBox { get; set; }

  [Dependency]
  public IInputBox InputBox { get; set; }
}
```

But if we change it to the `Mediator` way, the usages would be like this:

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

For the `IMediator` interface, there're several possibilities in terms of the signature of the Send method:

1. `Send` method accepts a parameter of type `object`. It's a bad design obviously.
   
2. `Send` method accepts a parameter of some sort of interface, e.g. `IMediatorContext`. It seems more reasonable than `object` but just worse. First of all, all parameters need to implement this interface before they can be passed, second, because the concept of Mediator is so general, this interface would end up being an empty interface without any member because nothing in common can be abstracted. The `IDialogService` example I used in [Interactions between the View and ViewModel](https://eagleboost.com/2021/11/02/Interactions-between-the-View-and-ViewModel/) is a comparable bad design. But `IDialogService` is in a slightly better position because at least it seems information related to a dialog can be abstracted into an interface.
   
3. `Send` method is generic, i.e. `Send<T>(...)`. It seems reasonable but nothing substantially gets better. 

In fact no matter which way above is used, the `Mediator` would inevitably put a lot of unrelated stuff together and becomes a mess quickly. One of the questions I often ask in the interview is how to open a dialog from a ViewModel and get user inputs without breaking the M-V-VM principal, i.e. keep clean separation between the View and the ViewModel. So far the best answer the candidates can give is the solution of [InteractionRequest](https://prismlibrary.com/docs/wpf/legacy/Advanced-MVVM.html) introduced by `Prism`, not perfect but usable. Most of the candidates can only come up with something similar to `IDialogService`, which is a solution that almost everybody can work out. Implementation of `IDialogService` would choose templates based on type of the parameters to create dialogs, so ultimately it would need to handle all kinds of dialogs in the application and would be a total mess. 

The mess so far is only about implementations, when it comes to tests, because the dependencies are not clear, writing tests for classes like `UserManagerWithMediator` would involve more work to do.

Finally `S` told me that `Mediator` is so good that he's PROUD of having chosen it to solve problems, and he would like to show me how to use it in detail if there's a chance, because it seems I still didn't quite get how good it is.

After the interview I reviewed the `Mediator` pattern and the [MediatR](https://github.com/jbogard/MediatR) library. Not surprisingly, I also found quite a few articles talking about "why you shouldn't use the `Mediator` pattern"

+ [Is Mediator/MediatR still cool?](https://alex-klaus.com/Mediator/)
+ [You Probably Don't Need to Worry About MediatR](https://jimmybogard.com/you-probably-dont-need-to-worry-about-mediatr/)
+ [You probably don't need MediatR](http://arialdomartini.github.io/mediatr)
+ [No, MediatR Didn't Run Over My Dog](https://scotthannen.org/blog/2020/06/20/mediatr-didnt-run-over-dog.html)

Below is a definition of quoted from [`C# Mediator`](https://www.dofactory.com/net/`Mediator`-design-pattern). 

>The `Mediator` design pattern defines an `object` that encapsulates how a set of `object`s interact. `Mediator` promotes loose coupling by keeping `object`s from referring to each other explicitly, and it lets you vary their interaction independently.

A common scenario where the `Mediator` pattern can be used is the Chat Room, user A and user B can send messages to each other via a Chat Room without directly connecting to each other. The good and bad of the `Mediator` pattern is out of the scope of this article, but it's very clear that it's not intended to solve the problem of too many dependencies or too many constructor parameters. It's simply a tool for solving certain types of problems. The candidate `S` just abused it to solve the problem it's not supposed to solve.

I've also read the core part of the [MediatR](https://github.com/jbogard/MediatR/blob/master/src/MediatR/Mediator.cs) source codes, it uses the third way stated above, aka based on type of T passed to the `Send<T>(...)` method, it creates an instance of the handler and caches it in a dictionary for future use. The quality is quite amateur level not only the interface design but also the implementations, although it has `8.5k` on [github](https://github.com/jbogard/MediatR). This also tells a truth that average programmers don't quite care about or understand what good designs are. 

I actually have used something similar to the `MediatR` library in one `Android` app I wrote this year, the `Xamrin` [MessageCenter](https://learn.microsoft.com/en-us/xamarin/xamarin-forms/app-fundamentals/messaging-center). The only difference is that `MessageCenter` implementation is better.