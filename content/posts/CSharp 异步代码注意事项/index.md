---
title: "C# 异步代码注意事项"
date: 2023-08-28T15:13:45+08:00
tags: ["C#", "C#基础", "ASP .NET Core"]
categories: [".NET"]
---

## 异步编程

异步编程已经在.NET平台存在了数年，但曾经异步程序的编写和维护却十分困难。而自从`async`/`await`关键字的问世(C# 5)，异步编程立刻变为了主流。现代化的框架(类似ASP .NET Core)的运行已经是完全异步的并且很难离开`async`关键字了。虽然`async`/`await`可以让你轻松写出高可用性的异步代码，但在使用这些关键字时仍有许多需要注意的细节。此文章将总结几点在ASP .NET Core中使用`async`/`await`关键字的注意事项，并探索编写C#异步程序的最佳实践。

---

## async的传染性以及同步化异步方法
一旦一个方法是异步的(使用了`async`关键字)，那么调用它的方法也**应当**是异步的（除非你有意让方法并行）。所以一旦使用了`async`关键字，整个程序直至Main方法都应当是异步的。

❌ 下面的例子使用了`Task.Result`使异步代码同步化并阻塞当前线程，这样“异步中的同步代码”将会消耗更多的资源。
```cs
public int DoSomethingAsync()
{
    var result = CallDependencyAsync().Result;
    return result + 1;
}
```

### 同步化异步代码
同步化异步方法（使用`Task.Result`或是`Task.Wait()`）将会执行以下操作：
1. 启动新线程执行异步方法
2. 调用线程被阻塞，等待异步方法完成
3. 当异步方法执行完后，取消调用线程的阻塞，这发生在另一个线程上。
所以为了完成异步代码的同步化，需要占用两个线程，如果大量使用可能会导致线程池不足。

### 死锁
在ASP .NET，WPF或是Winform中，对异步代码同步化可能会造成线程死锁，因为其`SynchronizationContext`的实现，异步方法在另一条线程上执行完**`await`关键字之前**的代码后，会去寻找调用方线程执行剩下的同步代码，而此时调用方线程正在阻塞并等待异步方法线程，无法响应，因此造成死锁。  

在ASP .NET Core中修复了此问题，所以不会出现死锁。但仍然不建议使用`Task.Result`或是`Task.Wait()`。

✅ 正确的调用方式是将调用方法也变为异步的并用`await`等待
```cs
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

---

## 永远不要使用async void
若使用异步方法，始终返回`Task`，**不要使用**`async void XXX()`。  
当在使用`async void`方法时，若方法中抛出了异常，可能导致整个应用程序崩溃且没有异常被抛出。  
async方法有着不同的异常处理方法，当`async Task`或`async Task<T>`方法中抛出异常时，异常将被自动捕获并记录在Task.Exceptions中，最终在`await Task`时被抛出，所以即使你对该方法使用`try-catch`也无法解决该问题。  
如果要在后台线程中执行方法而不想等待，可以使用Task.Run方法。

❌ 下面的代码使用`async void`方法并在其中抛出了异常，该异常不会被捕获并且可能会导致应用程序崩溃。
```cs
public class MyController : ControllerBase
{
    [HttpGet("/start")]
    public IActionResult Get()
    {
        BackGroundOperationAsync();
        return OK();
    }
    
    private async void BackGroundOperationAsync()
    {
        await Task.Delay(2000);
        throw new Exception();
    }
}
```

✅ 异步方法的返回值始终返回Task，确保异常被正确捕获
```cs
private async Task BackGroundOperationAsync()
{
    await Task.Delay(2000);
    throw new Exception();
}
```

---

## 简单的异步方法
在没有调用其他异步方法，并且执行耗时不多的简单异步方法中，可以不使用`async`/`await`关键字，而使用`Task.CompletedTask`或是`Task.FromResult()`来直接返回一个`Task`对象。这样就不会浪费额外的资源来启用线程去执行简单的异步方法。

```cs
public Task<int> AddAsync(int a, int b)
{
    return Task.FromResult(a + b);
}
```
> 因为Task是引用类型，`Task.FromResult`方法会实例化一个`Task`对象并带来微小的性能损耗。如果你在意这些性能消耗，可以使用[`ValueTask`](https://learn.microsoft.com/zh-cn/dotnet/api/system.threading.tasks.valuetask?view=net-7.0)来代替`Task`

```cs
public ValueTask<int> AddAsync(int a, int b)
{
    return new ValueTask<int>(a + b);
}
```

---

## async/await或直接返回Task
如果你的异步方法只是简单地对另一个异步方法进行转发，或是仅在函数的最后调用一个异步方法，则可以不使用`async`/`await`，而是在方法的最后直接返回另一个异步方法。
```cs
private Task NoAwait()
{
    /* 一些同步操作 */
    return DoSomeThingAsync();
}

private async Task DoSomeThingAsync()
{
    await Task.Delay(2000);
}
```
直接转发另一个异步方法会有微小的性能提升，并且使代码更简洁。但在安全性方面有一些弊端。  
- 当直接返回的异步方法中出现了异常时，异常的堆栈追踪将会缺少该方法中的部分。  
- 如果在转发另一个异步方法前使用了`using`关键字（无大括号），当异步方法出现了异常时，`IDisposable`对象将无法被正常关闭。

总的来说，如果只是简单的对另一个异步方法进行转发，则建议直接返回`Task`以提高性能，因为方法中没有其他操作且该方法的堆栈追踪也不太重要。而如果该方法中又其他操作并且可能引发异常，或是想要使用`using`关键字来关闭`IDisposable`对象，则建议使用`async`/`await`关键字。

---

## 始终传递CancellationToken
几乎所有.NET内部的异步方法都可以通过传递`CancellationToken`来取消异步方法的执行。所以在编写异步方法时，最好也提供`CancellationToken`参数，如果该方法执行时间较长，尝试在执行时检测`CancellationToken`来判断是否取消执行，如果该方法调用其他异步方法，传递`CancellationToken`参数。  

//TBD


## 参考文章
- Settling the Biggest Await Async Debate in .NET: <https://www.youtube.com/watch?v=aaC16Fv2zes>
- AsyncGuidance: <https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md>
- C# Task.ConfigureAwait方法能来做什么？: <https://blog.csdn.net/qq_38312617/article/details/104446307>
- ConfigureAwait FAQ: <https://devblogs.microsoft.com/dotnet/configureawait-faq/>