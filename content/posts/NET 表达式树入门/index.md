---
title: ".NET 表达式树入门"
date: 2023-03-15T15:44:03+08:00
tags: ["CSharp","Linq"]
categories: [".NET"]
series: []
---

## .NET 表达式树
https://zhuanlan.zhihu.com/p/247937380
https://cloud.tencent.com/developer/article/1817790  

.NET Expression表达式树将代码抽象成了一颗对象树，树上的每一个节点都是一个代码表达式，它的构造类似编译原理中的抽象语法树（AST）。

可以使用lambda表达式进行声明表达式树，编译器将会自动转换，但是该lambda表达式不能是语句（即表达式只能有一句，不能用{}组成多句）。  

表达式树可以转换为委托，在一些linq场景可以通过构建表达式树来动态拼接一些linq查询条件。

## 表达式树入门
一个表达式树是类似以下的结构
```
       表达式C
          |  
    +-----+-----+
    |           |
 表达式A       表达式B

```
比如简单的lambda表达式x => x + 1可以用转化为以下表达式
```
    x + 1
      |
  +---+---+
  |       |
  x       1
```
其中叶子节点`x`为变量表达式，叶子节点`1`为常量表达式，`x+1`节点为二叉表达式，使用了`+`运算符

在C#中，表达式树可以通过`Expression`及其派生类的各种方法手动构建，`Exression`类常用的静态方法如下：  
- `Expression.Constant()` 创建一个常量表达式。  
- `Expression.Variable()` 创建一个变量表达式。  
- `Expression.New()` 创建一个实例化表达式。  
- `Expression.Assaign()` 创建一个赋值表达式。  
- `Expression.Equal()` 创建一个相等比较表达式。  
- `Expression.Call()` 创建一个方法调用表达式。  
- `Expression.Condition()` 创建一个分支逻辑表达式。  
- `Expression.Loop()` 创建一个循环逻辑表达式。  
- `Expression.Add()/Subtract()/Multiply()/Divide()` 加/减/乘/除表达式  
    ...

### 使用C#代码构造简单的表达式树
`x => x + 1`
```cs
public static void Example1()
{
    var param1 = Expression.Variable(typeof(int), "x");//声明变量表达式，类型为int，参数名为"x"
    var const1 = Expression.Constant(1);//常量表达式
    var biExpr = Expression.Add(param1, const1);//声明进行加法运算的二叉表达式，左节点为x,右节点为1
    //x => x + 1表达式树就构成了
    //使用Expression.Lambda方法可以将表达式树转换为lambda表达式树，在类型参数中可以指定转换的委托类型
    var lambdaExpr = Expression.Lambda<Func<int, int>>(biExpr, param1);//lambda输入参数为x变量
    //使用LambdaExpression.Compile方法将转换好的lambda表达式树编译成委托
    Func<int,int> func = lambdaExpr.Compile();
    //调用编译出的委托
    Console.WriteLine(func(5));
}
```

### 更复杂一些的表达式树
`(a,b,c) => (a + b + c) / 3`
```cs
public static void Example2()
{
    var param1 = Expression.Variable(typeof(int), "a");
    var param2 = Expression.Variable(typeof(int), "b");
    var param3 = Expression.Variable(typeof(int), "c");
    var const1 = Expression.Constant(3.0f,typeof(float));
    var biExpr1 = Expression.Add(param1, param2);//构造参数1，2，生成加法表达式树1
    var biExpr2 = Expression.Add(biExpr1, param3);//构造加法表达式树1和参数3，生成加法表达式树2
    var biExpr3 = Expression.Convert(biExpr2, typeof(float));//两整形相除不会保留小数，转化为浮点型
    var biExpr4 = Expression.Divide(biExpr3, const1);//构造加法表达式树2和常量表达式1，生成除法表达式树3
    var lambdaExpr = Expression.Lambda<Func<int,int,int,float>>(biExpr4, param1,param2,param3);//根据表达式树3和三个参数，将表达式树转换为lambda表达式树
    var func = lambdaExpr.Compile();
    Console.WriteLine(func(2, 3, 3));
}
```

### 方法调用表达式
`(str,value) => !str.IsNullOrWhiteSpace(str) && str.Contains(value)`
```cs
public static void Example3()
{
    var param1 = Expression.Variable(typeof(string), "str");
    var param2 = Expression.Variable(typeof(string), "value");
    //使用反射获取静态方法，并与参数表达式构造成方法调用表达式
    var callExpr1 = Expression.Not(Expression.Call(typeof(string).GetMethod(nameof(string.IsNullOrWhiteSpace), new Type[]{typeof(string)})!, param2));
    //使用反射获取实例方法，实例方法需要指定调用实例的表达式
    var callExpr2 = Expression.Call(param1, typeof(string).GetMethod(nameof(string.Contains), new[] {typeof(string)} )!, param2);
    var biExpr1 = Expression.AndAlso(callExpr1, callExpr2);//与运算
    var lambdaExpr = Expression.Lambda<Func<string,string,bool>>(biExpr1, param1,param2);
    var func = lambdaExpr.Compile();
    Console.WriteLine(func("Azusa", "s"));
}
```

> 未完待续。。。