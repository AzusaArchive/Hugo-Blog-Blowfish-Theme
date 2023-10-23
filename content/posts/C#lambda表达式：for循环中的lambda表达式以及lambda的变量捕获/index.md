---
title: "C#lambda表达式：for循环中的lambda表达式以及lambda的变量捕获"
date: 2023-10-21T15:26:23+08:00
tags: ["C#", "Bug修复", "C#基础"]
categories: [".NET"]
series: []
---

## 问题来源
在Unity脚本中为组件绑定事件时，使用了for循环以及lambda表达式批量地对一个数组的组件进行事件绑定，代码如下：  

```cs
private void Awake()
{
    _weatherViews = new WeatherView[7];
    for (var i = 0; i < 7; i++)
    {
        _weatherViews[i] = Instantiate(Resources.Load<WeatherView>("WeatherCard"), weatherCards);
        _weatherViews[i].OnSelected += () => _selectedIndex = i;
    }
}
```

这段代码期望的结果是：  
当数组中某一`WeatherView`触发`OnSeleted`事件后，`_selectedIndex`的值变为对应`WeatherView`在`_weatherViews`数组中的索引。  

而实际运行之后，发现不论是哪个`WeatherView`触发了`OnSeleted`，`_selectedIndex`都会变成 ***7***。 

初步判断是lambda表达式在for循环中进行定义引起的，因为C#的lambda表达式产生的匿名函数是预先编译而不是运行时解析的，所以就展开了测试。

---

## 测试
创建了一个简单的控制台应用程序，使用以下代码进行测试：
```cs
Console.WriteLine("for lambda test");
var array1 = new Action[10];
int i = 0;
for (; i < array1.Length; i++)
    array1[i] = () => Console.WriteLine(i);

foreach (var action in array1)
    action();
```

运行程序，查看控制台的输出。
```
for lambda test
10          
10          
10          
10          
10          
10          
10          
10          
10          
10        
```
发现函数中使用lambda表达式`() => Console.WriteLine(i)`产生的委托中，所有的`i`都变成了 ***10*** ，而按照正常思维，期待的输出结果应当是 **0~9** 。

---

## 原因和解决方法
如果学习过C++，会知道C++的lambda表达式中需要手动进行变量捕获 `[&i](){ std::cout << i << '\n'; }`。  

而C#里lambda表达式的变量捕获是自动的，如果lambda中引用了外部变量，C#编译器会自动进行捕获。但是否捕获外部变量，以及外部变量定义的位置都会影响lambda编译后的结果。

通过查看项目编译出的**低级别C#代码**，得出了一些结论：

- 如果lambda没有捕获变量，则会单独编译成一个匿名函数，编译后类中所有的匿名函数都会一起存放在一个类中。这个类在程序启动时实例化并作为静态字段。
- 如果lambda捕获了局部变量，则会编译成一个函数闭包，每个函数闭包都会编译出一个类存放函数以及函数捕获的变量，捕获的变量将会作为这个类的字段，所有引用到该局部变量的代码（包括lambda以外的代码）都将使用该类中对应的字段来代替，闭包类将在距离lambda最近的一个局部变量所在的代码块的起始位置被实例化，如果多个lambda引用了完全相同的局部变量，那么该闭包类的实例将被复用。
- 如果lambda捕获的是一个对象，则该对象会被引用为闭包类的字段，如果捕获的是当前类中的字段，则当前类的实例会被引用。
- 当lambda被执行时，如果lambda不为函数闭包，则直接通过公共的匿名函数类调用对应的匿名函数  
    如果lambda为函数闭包，则通过闭包类调用对应函数

