## 1. The problem

Displaying dialogs/windows is a common scenario in desktop applications. Modal dialogs are used most often, like `MessageBox`, or more complex ones like `Login Dialog` that requires user inputs, while modeless dialogs is less common that most of the small size applications don't use it at all.

The concept seems simple, but when it comes to implementation "how to display a window" is indeed not trivial, at least it's not as simple as it looks. Below are some design problems to consider other than just "create a window and display it"：

+ **Separation** **of** **View** **and** **Data**——This is an important implication of `M-V-VM` because the views need be upgraded/optimized independently.
+ **Easy** **to** **test**——Codes that directly create and display windows are not easy to test, however, the business logic would be easy to test if good separation of the view and data is implemented.
+ **Asynchronous** **interaction**——Complex interaction processes sometimes involve asynchronous operations, for example, displaying a dialog for user to input during a login process.
+ **Unified** **programming** **model**——Modal dialogs are naturally `interactive` because the parent window is disabled before a modal dialog is closed. Modeless dialogs althought seems different but we should be able to handle its interaction in similar way. For example, click a button on the main window to show a modeless dialog (or pass data to an already opened modeless dialog), and click some button on the modeless dialog to notify completion of job, then we can either close/hide the modeless dialog or clear out its content so it can wait for the next invocation.

Common approaches of displaying dialogs usually only tackle the first two points, and with noticeable flaws and limitations.

 
## 2. Dialog Service

A `Dialog Service` is the most intuitive way almost all developers can think of, after all abstraction of interfaces is so text book that all modern application developers should know. Take `MessageBox` as an example, anyone can come up with below interface:

```c#
public interface IMessageBox
{
  bool Show(string header, string content, MessageBoxType type, MessageBoxIcon icon);
}
```

The implementation of the `IMessageBox` interface can then create a custom window or simply call methods of the `MessageBox` class (a `Win32 API` wrapper) shipped with `WPF`. Using interface ensures `view-data-separation` and makes it easy to test, but its problems are also as obvious as its idea:

+ We might want to add some visual effects for certain types of important messages.
+ User might want the message box to have different background in some cases.
+ Complex scenarios like supporting "`Yes`", "`No`" and ”`Cancel`" button along with a "`Do not show this dialog next time`" checkbox.
+ In a single `ViewModel` that interacts with different type of dialog, it's hard to differentiate the calls and test them separately.

To satisfy/fix those requirements/problems, the `Dialog Service` approach would most likely end up giving use more and more method parameters (can be replaced by an object), more and more methods, and more and more interfaces. To summaries, it's not flexible enough, it's incompatible with asynchronous programming fashion and not capable of handling interactions with modeless dialogs.


## 3. InteractionRequest

I've been using the well known [Prism](https://prismlibrary.com) library in my projects as the foundation of `M-V-VM` practices. [Prism](https://prismlibrary.com)‘s answer to this problem is something called [InteractionRequest](https://prismlibrary.com/docs/wpf/legacy/Advanced-MVVM.html), which is based on `System.Windows.Interactivity`. The `ViewModel` creates `IInteractionRequest` objects, and the `View` hook up to the `Raised` event of the objects via `InteractionRequestTriggers`. When the `Raised` event is triggered in the `ViewModel`, the Trigger would create a dialog specified in the `XAML` and make it visible. When the dialog is closed, the `callback` passed along with the parameters is invoked and result is passed back to the `ViewModel`.


```c#
public interface IInteractionRequest
{
  event EventHandler<InteractionRequestedEventArgs> Raised;
}
```

