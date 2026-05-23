# 幽灵闭包：为什么 'if' 没进去，'c_DisplayClassX_0' 还是分配了

> 一个关于 WPF 跨线程诊断补丁的实战故事：一个*本不该发生*的分配——直到你读了IL。

# 太长不看版

　　`Roslyn`把闭包对象（`c_DisplayClassX_0`）分配在**最外层被捕获变量的作用域**，而不是`lambda`的词法位置。

　　如果`lambda`捕获了方法的**参数**（或者作用域覆盖整个方法的模式匹配变量），`newobj` 会被生成到方法**序言**中。于是当闭包所在函数每次被调用时，即使闭包调用位于`if`语句的`body`里面，闭包还是会被无条件创建，即使包含`lambda`的分支从未进入。

　　修复很简单：把`lambda`挪到一个独立的`helper`方法里，让所有被捕获的变量都在`helper`的作用域内。于是函数调用就不会产生任何内存分配。

## 背景

　　我在项目中写了一个`hack`类，用`Harmony`对`DependencyObject.GetValue`打补丁，当有代码从错误的 `dispatcher`线程读取`DP`时打印`Visual Tree`从而知道是哪个资源的访问出了问题：

```csharp
private static void TryPrintVisualTreeInfo(DependencyObject obj, DependencyProperty dp)
{
  if (obj is FrameworkElement element && !element.CheckAccess())
  {
    var thread = Dispatcher.CurrentDispatcher.Thread.Name;
    element.BeginInvoke(() => PrintVisualTreeInfo(element, thread, dp));
  }
}
```

　　代码看起来人畜无害。`lambda`在`if`里面，所以我们的预期是：

* 在正确线程上调用时 => `CheckAccess()` 返回 `true` => 零分配。
* 跨线程访问时 => 1 个闭包 + 1 个委托 + 1 个`DispatcherOperation`。

　　然而对程序做内存性能分析则发现不是那么回事：

```
new WPFPatch.c_DisplayClassX_0()   ← 每秒数千次
WPFPatch.TryPrintVisualTreeInfo()
WPFPatch.DependencyObjectGetValue.Prefix()
[Lightweight Method Call]
Timeline.get_AccelerationRatio()
Clock.ComputeIntervalsWithParentIntersection()
...
TimeManager.Tick()
MediaContext.RenderMessageHandlerCore()
```

　　每一次动画`Tick`都在创建新的闭包——但`BeginInvoke`的回调从未执行。我们在`c_DisplayClassX_0..ctor`上打断点也确认：闭包在 `if` 的`body`明显被跳过的情况下仍然被分配了。

　　什么原因呢？

---

## 还得是IL

　　反汇编方法后一眼就明了了：

```
.method private hidebysig static void TryPrintVisualTreeInfo(
  [WindowsBase]System.Windows.DependencyObject obj,
  [opt] [WindowsBase]System.Windows.DependencyProperty dp)
{
  .locals init ([0] WPFPatch/c_DisplayClass4_0 'cs$<>8_locals0')

  //  METHOD PROLOG - runs every call ===
  IL_0000: newobj     instance void WPFPatch/<>>c_DisplayClassX_0::.ctor()
  IL_0005: stloc.0
  IL_0006: ldloc.0
  IL_0007: ldarg.1                      // dp
  IL_0008: stfld      ...::dp           // 立即捕获到闭包

  // === if (obj is FrameworkElement element && !element.CheckAccess()) ===
  IL_000d: ldloc.0
  IL_000e: ldarg.0                      // obj
  IL_000f: isinst     FrameworkElement
  IL_0014: stfld      ...::element      // 模式变量也被捕获到这里
  IL_0019: ldloc.0
  IL_001a: ldfld      ...::element
  IL_001f: brfalse.s  IL_005b           // null => 返回
  IL_0021: ldloc.0
  IL_0022: ldfld      ...::element
  IL_0027: callvirt   CheckAccess()
  IL_002c: brtrue.s   IL_005b           // 在正确线程上 => 返回

  // = if 的 body =
  IL_002e: ldloc.0
  IL_002f: call       Dispatcher::get_CurrentDispatcher
  IL_0034: callvirt   Dispatcher::get_Thread
  IL_0039: callvirt   Thread::get_Name
  IL_003e: stfld      ...::thread
  IL_0043: ldloc.0
  IL_0044: ldfld      ...::element
  IL_0049: ldloc.0
  IL_004a: ldftn      ...::`<TryPrintVisualTreeInfo>b__0
  IL_0050: newobj     Action:: .ctor
  IL_0055: call       DispatcherObjectExtensions::BeginInvoke
  IL_005a: pop
  IL_005b: ret
}
```

三个关键点：

1. `c_DisplayClassX_0` 的 `newobj` 在 **IL_0000**——方法的第一条指令。
2. 在`if`语句被执行之前`dp` 就已经被存入闭包（**IL_0008**）。
3. `element` 模式变量*也*是闭包上的一个字段（`...::element`），在 **IL_0014** 写入，同样在 `CheckAccess` 防护之前。

　　所以闭包对象在每一次调用中都会实例化。`if` 只是兜住了**委托创建**和 `BeginInvoke`，这就是为什么`PrintVisualTreeInfo`从未运行，但内存分配量仍然飙升。

# Roslyn 为什么这么做

　　`Roslyn`有一个优化：把闭包分配放在**覆盖所有被捕获变量和引用它们的`lambda`的最内层块**。意图很明确：等控制流真正进入需要闭包的作用域再分配。

　　很符合我们的直觉，但是这个优化只在**所有**捕获都处在在同一个内部块时才触发。一旦有一个捕获的作用域更宽，闭包就必须在更宽的那个作用域分配——否则被捕获的字段在源变量可见的整个区间内就不是可靠的了。

　　在我们的方法里，`lambda`捕获了三个局部量：

|捕获 | 声明位置 | 作用域 |
|--------|-----------------------------|--------|
|`dp` | 方法参数 | 整个方法 |
|`element` | `obj is FrameworkElement element` 中的模式匹配变量 | 整个方法（\*） |
|`thread`| `var thread = ...` 在 `if` 内部 | `if` 内部 |

　　（\*）`C# 7+` 中，`if` 条件里声明的模式匹配变量，当与 `&&` 等短路运算符一起使用时，作用域会**延伸到 `if` 之外**，被视为属于外层块而非 `if` body。就闭包作用域而言，编译器将其当作一个对整个方法可见的方法局部变量。

