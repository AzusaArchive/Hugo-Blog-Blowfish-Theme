---
title: "C# 正确的null检查"
date: 2023-06-14T03:17:52+08:00
tags: ["C#","C#基础"]
categories: [".NET"]
series: []
---

## null检查
```cs
object? obj = null;
if (obj == null)
{
    Console.WriteLine("object is null");
}
```

上述的代码是C#最基本，也是最常用的null检查，但在现代C#语言中，这种做法并不是最好的。


## 被重载的`==`运算符
使用`==`或是`!=`运算符进行null检查时，如果遇到对象类型的`==`和`!=`运算符被重载的情况，可能得到意料之外的结果：
```cs
class User
{
    public string Name { get; set; }
    public static bool operator ==(User left, User right) => left.Name == right.Name;
    public static bool operator !=(User left, User right) => !(left == right);
}
static void Main(string[] args)
{
    var user1 = new User { Name = "Azusa" };
    var user2 = new User { Name = "Hina" };
    Console.WriteLine(user1 != null); //此时会抛出空引用异常，因为在 != 运算符中引用了 null 的 Name 属性
}
```

这样的行为可能会导致程序中隐藏的漏洞，比如上述的情况只有在运行时抛出异常才能察觉。（如果使用了C# 8.0的空引用类型，则会有编译器警告。实际上在主流的IDE中，重载前后的==运算符显示在编辑器中的颜色是不一致的，不过这不是很容易察觉到）

## ReferenceEquals
如果要确保我们的意图是“检查对象的引用是否为空”，而不是通过`==`或`!=`运算符来检查，则需要使用`Object.ReferenceEquals()`函数：
```cs
if (ReferenceEquals(user1,null))
    ...
```
使用`ReferenceEquals`函数保证了null检查不会被影响，因为这个函数真的就是检查两参数的引用是否相等，且不会被重载。  

有趣的是，当转到`Object.ReferenceEquals()`函数的实现时，发现内部还是调用了`==`运算符：
```cs
[NonVersionable]
public static bool ReferenceEquals(object? objA, object? objB)
{
    return objA == objB;
}
```
但因为接受的两个参数都被转换为了可空的`object`类型，所以调用的`==`运算符依然是没被重载的，`object`类型中定义的`==`运算符。  
既然如此，在使用`==`进行null检查时，可以加一步转换，来防止运算符被重载的情况。
```cs
if ((object?)user1 == null)
{
    Console.WriteLine("object is null");
}
```
但这样的代码不够美观。

## 模式匹配
C# 7.0出现了模式匹配后，null检查出现了新的方法。使用`is`运算符进行[常量模式](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/patterns#constant-pattern)匹配时可以接收null值，所以null检查变成了这样：
```cs
if (user1 is null)
{
    Console.WriteLine("object is null");
}
```
如果使用的C#是9.0版本之后，则可以加上`not`运算符来实现非null检查：
```cs
if (user1 is not null)
{
    Console.WriteLine("object is not null");
}
```

这样的代码看起来更直观了。实际上，这也是微软官方推荐的做法：[null检查](https://learn.microsoft.com/zh-cn/dotnet/csharp/fundamentals/functional/pattern-matching#null-checks)

在C#内部，实际上使用`is`进行null检查时，也是先把对象转换为了`object`类型。


## 总结
所以，在进行null检查时，模式匹配`obj is null`是最推荐的做法，但如果语言版本不满足(<7.0)，进行null检查则要更加谨慎，`object.ReferenceEquals()`函数是最安全的做法，而使用`==`运算符进行检查是比较危险的，因为运算符可能被重载，得到意料之外的结果。