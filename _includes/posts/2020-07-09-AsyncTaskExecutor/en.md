## 1. The Problem

During application development we often see some patterns can be reused. For example, in a `WPF` application we send some async calls to a background `Task` to execute without blocking the main GUI, when the  `Task` is completed we then update the result on the main GUI thread. The process of starting a task and wait for its finish can be a good candidate of a pattern to encapsulate: Show name of the task and feedback during execution of the task etc.

Let's take below case as an example: say we have a button, click it to start a task to retrieve data from some remote service. In the mean time GUI should also be updated, developers often set the button to disabled and enable it back after the task is completed. But a more optimized design can provide richer UI to show the task execution states in-place as well as interaction abilities:

+ Display the name of the task
  
+ Show busy status to indicate there's something going on
  
+ Show progress if the task reports status like % of completion to manage user expectations
  
+ Allow user cancel the task

+ Optionally, allow user pause and resume the task based on the use case

+ Handle task exceptions gracefully


![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/AsyncTaskExecutor_Load.png)

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/AsyncTaskExecutor_Executing.png)

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/AsyncTaskExecutor_Paused.png)


## 2. Analysis

The `IAsyncTaskExecutor` interface below is the abstraction of the async task executions.

```c#
public interface IAsyncTaskExecutor : IBusyStatus
{  
  string TaskName { get; }
    
  AsyncTaskExecutionStatus Status { get; }

  Task ExecuteTask(object parameter);

  void CancelTask();

  void PauseTask();
    
  void ResumeTask();
  
  event EventHandler Started;
    
  event EventHandler<ExecutionErrorEventArgs> Faulted;
    
  event EventHandler Executed;
}
```

`.net` provides built-in support for task cancellation via `CancellationTokenSource`, but does not support pause/resume tasks, probably because pause/resume is not used that often.[*Stephen Toub*](https://devblogs.microsoft.com/pfxteam/author/toub/) demonstrated an elegant pause/resume implementation `PauseTokenSource` in his post [*Cooperatively pausing async methods*](https://blogs.msdn.microsoft.com/pfxteam/2013/01/13/cooperatively-pausing-async-methods/), we'll use it in this article to support pause/resume.


For the purpose of Binding we also abstract an `IAsyncTaskComponent` interface.

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

For the UI feedback of the task execution, it can be very different case by case, in this article we only focus on an in-place UI feedback, aka show an overlay on top of the button with the task execution status.

## 3. Implementations

`AsyncTaskExecutor` does all the jobs of start an task and cancel/pause/resume the task, some of the key parts are shown below:

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

`AsyncTaskComponent<T>` in turn creates an instance of `AsyncTaskExecutor<T>` or accept a custom implementation of `IAsyncTaskExecutor` in the constructor, then expose the API of `IAsyncTaskExecutor` as Commands. Please note that `AsyncTaskComponent<T>` also contains a reference to `IProgress`, it's not necessary but only to make data binding easier in the `DataTemplate`.

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

For complete implementations, please visible [*github*](https://github.com/eagleboost/AsyncTaskExecutor)

## 4. Usages

Let's revisit the example presented in the beginning of the article, the `ViewModel` below shows the usages of `AsyncTaskComponent` and `AsyncTaskExecutor`. The client codes only need to provider name of the task and an async methods that matches below signature:

```c#
Func<T, TaskExecutionOption, Task> taskFunc
```

`AsyncTaskComponent` and `AsyncTaskExecutor` would do the rest of the jobs. `TaskExecutionOption` contains context information including the task name, `CancellationTokenSource` and `PauseTokenSource` for the async method to use。

```c#
public class ViewModel : NotifyPropertyChangedBase
{
  private readonly ProgressViewModel _loadProgress = new ProgressViewModel();

  public ViewModel()
  {
    AsyncLoad = new AsyncTaskComponent<string>("Loading items", ExecuteAsync) {Progress = _loadProgress};
  }
  
  public IAsyncTaskComponent AsyncLoad { get; }
  
  ////Query if the task is paused in each loop and report progress every one second
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

In the `XAML`, we bind the button's `Command` property to `IAsyncTaskComponent.ExecuteCommand`, and created an `Attached Behavior` called `AsyncTaskUi` to handle the overlay of the button, i.e. `AsyncTaskUi` listens to the events of `IAsyncTaskExecutor`, add overlay to the button's `AdornerLayer` when `Started` is triggered and remove overlay from the `AdornerLayer` when `Executed` is triggered.

`AsyncTaskUi` also accepts a `DataTemplate` to allow customize template.

```xml
<Button Command="{Binding AsyncLoad.ExecuteCommand}"
        Width="150" Height="30" HorizontalAlignment="Center" Content="Load">
  <b:Interaction.Behaviors>
    <controls:AsyncTaskUi AsyncTaskComponent="{Binding AsyncLoad}" Template="{StaticResource AsyncTaskTemplate}"/>
  </b:Interaction.Behaviors>
</Button>
```


## References
+ [*Cooperatively pausing async methods*](https://blogs.msdn.microsoft.com/pfxteam/2013/01/13/cooperatively-pausing-async-methods/)