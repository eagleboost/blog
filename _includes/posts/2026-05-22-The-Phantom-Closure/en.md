# The Phantom Closure: Why 'c_DisplayClassX_0' Is Allocated Even When Your 'if' Is False

> A war story about a WPF cross-thread diagnostic patch, an animation tick that
> ran 60 times per second, and an allocation that *shouldn't have happened* -
> until you read the IL.

# TL;DR
Roslyn allocates the closure object (`c_DisplayClassX_0`) at the **scope of the outermost captured variable**, not at the lexical position of the lambda.

If a lambda captures a method **parameter** (or a pattern variable whose scope spans the entire method), the `newobj` is emitted in the method **prolog** before any `if` guard. The closure is therefore created on every call, unconditionally, even when the branch containing the lambda is never entered.

The fix is mechanical: move the lambda into a separate helper method so the only captured variables live in that helper's scope. The hot path then becomes allocation-free.

## The setup

We had a WPF "hack" class that uses Harmony to patch `DependencyObject.GetValue` and emit a diagnostic whenever a DP is read from the wrong dispatcher thread:

```csharp
private static void TryPrintVisualTreeInfo(DependencyObject obj, DependencyProperty? dp = null)
{
  if (obj is FrameworkElement element && !element.CheckAccess())
  {
    var thread = Dispatcher.CurrentDispatcher.Thread.Name;
    element.BeginInvoke(() => PrintVisualTreeInfo(element, thread, dp));
  }
}
```

Looks innocent. The lambda is inside the `if`, so we expected:

* When the call is on the right thread => `CheckAccess()` returns `true` => no allocation.
* When it's a cross-thread access => 1 closure + 1 delegate + 1 `DispatcherOperation`.


In production, the allocation profiler told a different story:

```
new WPFPatch.c_DisplayClassX_0()   ← thousands per second
WPFPatch.TryPrintVisualTreeInfo()
WPFPatch.DependencyObjectGetValue.Prefix()
[Lightweight Method Call]
Timeline.get_AccelerationRatio()
Clock.ComputeIntervalsWithParentIntersection()
...
TimeManager.Tick()
MediaContext.RenderMessageHandlerCore()
```

Every animation tick was minting a fresh closure - but the `BeginInvoke` callback never ran. We confirmed via a breakpoint on `c_DisplayClassX_0..ctor` that the closure was being allocated even though the body of the `if` was apparently skipped.

How?

---

## The IL doesn't lie

Setting a method breakpoint on the closure's constructor stopped execution at the `if` line. The disassembly of the method clarifies why:

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
  IL_0007: ldarg.1                    // dp
  IL_0008: stfld      ...::dp           // captured into closure immediately
  
  // === if (obj is FrameworkElement element && !element.CheckAccess()) ===
  IL_000d: ldloc.0
  IL_000e: ldarg.0                    // obj
  IL_000f: isinst     FrameworkElement
  IL_0014: stfld      ...::element      // pattern var also captured here
  IL_0019: ldloc.0
  IL_001a: ldfld      ...::element
  IL_001f: brfalse.s  IL_005b          // null => return
  IL_0021: ldloc.0
  IL_0022: ldfld      ...::element
  IL_0027: callvirt   CheckAccess()
  IL_002c: brtrue.s   IL_005b           // on right thread => return

  // = Body of the if =
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

Three things to notice:

1. The `newobj` for `c_DisplayClassX_0` is at **IL_0000** - the very first
instruction of the method.
2. `dp` is stored into the closure immediately (**IL_0008**), before any check.
3. The `element` pattern variable is *also* a field on the closure
(`...::element`), and gets written at **IL_0014**, also before the
`CheckAccess` guard.

So the closure object materialises on every single call. The `if` only gates the **delegate creation** and the `BeginInvoke`, which is why `PrintVisualTreeInfo` never runs but allocations still climb.


# Why Roslyn does this

Roslyn has an optimisation that places the closure allocation at the **innermost block that encloses every captured variable and every lambda that references them**. The intent is the obvious one: don't allocate the closure until control flow has actually entered a scope that needs it.

That optimization only fires when **all** captures live inside the same inner block. As soon as one capture has wider scope, the closure must be allocated at that wider scope - otherwise the captured field wouldn't be live for the whole region in which the source variable is visible.

