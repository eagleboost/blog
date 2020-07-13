## 1. 问题

在应用程序开发的过程中注意观察的话总会发现一些通用的模式可以选择封装起来重用。比如在`WPF`应用程序中我们常常把一些需要异步调用的工作交给某个后台`Task`去执行以便不阻塞主线程，等到`Task`结束后再回到主线程来更新状态。这个从开始执行一个任务到等待其结束的过程就是一个很好的封装对象：执行的任务有一个名字，在执行的过程中显示某种反馈等等。

具体可以用下面的例子来说明：假设有一个按钮，点击按钮开始调用某个远程服务获取数据，同时界面发生相应变化，通常开发者会把按钮设置为失效状态，任务结束后再恢复。一个更优质的设计可以在按钮失效后`就地`显示任务执行状态并提供丰富的交互：

+ 显示当前执行的任务名
  
+ 显示繁忙动画告知用户有任务在执行
  
+ 如果该任务有反馈，比如完成的百分比进度，也应显示进度以便管理用户的心理预期
  
+ 允许用户取消任务

+ 根据实际情况，允许用户暂停任务

+ 处理异步任务中发生的异常


![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/AsyncTaskExecutor_Load.png)

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/AsyncTaskExecutor_Executing.png)

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/AsyncTaskExecutor_Paused.png)


## 2. 分析

对异步任务执行的封装大致可以抽象出下面的`IAsyncTaskExecutor`接口。

```c#
public interface IAsyncTaskExecutor : IBusyStatus
{  
  string TaskName { get; } ////任务名称
    
  AsyncTaskExecutionStatus Status { get; } ////当前执行状态

  Task ExecuteTask(object parameter); ////执行任务

  void CancelTask(); ////取消任务

  void PauseTask(); ////暂停任务
    
  void ResumeTask(); ////恢复任务
  
  event EventHandler Started; ////任务开始后触发
    
  event EventHandler<ExecutionErrorEventArgs> Faulted; ////任务出错后触发
    
  event EventHandler Executed; ////任务结束后触发
}
```