> 例1：  
> C#代码：
> ```cs
> var a = delegate()
> {
>     Console.WriteLine(10);
> };
> ```
> 低级别C#代码：
> ```cs
> Action a = Program.<>c.<>9__0_0 ?? (Program.<>c.<>9__0_0 = new Action((object) Program.<>c.<>9, > __methodptr(<Main>b__0_0)));
> 
> //匿名函数编译后的类
> [CompilerGenerated]
> [Serializable]
> private sealed class <>c
> {
>     public static readonly Program.<>c <>9;
>     public static Action <>9__0_0;
>     public static Func<int, Program.DelegateTest> <>9__0_2;
> 
>     static <>c()
>     {
>         Program.<>c.<>9 = new Program.<>c();
>     }
> 
>     public <>c()
>     {
>         base..ctor();
>     }
> 
>     internal void <Main>b__0_0() // 委托对应的匿名函数
>     {
>         Console.WriteLine(10);
>     }
> 
>     internal Program.DelegateTest <Main>b__0_2(int _)
>     {
>         return new Program.DelegateTest();
>     }
> }
> ```
>
> 例2：
> C#代码：
> ```cs
> for (var i = 0; i < array1.Length; i++)
>     array1[i] = () => Console.WriteLine(i);
> ```
> 低级别C#代码：
> ```cs
> //lambda编译成的闭包类，注意捕获的局部变量 i 被作为了字段
> [CompilerGenerated]
> private sealed class <>c__DisplayClass0_0
> {
>     public int i;
>
>     public <>c__DisplayClass0_0()
>     {
>         base..ctor();
>     }
>
>     internal void <Main>b__1()
>     {
>         Console.WriteLine(this.i);
>     }
> }
> 
> // 闭包类在捕获变量所处代码块的起始处被实例化
> Program.<>c__DisplayClass0_0 cDisplayClass00 = new Program.<>c__DisplayClass0_0();
> // 原本的局部变量i被闭包类中的字段所代替
> for (cDisplayClass00.i = 0; cDisplayClass00.i < array1.Length; cDisplayClass00.i++)
>     array1[cDisplayClass00.i] = new Action((object) cDisplayClass00, __methodptr>(<Main>b__1));
> ```

> - 如果使用的IDE是 JetBrains Rider，可以使用内置的*IL Viewer*查看该C#文件编译后的结果。构建当前项目，并使用*IL Viewer*查看**低级别C#代码**。
> - <https://sharplab.io>也可以查看C#代码的编译结果。

回到最初的问题，在for循环中，lambda引用变量i后，i将作为字段存储在闭包类中，此时闭包类将在for循环所在代码块被实例化，循环中创建的所有lambda共用一个闭包类，在循环执行过程中字段的值将一直被改变，所以当循环结束，lambda被调用时，i的值将会是循环退出时的状态。  

解决方案是在for循环的过程中新建一个局部变量i1，用i来赋值，然后lambda引用该局部变量i1而不是i。
```cs
var array1 = new Action[10];
int i = 0;
for (; i < array1.Length; i++)
{
    var i1 = i; //在for循环内部添加一个局部变量
    array1[i] = () => Console.WriteLine(i1);
}

foreach (var action in array1)
    action();
```
这样做会改变编译后的代码，lambda捕获了for循环内的局部变量i1，所以其闭包类将会在for循环内进行实例化，也就是每次循环实例化一次闭包类，循环中创建的每个lambda的闭包类实例都不相同。这样就避免了闭包类实例的复用，每个lambda都是独立的函数闭包。

编译后的低等级C#代码：
```cs
for (int i = 0; i < array1.Length; ++i)
{
    // 可以看到闭包类在for循环内实例化，每次循环创建的lambda表达式都是独立的函数闭包。
    Program.<>c__DisplayClass0_0 cDisplayClass00 = new Program.<>c__DisplayClass0_0();
    cDisplayClass00.i1 = i;
    array1[i] = new Action((object) cDisplayClass00, __methodptr(<Main>b__2));
}

// lambda编译的函数闭包类
[CompilerGenerated]
private sealed class <>c__DisplayClass0_0
{
public int i1;

public <>c__DisplayClass0_0()
{
    base..ctor();
}

internal void <Main>b__2()
{
    Console.WriteLine(this.i1);
}
}
```

参考文章： 
> <https://www.jetbrains.com/help/rider/AccessToModifiedClosure.html>