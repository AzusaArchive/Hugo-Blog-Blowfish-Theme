---
title: "C#10新增功能"
date: 2023-04-12T16:55:10+08:00
tags: ["C#基础"]
categories: ["C#"]
series: []
---

> 本文章将举例介绍绝大多数C# 10的新特性，在.NET 6以上的项目支持这些特性。  
有些预览特性需要在项目配置文件(.csproj)中配置`<LangVersion>preview<LangVersion/>`才可生效  
[C#10新增功能](https://learn.microsoft.com/zh-cn/dotnet/csharp/whats-new/csharp-10)  

- ## 全局Using
在任何一个C#类中使用global关键字修饰using指令，该using指令将会应用于整个项目

globalUsing.cs
```
//这些using将会应用到整个项目
global using System.Collection.Generic;
global using Newtonsoft.Json;
...
```

- ## 文件范围的命名空间
在C#文件中指定命名空间时可以不用花括号{}了，该命名空间将会应用于整个文件范围。但相反的，你无法在一个cs中定义多个命名空间中的内容（一般来说也不会这样做）。

```
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.IdentityModel.Tokens;

namespace Azusa.Shared.Security;//不使用{}指定范围，整个文件都将被定义到该命名空间中。

public static class JwtTokenGenerator
{
    ...
}
```

- ## 常量内插字符串
你可以在const string中使用内插字符串了，你可以将用在某些只接受常量字符串的地方。
```
...
private const string BaseUrl = "localhost:7890";
public const string SignInUrl = $"{BaseUrl}/SignIn";//你可以使用$内插常量字符串了
...

[HttpGet(SignInUrl)]//在这种地方适合使用常量字符串！

```

- ## 泛型特性（注解）
C#特性也支持泛型了，传递类型参数更加方便，不需要typeof了。
```
public class UnitOfWorkAttribute<TDbContext> : Attribute where TDbContext : DbContext
{
    ...
}
```

- ## Lambda表达式改进
`var`关键字可以推断lambda表达式的返回值了
```
var func = () => "Nice";//现在编译器可以推断出这个lambda是Func<string>了

var func1 = () => null;//不过这样是不行的，因为null没有类型
var func2 = string? () => null;//但是可以显式指定一下
```

**特性**也可以支持在lambda上标注了
```
var func3 = [UnitOfWorkAttribute<MyDbContext>()](long id) => myDbCtx.Books.Find(id);
```

- ## 模式匹配属性扩展
在模式匹配的时候可以匹配对象的属性，并且用成员运算符`.`来匹配属性对象的属性
```
if (book is Book{PubTime.Month:12})//用成员运算符来读取对象属性的属性
{
    ...
}

//以前得这样
if (book is Book { PubTime: { Month: 12 } })
{
    
}
```

- ## 记录(record)结构体
`record`类型也可以声明为结构体了，也是值类型。
```
public record struct Color(short Red, short Green, short Blue);
```

- ## 结构体的优化
在C#10以前，结构体的显式构造函数必须得给所有字段和自动属性赋值
比如这样一个结构体
```
public struct Color
{
    public short Red { get; set; }
    public short Green { get; set; }
    public short Blue { get; set; }
    ...
}
```
构造函数要么不写（隐式无参），要么写全（显式，并且所有自动属性和字段都得赋值）  
（很迷惑的操作，不知为何要这样设定）
```
...
public Color(short red, short green, short blue)//字段和自动属性都得赋值
{
    Red = red;
    Green = green;
    Blue = blue;
}
...
```
C#10以后，将接受赋值不完全的显式构造函数
```
public Color(short red, short green)
{
    Red = red;
    Green = green;
}
```
并且，C#10的结构体也支持with关键字
```
var color = new Color(122, 255, 255);
var color1 = color with { Green = 12 };
```

- ## 在同一析构中进行赋值和声明
可以通过析构对象来给变量赋值，或是
```
(int x, int y) = point;//析构对象来初始化两个变量，注意这不是元组

int x1 = 0;
int y1 = 0;
(x1, y1) = point;//析构对象给现有变量赋值
```

在C#10以前初始化和赋值不能同时进行
```
int x = 0;
(x, int y) = point;//C#10以前会错误
```
但现在不会