　　因为 `dp` 和 `element` 在整个方法中可见，`c_DisplayClassX_0` 被提升到了方法作用域。`thread` 虽然作用域很窄，不过是稍后多赋一个字段——它没法把分配点拉回来。

　　最终结果：**只要有一个方法级作用域的捕获，就足以让其他所有捕获的优化全部失效。**

---

## 为什么症状这么严重

　　被补丁的方法会被`WPF`媒体上下文的动画`Tick`从`Timeline.get_AccelerationRatio` 调起。渲染线程上，每个活跃的时间线每秒大约`60`次调用，更不用说`DependencyObject.GetValue`几乎每时每刻都在被调用。

　　每次调用：

* 分配一个 `c_DisplayClassX_0`。
* 在跨线程路径上，还要分配一个 `Action` 并入队一个`DispatcherOperation`到外部`dispatcher`。

　　于是在股市关市的那一刻，大量订单涌入系统，代码的性能没有问题，但是内存分配却暴增，导致`GC`把全部线程停住开始垃圾回收，交易员暴跳如雷咆哮着喊“又卡住了”。

---

## 修复

　　把方法拆开，把`lambda`以及所有需要捕获的变量都移动到一个`helper`里。这样就没有任何捕获的作用域比`helper`更宽，`Roslyn`的优化就能把`newobj`放在我们所期望的位置。

## 修改前

```csharp
private static void TryPrintVisualTreeInfo(DependencyObject obj, DependencyProperty dp)
{
  if (obj is FrameworkElement element && !element.CheckAccess())
  {
    var thread = Dispatcher.CurrentDispatcher.Thread.Name;
    element.BeginInvoke(() => PrintVisualTreeInfo(element, thread, dp));
  }
}
```

## 修改后

```csharp
private static void TryPrintVisualTreeInfo(DependencyObject obj, DependencyProperty dp)
{
  if (obj is FrameworkElement element && !element.CheckAccess())
  {
    DispatchPrintVisualTreeInfo(element, dp);
  }
}

private static void DispatchPrintVisualTreeInfo(FrameworkElement element, DependencyProperty? dp)
{
  var thread = Dispatcher.CurrentDispatcher.Thread.Name;
  element.BeginInvoke(() => PrintVisualTreeInfo(element, thread, dp));
}
```

　　`TryPrintVisualTreeInfo`现在没有`lambda`，也没有捕获 => 热路径上不再有`DisplayClass`分配。`DispatchPrintVisualTreeInfo`每次调用仍然分配一个闭包，但它只在跨线程条件真正成立时才运行——这正是我们一开始想要的效果。

# 核心要点

1. **lambda 的词法位置不等于其闭包分配的词法位置。** `Roslyn`根据捕获的作用域来放置`newobj`。
2. **lambda 只要捕获了方法参数，闭包就一定会被分配在方法入口**，无论`lambda`嵌套多深。
3. **`if (x is T y && ...)` 中的模式变量在闭包作用域上参与的是方法级作用域**——捕获它们和捕获参数有相同的提升效应。
4. **热路径绝不能让`lambda`捕获方法级变量。**如果慢路径需要一个`lambda`，就把`lambda`和慢路径的捕获都移到一个单独的方法里。快路径保持零分配。
5. **信 IL，别信源码。** 当分配分析器和你的心智模型打架时，反编译那个方法。花两分钟看一眼`IL`序言，`newobj` 在哪、为什么在那，一目了然。

## 怎么在自己的代码里发现这个问题

　　代码审阅时的启发：

> 如果一个`guard`里的`lambda`捕获了**外层方法的参数**（或者`guard`自身的模式匹配变量），并且外层方法在热路径上，那就重构它。

　　或者更机械一点：在反编译器里打开文件，看方法的 `.locals init` 和前几条`IL`指令。如果你在 `IL_0000`看到了 `newobj ...<>c_DisplayClassN_M::.ctor()`，这个闭包就是无条件的。让它变成有条件的方式只有一个：把`lambda`挪进`helper`。