In our method, the lambda captures three locals:

|Capture | Where it's declared         |   Scope|
|--------|-----------------------------|--------|
|`dp` |  method parameter           | the whole method|
|`element` |pattern variable in `obj is FrameworkElement element` | the whole method (\*) |
|`thread`| `var thread = ...` inside the `if`        | inside the `if`|

(\*) A C# 7+ pattern variable declared in an `if` condition has a scope that **extends past the `if`** when used with `&&` and similar short-circuiting operators, and is considered to belong to the enclosing block, not the `if` body. For the purposes of closure scoping, the compiler treats it as a method local visible to the whole method.

Because `dp` and `element` are visible across the whole method, `c_DisplayClassX_0` is hoisted to the method scope. `thread`, even though narrowly scoped, is just an extra field assigned later - it doesn't pull the allocation back down.

Net result: **one capture with method-wide scope is enough to defeat the
optimisation for every other capture.**

---

## Why the symptom was so bad

The patched method `DependencyObjectGetValue.Prefix` is invoked from `Timeline.get_AccelerationRatio` on the WPF media context's animation tick. That's ~60 calls per second, per active timeline, on the render thread.

Each invocation:

* Allocated one `c_DisplayClassX_0`.
* On cross-thread paths, also allocated one `Action` and queued one
`DispatcherOperation` to a foreign dispatcher.

At market close, a flood of orders poured into the system. The code's throughput was fine, but memory allocations spiked, causing the GC to pause all threads for garbage collection. The traders were furious, shouting "It's frozen again!"

---

## The fix

Split the method so the lambda lives in a helper whose only locals *are* the captures. Then no capture has scope wider than the helper, and Roslyn's optimisation can place the `newobj` where you'd naively expect.

## Before

```csharp
private static void TryPrintVisualTreeInfo(DependencyObject obj, DependencyProperty? dp = null)
{
  if (obj is FrameworkElement element && !element.CheckAccess())
  {
    var thread = Dispatcher.CurrentDispatcher.Thread.Name;
    element.BeginInvoke(() => PrintVisualTreeInfo(element, thread, dp));
  }
}
```

## After

```csharp
private static void TryPrintVisualTreeInfo(DependencyObject obj, DependencyProperty? dp = null)
{
  if (obj is FrameworkElement element && !element.CheckAccess())
  {
    QueuePrintVisualTreeInfo(element, dp);
  }
}

private static void QueuePrintVisualTreeInfo(FrameworkElement element, DependencyProperty? dp)
{
  var thread = Dispatcher.CurrentDispatcher.Thread.Name;
  element.BeginInvoke(() => PrintVisualTreeInfo(element, thread, dp));
}
```

`TryPrintVisualTreeInfo` now has no lambda and no captures => no DisplayClass allocation on the hot path. `QueuePrintVisualTreeInfo` still allocates one closure per call, but it only runs when the cross-thread condition is actually true - which is what we wanted all along.

# Take-aways

1. **The lexical position of a lambda is not the same as the lexical position of its closure allocation.** Roslyn places the `newobj` based on capture scope.
2. **Any method parameter captured by a lambda forces the closure to be allocated at method entry**, regardless of how deeply nested the lambda is.
3. **Pattern variables in `if (x is T y && ...)` participate in method scope for closure purposes** - capturing them has the same hoisting effect as capturing a parameter.
4. **Hot paths must not capture method-wide variables in a lambda.** If you need a lambda on a slow path, move both the lambda and the slow-path captures into a separate method. The fast path stays allocation-free.
5. **Trust the IL, not the source.** When the allocation profiler and your mental model disagree, decompile the method. A two-minute look at the IL prolog will tell you exactly where the `newobj` is and why.


## How to spot this in your own code

A quick heuristic during code review:

> If a lambda inside a guard captures **a parameter of the enclosing method**
> (or a pattern variable from the guard itself), and the enclosing method is
> on a hot path, refactor it.

Or, even more mechanically: open the file in a decompiler, look at the method's `.locals init` and the first few IL instructions. If you see a `newobj ...<>c_DisplayClassN_M::.ctor()` at IL_0000, the closure is unconditional. The only way to make it conditional is to move the lambda into a helper.