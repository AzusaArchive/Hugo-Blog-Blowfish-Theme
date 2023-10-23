---
title: "C# Span&lt;T&gt;的使用"
date: 2023-06-15T00:29:38+08:00
tags: ["C#","C#基础"]
categories: [".NET"]
series: []
---

详见 [官方文档](https://learn.microsoft.com/zh-cn/dotnet/api/system.span-1?view=net-7.0)  

---

## Span\<T>
Span<T>是一种[引用结构体(ref struct)](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/ref-struct)，它提供任意内存的连续区域的类型安全和内存安全表示形式。Span<T>通常用于充当数组，但与数组不同的是，Span<T> 实例可以指向堆栈上托管的内存、本机内存或托管的内存。  
```cs
// 创建一个数组，并用Span指向这个数组。
// 注意这里的Span仅是指向了array的地址，并没有在堆上创建内存。
 var array = new int[]{1,2,3,4,5};
var arraySpan = new Span<int>(array);

for (int ctr = 0; ctr < arraySpan.Length; ctr++)//使用Length获取这串内存的长度，单位是int
    arraySpan[ctr] = 0; //使用[]运算符对特定位置的内存进行访问

foreach (var num in array)
    Console.Write(num);
// 输出：00000
```
上面的例子声明了一个内容为 {1,2,3,4,5} 的int数组，并定义了一个`Span<int>`指向该数组。此时`Span`起到了类似于指针的作用，对一段连续的内存进行直接访问。  

## Span\<T>和内存
`Span<T>`不仅可以指向某个数组（即.NET运行时托管的内存），还可以指向本地内存或是堆栈，如以下代码所示：

使用Marshal类分配一块本地内存（并非.NET运行时中的内存），并使用`Span<byte>`指向它：
```cs
// 创建一块本机内存，并用Span指向它，对非.NET运行时托管内存进行操作是不安全代码，使用unsafe块
var native = Marshal.AllocHGlobal(100);
Span<byte> nativeSpan;
unsafe
{
    nativeSpan = new Span<byte>(native.ToPointer(), 100); //将Span指向分配的本机内存
}
byte data = 0;
for (int ctr = 0; ctr < nativeSpan.Length; ctr++)
    nativeSpan[ctr] = data++; // 直接对内存进行操作

int nativeSum = 0;
foreach (var value in nativeSpan)
    nativeSum += value;

Console.WriteLine($"The sum is {nativeSum}");
Marshal.FreeHGlobal(native); //释放内存
// Output:  The sum is 4950
```

使用[`stackalloc`](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/operators/stackalloc)关键字在堆栈上分配一块内存，并用`Span`指向它：
```cs
Span<int> arraySpan = stackalloc int[5]; // 使用stackalloc关键字在堆栈上分配一个int数组

var n = 0;
for (int ctr = 0; ctr < arraySpan.Length; ctr++)
    arraySpan[ctr] = n++; // 访问数组内的元素

foreach (var num in arraySpan)
    Console.Write(num);
// 输出：01234
```

---

## Span\<T>和切片
使用`Span<T>`的构造函数，或是`Span<T>`对象的`Slice`方法，可以对连续内存的中的一部分区域进行操作：
```cs
var array = new int[]{1,2,3,4,5};
var arraySpan = new Span<int>(array, 1,3); //指向 array[1]~array[3]的内存

for (int ctr = 0; ctr < arraySpan.Length; ctr++)
    arraySpan[ctr] = 0;

foreach (var num in array)
    Console.Write(num);
// 输出：10005
```

`Span<T>`的切片操作常用于字符串，因为C#字符串的[不可变性](https://learn.microsoft.com/zh-cn/dotnet/api/system.string?view=net-7.0#immutability-and-the-stringbuilder-class)，对字符串的修改操作（如分割、拼接、修改内容、提取子串）并不是直接在原有字符串上进行操作，而是会返回新的字符串对象。因此频繁对字符串的修改操作会产生大量冗余内存给[垃圾回收期(GC)](https://learn.microsoft.com/zh-cn/dotnet/standard/garbage-collection/)增加压力，从而对性能产生影响。

```cs
public static (int year, int month, int day) StringToDate(string str)
{
    var year = str.Substring(0,4);
    var month = str.Substring(4,2);
    var day = str.Substring(6,2);
    
    return (int.Parse(year), int.Parse(month), int.Parse(day));
}
```
上述的代码对一段 `yyyyMMdd` 格式的日期字符串进行分割处理，并返回包含整型 (年，月，日) 的元组。  
通过`SubString`方法分割字符串，并用`int.Parse`方法转换为整形。  
这个方法多次对字符串进行提取子串的操作，由于字符串不可变的特性，每次提取子串时都会产生新的字符串实例，在内存分析器中可以看出产生了三个新的字符串：  

![SubString](./1.png "SubString产生的新字符串对象")  

所以，如果有大量与上述类似的字符串处理操作被执行，将会产生非常多的无用内存，而GC回收这些内存时会对性能产生大幅影响。所以对于大量的字符串处理操作，往往会选择使用[`StringBuilder`](https://learn.microsoft.com/zh-cn/dotnet/api/system.text.stringbuilder?view=net-7.0)类来避免产生过多垃圾。

但因为string对象也是一段连续的内存，此时也可以使用`Span<char>`来对直接对字符串进行操作：
```cs
public static (int year, int month, int day) StringToDate(ReadOnlySpan<char> str)
{
    var year = str.Slice(0,4);
    var month = str.Slice(4,2);
    var day = str.Slice(6,2);
    
    return (int.Parse(year), int.Parse(month), int.Parse(day));
}
```
方法将输入的字符串隐式转换为`ReadOnlySpan<char>`，并用`Slice`方法对这段内存进行切片，将切片的结果转换为`int`类型。  
`Span<T>`的是对该段内存的直接操作，所以不像`SubString`，`Slice`方法并不会在堆上分配任何的内存，仅仅是返回一个指向了该段内存的`Span<T>`对象而已。  
可以看到内存中没有产生多余的字符串对象：  

![Slice](./%E6%89%B9%E6%B3%A8%202023-06-15%20182459.png "Slice方法并不会产生新的字符串")  

直接对对象的内存进行操作避免产生垃圾，这对大量的字符串处理操作有非常可观的性能优化。  

## 缺陷
虽然使用`Span<T>`可以对性能有很大的提升，但因为它是[引用结构体](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/ref-struct)类型，所以使用起来有较多的限制，比如：

1. ref struct 不能是数组的元素类型。
2. ref struct 不能是类或非 ref struct 的字段的声明类型。
3. ref struct 不能实现接口。
4. ref struct 不能被装箱为 System.ValueType 或 System.Object。
5. ref struct 不能是类型参数。
6. ref struct 变量不能由 Lambda 表达式或本地函数捕获。
7. ref struct 变量不能在 async 方法中使用。 但是，可以在同步方法中使用 ref struct 变量，例如，在返回 Task 或 Task<TResult> 的方法中。
8. ref struct 变量不能在迭代器中使用。

这些限制决定了`Span<T>`不能在大多数场景中使用，但如果在编写某些方法时，对性能或是内存要求十分苛刻，那么使用`Span<T>`是个很不错的选择。