It looks great when I saw it for the first time, and I have used it extensively before in my projects. [InteractionRequest](https://prismlibrary.com/docs/wpf/legacy/Advanced-MVVM.html) creatively uses event as the bridge and achieves separation of data and view, but also because of event, cross threading interaction becomes hard to implement. It also requires view to subscribe to the event. So if we want to display a dialog from a background thread (although not recommended) and wait for user inputs, it's certainly not a good idea to create a view to subscribe to the event of some background thread codes.

Event if we put asynchronous interaction aside, the `InteractionRequest` idea still has one drawback often overlooked by people——easy to cause memory leaks. Imaging in a `XMAL` that composed of several reusable controls of the same type, if the control wrapped `InteractionRequest`, then the `Raised` event can be hooked up several times without any warnings (usually we don't limit number of subscribers of a event), we might end up seeing the same dialog being displayed, closed and displayed again because the event is listened by more than one subscribers.

To fix the problem usually a boolean property needs to be added to the reusable control to enable/disable the `InteractionRequestTrigger` so that only one is enabled in the case of multiple occupance of the same type of control.

The [Prism](https://prismlibrary.com) team also realized the [problems of InteractionRequest](https://github.com/PrismLibrary/Prism/issues/864), its current key contributor [Brian Lagunas](https://github.com/brianlagunas) even frankly says that he does not like it, and he proposed a [new Dialog Service](https://github.com/PrismLibrary/Prism/issues/1666) on `2019`, it's now part of the `Prism` [official document](https://prismlibrary.com/docs/wpf/dialog-service.html). However, in my opinion this so called `new approach` is nothing but the `Dialog Service` discussed earlier in this blog, the only improvement is that the method parameters are replaced by a `IDialogParameters` interface, it still does't support asynchronous interaction and modeless dialogs.


## 3. AsyncInteraction

Let's take a step back and review the requirement again, it's indeed `display a dialog, wait for user inputs and get the result`, sounds familiar? Yes, it's exactly same as `start a task, wait for its completion and get the result`. So now since we have clarified the concept, the solution is already right there.

First let's define a generic interface `IAsyncInteraction<T>`, it has only one method `StartAsync` that accepts one parameter and returns a `Task`，`AsyncInteractionArgs` can be used to pass any number of parameter for the sake of flexibility。

```c#
public interface IAsyncInteraction<T> where T : class
{
  Task<AsyncInteractionResult<T>> StartAsync(AsyncInteractionArgs args);
}

public sealed class AsyncInteractionArgs : ReadOnlyCollection<object>
{
  public AsyncInteractionArgs(params object[] args) : base(args)
  {
  }
}

public class AsyncInteractionResult<T> where T : class
{
  public readonly bool IsConfirmed;
  
  public readonly T Result;

  public AsyncInteractionResult(bool isConfirmed, T result)
  {
    IsConfirmed = isConfirmed;
    Result = result ?? throw new ArgumentNullException(nameof(result));
  }
}
```

Obviously the use of generic type `<T>` makes dependency injection possible. For example, in a `ViewModel` that needs to interact with a `MessageBox` and a `LoginDialog`, we can have two properties injected like below, the properties would be easy to mock in unit tests, it also clearly state the dependencies of the `ViewModel` (compare to `IDialogService`, which is more general).

```c#
[Depencency]
public IAsyncInteraction<MessageBoxData> MessageBox { get; set; }
    
[Depencency]
public IAsyncInteraction<ILoginViewModel> LoginBox { get; set; }
```

To use this interface, we just start a `Task`: pass the parameter via the `AsyncInteractionArgs`, the implementation of the interface then switch to its specified `GUI` thread (main GUI or secondary thread), create and render the window based on the parameter, set owner window and display the dialog. When the modal dialog is closed (or when the modeless dialog completes operation), set result to the `Task` so the caller can continue.

Below is an example of displaying a `MessageBox` in the async/await fashion:

```c#
private async Task<AsyncInteractionResult<MessageBoxData>> ShowMessageAsync()
{
  var data = CreateMessageBoxData();
  var args = new AsyncInteractionArgs(data);
  return MessageBox.StartAsync(args);
}
```

Now we can say with confidence that the problems listed in the beginning of this blog are solved:

+ **Separation** **of** **View** **and** **Data**——The `ViewModel` only cares about the interface, how the view is created/rendered is completely hidden in the implementation. Mean while, developers have maximum freedom to define the contract between the view and data.
+ **Easy** **to** **test**——Mocking interfaces and testing `Tasks` are easy.
+ **Asynchronous** **interaction**——`Task` is invented for asynchronous operations, so it's easy to handle even if we want do display some UI in a background thread and wait for user inputs.
+ **Unified** **programming** **model**——Since the caller only cares about state of the `Task`, so the difference between modal or modeless dialog does not matter anymore and can be handled in the same way.
+ **Compare** **to** **`Dialog Service`**——Generic interface is type safe and more flexible, we also don't have to force the parameters implement things like `IDialogParameters`, dependencies are also clearer.
+ **Compare** **to** **`InteractionRequest`**——Nor more event subscribers and not more memory leaks because of multiple event subscribers.


## 4. Conclusion

Please visit [github](https://github.com/eagleboost/AsyncInteraction) for full details and sample codes。To simplify discussions, all codes above is based on`.net 5`, but the principle of the `IAsyncInteraction<T>` interface does not limit to `.net 5`. Similar implementation can be done as long some `Task` like thing is supported, no matter it's `WPF` or `WinForm`, `.net Core` or `.net Framework`, even any other programing languages.