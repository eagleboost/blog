---
layout:     post
title:      "Prism Splash Screen Module"
date:        2011-06-02 20:00:00
author:     "eagleboost"
header-img: "img/post-bg-gloden-bridge.jpg"
catalog:    true
tags:
  - Prism 
  - User Experience
  - Splash Screen
  - WPF高级编程
  - Advanced WPF
  - .Net
---

> 本文转载自[我2011年在wordpress.com发布的博客](https://eagleboost.wordpress.com/2011/06/02/prism-splash-screen-module/)

`Splash screen` is a good thing. It makes your application looks professional while numbers of libraries are being loaded in the background, mean while, users may feel the loading time is actually not that long if they see some status updates. So the way we designing a splash screen is indeed very important in order to get smooth user experience.

There’re various ways you can find everywhere to implement a splash screen but very few of them meet my criteria of good ones:

+ A splash screen should provide status updating to prove the application is not hung
+ A splash screen should show as the first visual of the application, especially, the main window should only be visible after the splash screen is gone
+ A splash screen should be interactable – Some applications like to show splash screen as topmost – users can’t bring their other stuff to front if it’s overlapped by the splash; A lot of applications’ splash screen is not moveable, even worse if it's topmost; Some applications has status updates on the splash screen but when you click on it, Windows tends to tell you the application window is busy and in some cases window would be frozen and not responding.
+ A better splash screen should give user more options: either allow them to close it (by double clicking it for example), or allow them drag and move the splash screen around without freeing the window if user still want to see the status updating

This blog will introduce one neat approach of implementing a splash screen that meet all of the above criteria, in a `MVVM` manner, in `Prism`. Prism is an excellent application framework that makes applications modulized naturally, therefore it’s also an intuitive thinking that we can make splash screen a module, so it’s independent to other modules and can be easily reused in future applications.

Here's the implementation details.

Usually in the unity `bootstrapper`, we create the shell and show it, like this:

```c#
public class Bootstrapper : PrismBootstrapper
{
  protected override DependencyObject CreateShell()
  {
    var shell = Container.Resolve<Shell>();
    shell.Show();
    return shell;
  }
}
```

The first thing to do is creating the shell but not showing it – instead we let the splash screen module to do that (through the `Show()` method of the `IShell` interface) as we don’t want the shell to be visible before all module loadings are finished.

The `Shell`, aka the `MainWindow` should be registered to the `IShell` interface as a singleton, the BootStrapper would be like this:

```c#
public class Bootstrapper : PrismBootstrapper
{
  protected override void RegisterTypes(IContainerRegistry containerRegistry)
  {
    containerRegistry.RegisterSingleton<IShell, Shell>();
  }

  protected override DependencyObject CreateShell()
  {
    var shell = Container.Resolve<IShell>();
    return (DependencyObject)shell;
  }
}
```

Next step is the splash. Double click to close and drag move is easy, two lines of code like below will do.

```c#
splash.MouseDoubleClick += (s_, e_) => splash.Close();
splash.MouseLeftButtonDown += (s_, e_) => splash.DragMove();
```

Hard part is the interactivity. If we create the splash screen on the GUI thread, it might not be as responsive as expected when loading modules simply because the GUI thread has no enough bandwidth. The trick is moving the splash screen off the main GUi thread to a secondary STA thread:

1. Create the shell but don’t show it
2. Start the splash screen module. The splash screen module will queue a request on the Shell’s dispatcher, create a new STA thread to host the splash screen and subscribe to the MessageUpdateEvent.
5. Initialize other modules and publish the status through the MessageUpdateEvent so the splash screen can show it
6. After the initialization is done, the request queued on the dispatcher gets called, then the Shell shows up, the splash screen is closed and the hosting STA thread is shut down.

Please visit [github](https://github.com/eagleboost/PrismSplashScreen) for sample project.