`.net`对于取消任务通过`CancellationTokenSource`提供内建支持。对于暂停/恢复任务，由于使用场景不多并没有内建支持。[*Stephen Toub*](https://devblogs.microsoft.com/pfxteam/author/toub/)在这篇博客[*Cooperatively pausing async methods*](https://blogs.msdn.microsoft.com/pfxteam/2013/01/13/cooperatively-pausing-async-methods/)给出了一个优雅的实现`PauseTokenSource`，本文对于暂停/恢复任务的支持即来源于此。


为方便数据绑定再定义一个`IAsyncTaskComponent`接口，

```c#
public interface IAsyncTaskComponent
{
  IAsyncTaskExecutor Executor { get; }

  IProgress Progress { get; }

  ICommand ExecuteCommand { get; }
  
  ICommand CancelCommand { get; }

  ICommand PauseResumeCommand { get; }
}
```

至于任务执行的界面反馈，不同的场景有不同的方式，本文只给出就地显示状态执行反馈的实现。

## 3. 实现

`AsyncTaskExecutor`是执行任务的核心，其关键代码摘录如下：

```c#
public class AsyncTaskExecutor<T> : NotifyPropertyChangedBase, IAsyncTaskExecutor
{
  private CancellationTokenSource _cts;
  private readonly PauseTokenSource _pts = new PauseTokenSource();

  public Task ExecuteAsync(object parameter)
  {
    var ct = RefUtils.Reset(ref _cts);
    ct.Register(OnCanceled);
    
    SetStatus(AsyncTaskExecutionStatus.Started);
    var option = new TaskExecutionOption(TaskName, ct, _pts.Token);
    return ExecuteAsyncCore(option, CreateExecuteTask(parameter, option));
  }
  
  public void CancelTask()
  {
    RefUtils.Cancel(ref _cts);
  }
  
  public void PauseTask()
  {
    _pts.IsPaused = true;
    SetStatus(AsyncTaskExecutionStatus.Paused);
  }
  
  public void ResumeTask()
  {
    _pts.IsPaused = false;
    SetStatus(AsyncTaskExecutionStatus.Resumed);
  }
  
  private async Task ExecuteAsyncCore(TaskExecutionOption option, Task task)
  {
    try
    {
      RaiseStarted();
      SetBusy(option.TaskName + "...");

      await task.ConfigureAwait(false);

      SetBusy("Completed");

      await Task.Delay(800).ConfigureAwait(false);
      
      SetStatus(AsyncTaskExecutionStatus.Completed);
    }
    catch (Exception ex)
    {
      SetStatus(AsyncTaskExecutionStatus.Faulted);
      RaiseFaulted(ex);
    }
    finally
    {
      ClearBusy();
      RaiseExecuted();
    }
  }

  private Task CreateExecuteTask(object parameter, TaskExecutionOption option)
  {
    return _taskFunc((T) parameter, option);
  }
  
  private void SetStatus(AsyncTaskExecutionStatus status)
  {
    Status = status;
  }

  private void OnCanceled()
  {
    _pts.IsPaused = false;
    ClearBusy();
    SetStatus(AsyncTaskExecutionStatus.Canceled);
  }
  
  private void SetBusy(string status)
  {
    IsBusy = true;
    BusyStatus = status;
  }

  private void ClearBusy()
  {
    IsBusy = false;
    BusyStatus = null;
  }
}
```

`AsyncTaskComponent<T>`创建一个`AsyncTaskExecutor<T>`的实例或接受一个自定义的`IAsyncTaskExecutor`实现，并把`IAsyncTaskExecutor`的`API`暴露为`Command`供绑定使用，注意`AsyncTaskComponent<T>`也包含了对`IProgress`的引用，并非必须，只是方便`DataTemplate`中的绑定使用。

```c#
public class AsyncTaskComponent<T> : NotifyPropertyChangedBase, IAsyncTaskComponent
{
  public AsyncTaskComponent(IAsyncTaskExecutor executor, Func<T, bool> canExecute = null)
  {
    ...
  }
  
  public AsyncTaskComponent(string taskName, Func<T, TaskExecutionOption, Task> taskFunc, Func<T, bool> canExecute = null)
  {
    ...
  }
  
  public IAsyncTaskExecutor Executor { get; }

  public IProgress Progress { get; }

  public ICommand ExecuteCommand => _executeCommand ??= CreateExecuteCommand();

  public ICommand CancelCommand => _cancelCommand ??= CreateCancelCommand();

  public ICommand PauseResumeCommand => _pauseResumeCommand ??= CreatePauseResumeCommand();
}
```

完成实现请移步[*github*](https://github.com/eagleboost/AsyncTaskExecutor)

## 4. 使用简介

就本文开篇给出的例子，假设有个`ViewModel`，如下实现演示了如何使用`AsyncTaskComponent`及`AsyncTaskExecutor`。客户端代码所需提供的只是任务名称和一个签名如下的异步方法。

```c#
Func<T, TaskExecutionOption, Task> taskFunc
```

其余的任务则完全由`AsyncTaskComponent`和`AsyncTaskExecutor`完成。`TaskExecutionOption`则包含了上下文信息，包括任务名称，`CancellationTokenSource`和`PauseTokenSource`以便异步方法使用。

```c#
public class ViewModel : NotifyPropertyChangedBase
{
  private readonly ProgressViewModel _loadProgress = new ProgressViewModel();

  public ViewModel()
  {
    AsyncLoad = new AsyncTaskComponent<string>("Loading items", ExecuteAsync) {Progress = _loadProgress};
  }
  
  public IAsyncTaskComponent AsyncLoad { get; }
  
  ////循环100次，每次循环开始查询任务是否被暂停，每间隔一秒报告一次进度。
  private async Task ExecuteAsync(string parameter, TaskExecutionOption option)
  {
    var progress = _loadProgress;
    progress.Report(0);
    
    for (var i = 0; i < 100; i++)
    {
      var pt = option.PauseToken; 
      await pt.WaitWhilePausedAsync();
      if (!option.CancellationToken.IsCancellationRequested)
      {
        await Task.Delay(1000, option.CancellationToken);
        progress.Report(i * 1);
      }
    }
  }
}
```

在`XAML`中一方面按钮的`Command`绑定到`IAsyncTaskComponent.ExecuteCommand`，另一方面通过一个名为`AsyncTaskUi`的`Attached Behavior`为按钮添加和删除用于显示任务执行反馈的蒙版，也就是响应`IAsyncTaskExecutor`的事件，当`Started`触发时在按钮的`AdornerLayer`中显示蒙版，`Executed`触发时从按钮的`AdornerLayer`中删除蒙版。

从灵活性角度考虑`AsyncTaskUi`还接受一个`DataTemplate`用于生成任务执行反馈的界面。

```xml
<Button Command="{Binding AsyncLoad.ExecuteCommand}"
        Width="150" Height="30" HorizontalAlignment="Center" Content="Load">
  <b:Interaction.Behaviors>
    <controls:AsyncTaskUi AsyncTaskComponent="{Binding AsyncLoad}" Template="{StaticResource AsyncTaskTemplate}"/>
  </b:Interaction.Behaviors>
</Button>
```


## 参考资料
+ [*Cooperatively pausing async methods*](https://blogs.msdn.microsoft.com/pfxteam/2013/01/13/cooperatively-pausing-async-methods/)