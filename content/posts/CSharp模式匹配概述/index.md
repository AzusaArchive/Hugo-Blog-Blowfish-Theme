---
title: "C#模式匹配概述"
date: 2023-06-13T00:55:21+08:00
tags: ["CSharp基础","CSharp"]
categories: [".NET"]
series: []
---

> 本文内容参考[微软官方文档](https://learn.microsoft.com/zh-cn/dotnet/csharp/fundamentals/functional/pattern-matching)

“模式匹配”是一种测试表达式是否具有特定特征的方法。在代码中使用模式匹配可以有效提高开发效率和代码可读性。C#从7.0开始支持模式匹配，并支持以下匹配方式：

>- 声明模式：用于检查表达式的运行时类型，如果匹配成功，则将表达式结果分配给声明的变量。
>- 类型模式：用于检查表达式的运行时类型。 在 C# 9.0 中引入。
>- 常量模式：用于测试表达式结果是否等于指定常量。
>- 关系模式：用于将表达式结果与指定常量进行比较。 在 C# 9.0 中引入。
>- 逻辑模式：用于测试表达式是否与模式的逻辑组合匹配。 在 C# 9.0 中引入。
>- 属性模式：用于测试表达式的属性或字段是否与嵌套模式匹配。
>- 位置模式：用于解构表达式结果并测试结果值是否与嵌套模式匹配。
>- var 模式：用于匹配任何表达式并将其结果分配给声明的变量。
>- 弃元模式：用于匹配任何表达式。
>- 列表模式：测试序列元素是否与相应的嵌套模式匹配。 在 C# 11 中引入。  

在C#代码中，可以使用`is`,`or`,`not`,`and`和`switch`表达式进行模板匹配，详见：  
[模式匹配 - 模式中的 is 和 switch 表达式，以及 and、or 和 not 运算符](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/patterns#list-patterns)


## 声明和类型模式

### 条件语句
检查对象的类型是否与给定的一致（C# 9.0以上）：
```cs
object greeting = "Hello, World!";
if (greeting is string) // 如果greeting是string类型
{
    Console.WriteLine((string)greeting);
}
```

声明模式，匹配时进行声明，省去了一步类型转换：
```cs
if (greeting is string message) // 如果greeting类型为string，则将其以string类型赋值到message上
{
    Console.WriteLine(message);
}
```
(其实是做了这一步转换
```cs
var message = greeting as string;
if (message is not null)
...
```
</br>

可以对其进行null检查，在进行上述的类型匹配也会自动进行null检查
```cs
string? str = "hello world!";
if (str is not null)
{
    Console.WriteLine(str);
}
```

加上与或表达式：
```cs
object a = 1;
object b = 1.22f;
if (a is int and b is float)
...//true
if (a is float or b is float)
...//true
```

注意，如果对象派生，实现，或是存在任何到匹配类型的[隐式引用转换](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/language-specification/conversions#implicit-reference-conversions)，表达式都会为`true`
```cs
var list = new List<int>{1,2,3};
if (list is IEnumerable) // list实现IEnumerable接口
{
    Console.WriteLine("list is Enumerable!");
}
```

### Switch表达式
使用[switch 关键字的模式匹配表达式](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/switch-expression)  

C#将Switch语句融合进了模板匹配，上述的类型匹配可以使用Switch表达式来实现：
```cs
object obj = DateTime.Now;
string type = obj switch
{
    int => "int",
    string => "string",
    float => "float",
    DateTime => "DateTime",
    null => "null",
    _ => "Unknown Type"
};
Console.WriteLine("object type is " + type);
```
并非switch表达式，正常的switch语句也是可行的
```cs
string type;
switch (obj)
{
    case int: type = "int";
        break;
    case string: type = "string";
        break;
    case float: type = "float";
        break;
    case DateTime: type = "DateTime";
        break;
    case null: type = "null";
        break;
    default: type = "Unknown type";
        break;
}
```
但很显然switch表达式更加精练

---

## 常量模式
模板匹配可以对常量值进行匹配
```cs
string str = "hello";

if (str is "hello")
{
    Console.WriteLine("str is \"hello\"");
}
```

---

## 关系模式
从 C# 9.0 开始，可使用关系模式将表达式结果与常量进行比较
```cs
int i = Random.Shared.Next(10);

if (i is > 5)
{
    Console.WriteLine("i > 5");
}
```
感觉有点蠢？用上switch就不一样了
```cs
Console.WriteLine(Classify(13));  // output: Too high
Console.WriteLine(Classify(double.NaN));  // output: Unknown
Console.WriteLine(Classify(2.4));  // output: Acceptable

static string Classify(double measurement) => measurement switch
{
    < -4.0 => "Too low",
    > 10.0 => "Too high",
    double.NaN => "Unknown",
    _ => "Acceptable",
};
```
> *"在关系模式中，可使用关系运算符<、>、<= 或 >= 中的任何一个。 关系模式的右侧部分必须是常数表达式。 常数表达式可以是 integer、floating-point、char 或 enum 类型。"*

---

## 逻辑模式
C# 9.0 以上，可以使用`and`,`or`,`not`关键字来创造'与'，'或'，'非'的逻辑
```cs
if (input is not null)
{
    // ...
}
```
运行的优先级也是 `not` > `and` > `or`，并且从右开始扫描

### 括号模式
使用括号强调或改变模式匹配优先级：
```cs
if (input is not (float or double))
{
    return;
}
```


---

## 属性模式
将对象的属性或是字段进行匹配：
```cs
static bool IsConferenceDay(DateTime date) => date is { Year: 2020, Month: 5, Day: 19 or 20 or 21 };
```
属性模式可以用一串优雅的模式匹配代替复杂的条件语句，还能结合声明模式和类型模式：
```cs
static string TakeFive(object input) => input switch
{
    string { Length: >= 5 } s => s.Substring(0, 5), //结合了声明模式，类型模式，逻辑模式，常量模式和属性模式
    string s => s, 

    ICollection<char> { Count: >= 5 } symbols => new string(symbols.Take(5).ToArray()),
    ICollection<char> symbols => new string(symbols.ToArray()),

    null => throw new ArgumentNullException(nameof(input)),
    _ => throw new ArgumentException("Not supported input type."),
};
```

属性模式是可以嵌套的，对于匹配的属性类型，还可以继续进行模式匹配：
```cs
public record Point(int X, int Y);
public record Segment(Point Start, Point End);

static bool IsAnyEndOnXAxis(Segment segment) =>
    segment is { Start: { Y: 0 } } or { End: { Y: 0 } };
```
对属性嵌套进行模式匹配有些麻烦，C# 10.0 以上可以使用"扩展属性模式"，即成员运算符来对属性进行匹配，可将上述示例中的方法重构为以下等效代码：
```cs
static bool IsAnyEndOnXAxis(Segment segment) =>
    segment is { Start.Y: 0 } or { End.Y: 0 };
```
> *“优雅，实在太优雅了"（指属性模式*

---

## 位置模式
位置模式可以匹配[元组](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/value-tuples)中的元素：
```cs
var location = (5, 10);
if (location is (5, 10))
    ...
```

位置模式也可以和其他各种模式结合：
```cs
if (location is (>5, <10))
    ...
```

这使得switch表达式可以结合元组一次匹配任意个对象：
```cs
static int[] Encode(int x, int y) => (x, y) switch
{
    (< left, > top) => new[] { 1, 0, 0, 1 },
    (< left, < bottom) => new[] { 0, 1, 0, 1 },
    (> right, > top) => new[] { 1, 0, 1, 0 },
    (> right, < bottom) => new[] { 0, 1, 1, 0 },
    (_, > top) => new[] { 1, 0, 0, 0 },
    (_, < bottom) => new[] { 0, 1, 0, 0 },
    (< left, _) => new[] { 0, 0, 0, 1 },
    (> right, _) => new[] { 0, 0, 1, 0 },
    _ => new[] { 0, 0, 0, 0 }
};
```


如果对象声明了[解构函数](https://learn.microsoft.com/zh-cn/dotnet/csharp/fundamentals/functional/deconstruct#user-defined-types)，可以直接对其使用位置匹配：
```cs
public readonly struct Point
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) => (X, Y) = (x, y);
    
    public void Deconstruct(out int x, out int y) => (x, y) = (X, Y); // 声明解构函数

    static string Classify(Point point) => point switch //对对象解构出的元组进行匹配
    {
        (0, 0) => "Origin", //使用位置模式匹配point对象解构出的元组
        (1, 0) => "positive X basis end",
        (0, 1) => "positive Y basis end",
        _ => "Just a point",
    };
}
```

结合关系模式，使用元组内元素的名称：
```cs
var numbers = new List<int> { 1, 2, 3 };
if (SumAndCount(numbers) is (Sum: var sum, Count: > 0)) //返回元组可以使用关系模式访问其元素
{
    Console.WriteLine($"Sum of [{string.Join(" ", numbers)}] is {sum}");  // output: Sum of [1 2 3] is 6
}

static (double Sum, int Count) SumAndCount(IEnumerable<int> numbers)
{
    int sum = 0;
    int count = 0;
    foreach (int number in numbers)
    {
        sum += number;
        count++;
    }
    return (sum, count);
}
```

---

## var模式
可使用 var 模式来匹配任何表达式（包括 null），并将其结果分配给新的局部变量
```cs
static bool IsAcceptable(int id, int absLimit) =>
    SimulateDataFetch(id) is var results 
    && results.Min() >= -absLimit 
    && results.Max() <= absLimit;

static int[] SimulateDataFetch(int id)
{
    var rand = new Random();
    return Enumerable
               .Range(start: 0, count: 5)
               .Select(s => rand.Next(minValue: -10, maxValue: 11))
               .ToArray();
}
```

需要布尔表达式中的临时变量来保存中间计算的结果时，var 模式很有用。 当需要在 switch 表达式或语句的 when 大小写临界子句中执行更多检查时，也可使用 var 模式，如以下示例所示：
```cs
public record Point(int X, int Y);

static Point Transform(Point point) => point switch
{
    var (x, y) when x < y => new Point(-x, y),
    var (x, y) when x > y => new Point(x, -y),
    var (x, y) => new Point(x, y),
};

static void TestTransform()
{
    Console.WriteLine(Transform(new Point(1, 2)));  // output: Point { X = -1, Y = 2 }
    Console.WriteLine(Transform(new Point(5, 2)));  // output: Point { X = 5, Y = -2 }
}
```

---

## 弃元模式
用`_`符号来在[switch表达式](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/switch-expression)中充当`default`开关


---

## 列表模式
从C# 11开始，可以使用列表模式匹配列表中的内容：
```cs
int[] numbers = { 1, 2, 3 };

Console.WriteLine(numbers is [1, 2, 3]);  // True
Console.WriteLine(numbers is [1, 2, 4]);  // False
Console.WriteLine(numbers is [1, 2, 3, 4]);  // False
Console.WriteLine(numbers is [0 or 1, <= 2, >= 3]);  // True
```

如前面的示例所示，当每个嵌套模式与输入序列的相应元素匹配时，列表模式就会匹配。 可使用列表模式中的任何模式。 若要匹配任何元素，请使用弃元模式，或者，如果还想捕获元素，请使用 var 模式，如以下示例所示：
```cs
List<int> numbers = new() { 1, 2, 3 };

if (numbers is [var first, _, _]) //捕获第一个元素，丢弃后两个元素
{
    Console.WriteLine($"The first element of a three-item list is {first}.");
}
```

如果无需匹配整个序列，而是只需要匹配列表的头尾，可以使用切片模式`..`：
```cs
Console.WriteLine(new[] { 1, 2, 3, 4, 5 } is [> 0, > 0, ..]);  // True 匹配前两个
Console.WriteLine(new[] { 1, 1 } is [_, _, ..]);  // True 匹配前两个并弃元
Console.WriteLine(new[] { 1 } is [1, 2, ..]);  // False 数量不一致，匹配错误

Console.WriteLine(new[] { 1, 2, 3, 4 } is [.., > 0, > 0]);  // True 匹配最后两个
Console.WriteLine(new[] { 2, 4 } is [.., > 0, 2, 4]);  // False 匹配最后三个
Console.WriteLine(new[] { 2, 4 } is [.., 2, 4]);  // True 

Console.WriteLine(new[] { 1, 2, 3, 4 } is [>= 0, .., 2 or 4]);  // True 匹配第一个和最后一个
Console.WriteLine(new[] { 1, 0, 0, 1 } is [1, 0, .., 0, 1]);  // True 匹配前两个和最后两个
Console.WriteLine(new[] { 1, 0, 1 } is [1, 0, .., 0, 1]);  // False 
```

嵌套其他模式：
```cs
void Validate(int[] numbers)
{
    var result = numbers is [< 0, .. { Length: 2 or 4 }, > 0] ? "valid" : "not valid";
    Console.WriteLine(result);
}

Validate(new[] { -1, 0, 1 });  // output: not valid
Validate(new[] { -1, 0, 0, 1 });  // output: valid
```