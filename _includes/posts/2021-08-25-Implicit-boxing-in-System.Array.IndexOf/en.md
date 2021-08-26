## 1. The Problem

I was fixing some performance issues for our project last week, and interestingly the profiler shows boxing operation (allocation of `char`) happened inside a method that calls `System.Array.IndexOf<T>()`. That method is fairly simple and no boxing at all, so the `.net framework` source code becomes the top suspect.

## 2. Analysis

Although `System.Array.IndexOf<T>()` seems to be a generic method that shouldn't cause any boxing, but its [source code](https://referencesource.microsoft.com/#mscorlib/system/array.cs,35ebfdcba4f0e942) shows the real job is actually done by `EqualityComparer<T>.IndexOf()`.

```c#
public static int IndexOf<T>(T[] array, T value, int startIndex, int count) 
{
  ......
  return EqualityComparer<T>.Default.IndexOf(array, value, startIndex, count);
}
```

In our case the data type is `char`, so`EqualityComparer<char>.Default` would return an instance of `GenericEqualityComparer<char>`.


```c#
public abstract class EqualityComparer<T> : IEqualityComparer, IEqualityComparer<T>
{
  private static EqualityComparer<T> CreateComparer() 
  {
    ......
    // If T implements IEquatable<T> return a GenericEqualityComparer<T>
    if (typeof(IEquatable<T>).IsAssignableFrom(t)) {
        return an instance of GenericEqualityComparer<T>;
    }

    ......
    // Otherwise return an ObjectEqualityComparer<T>
    return new ObjectEqualityComparer<T>();
}
```

The problem is that `GenericEqualityComparer<T>` would check if the data is null before calling `Equals`, thus boxing would happen when the data is value type:

```c#
internal class GenericEqualityComparer<T>: EqualityComparer<T> where T: IEquatable<T>
{
  internal override int IndexOf(T[] array, T value, int startIndex, int count) 
  {
    int endIndex = startIndex + count;
    if (value == null)  ////Boxing
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] == null) return i;
      }
    }
    else 
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] != null && array[i].Equals(value)) return i;
      }
    }
    return -1;
  }
```

The IL code of `GenericEqualityComparer<T>` also prove the existence of the `box` instruction:

![](https://filedn.com/lCdMuPWubK2H86dRAWfspRh/BlogImages/System.Array.IndexOf.png)

I thought someone else might have encountered the same problem already so I searched around and did find one [Collections Without the Boxing](https://www.jacksondunstan.com/articles/5148). Its problem was that `ObjectEqualityComparer<T>` is created because of a `struct` does not implement the `IEquatable<T>` interface, after reading the source code the author of the article had his `struct` implemented `IEquatable<T>` so that `GenericEqualityComparer<T>` would be created, and he thought problem was solved.

Obviously the author of that article did not realize the `boxing` problem of the `GenericEqualityComparer<T>.IndexOf`, and if we look at the source code of [`ObjectEqualityComparer<T>`](
https://referencesource.microsoft.com/#mscorlib/system/collections/generic/equalitycomparer.cs,ac282b3e1817bb9b), we can see its `IndexOf` implementation is exactly same as `GenericEqualityComparer<T>`!

```c#
internal class ObjectEqualityComparer<T>: EqualityComparer<T>
{
  internal override int IndexOf(T[] array, T value, int startIndex, int count) 
  {
    int endIndex = startIndex + count;
    if (value == null) ////Boxing
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] == null) return i;
      }
    }
    else 
    {
      for (int i = startIndex; i < endIndex; i++) 
      {
        ////Boxing
        if (array[i] != null && array[i].Equals(value)) return i;
      }
    }
    return -1;
  }
```

## 3. Solution

Actually the default implementation of `EqualityComparer<T>.IndexOf()` is already what we need. But `EqualityComparer<T>` is an abstract base class that cannot used directly, so we either subclass it (but also need to implement some methods that are not needed when calling `IndexOf`）, or just copy and paste this piece of codes.

```c#
public abstract class EqualityComparer<T> : IEqualityComparer, IEqualityComparer<T>
{
  internal virtual int IndexOf(T[] array, T value, int startIndex, int count) 
  {
    int endIndex = startIndex + count;
    for (int i = startIndex; i < endIndex; i++) 
    {
      if (Equals(array[i], value)) return i;
    }
    return -1;
  }
```

The good news is that `.net Core` does not have such problem anymore, after all a lot of codes are re-written for the sake of performance improvements.