---
title: ".NET 表达式树入门"
date: 2023-03-15T15:44:03+08:00
tags: ["CSharp","Linq"]
categories: [".NET"]
series: []
---

## .NET 表达式树
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
`(str,value) => !string.IsNullOrWhiteSpace(str) && str.Contains(value)`
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

## 一些实际的应用
```cs
/// <summary>
/// 推断实体属性的特性和名称，并根据搜索关键字构建过滤器表达式树
/// 注意，反射推断比较消耗性能
/// </summary>
/// <param name="keyword"></param>
/// <returns></returns>
private static Expression<Func<TEntity, bool>> BuildKeywordSearchExpression(string keyword)
{
    //获取实体类所有公用|实例属性
    var propInfos = typeof(TEntity).GetProperties(BindingFlags.Public | BindingFlags.Instance | BindingFlags.GetProperty);
    //获取所有具有SearchKeywordAttribute特性的属性，作为目标属性
    var targetProps = propInfos.Where(info => info.GetCustomAttribute<SearchKeywordAttribute>() is not null);
    //如果没有任何属性带SearchKeywordAttribute则按照属性名匹配，如果符合"name"或是"title"或是"content"，那就作为目标属性
    if (!targetProps.Any())
        targetProps = propInfos.Where(info => info.Name.ToUpper() is "NAME" or "TITLE" or "CONTENT");
    //没有任何属性能够匹配则异常
    if (!targetProps.Any())
        throw new ServerErrorException("没有任何能够进行过滤查询的属性，请在属性上添加[SearchKeywordAttribute]启用过滤");
    //检查属性是否为字符串类型
    if (!targetProps.All(info => info.PropertyType == typeof(string)))
        throw new ServerErrorException("过滤属性必须是字符串");
    
    //构建 e => (e.Name || Title || Content || [property with attribute]).Contain(keyword) 表达式树
    //参数e表达式
    var entityExpr = Expression.Parameter(typeof(TEntity),"entity");
    //常量关键词表达式
    var keywordExpr = Expression.Constant(keyword);
    //所有目标属性的表达式
    var propExprs = targetProps.Select(info => Expression.Property(entityExpr,info)).ToArray();
    //所有的属性调用.Contain(keyword)并且进行或运算
    var firstProp = propExprs.First();
    Expression resultExpr = Expression.Call(firstProp,
        typeof(string).GetMethod(nameof(string.Contains), new[] { typeof(string) })!, keywordExpr);
    for (int i = 1; i < propExprs.Length; i++)
    {
        var containsExpr = Expression.Call(propExprs[i],
            typeof(string).GetMethod(nameof(string.Contains), new[] { typeof(string) })!, keywordExpr);
        resultExpr = Expression.Or(resultExpr,containsExpr);
    }
    //构造成委托
    var lambda = Expression.Lambda<Func<TEntity, bool>>(resultExpr, entityExpr);
    Console.WriteLine(lambda.ToString());
    return lambda;
}
```
上述代码根据所给的关键词，使用反射搜索类型参数中的`TEntity`类型的属性，探测带有`SearchKeywordAttribute`特性或是名为"*Name*","*Title*"或是"*Content*"的属性，并使用它们构造一个用于多个属性模糊查询的表达式树，返回一个`Expression<Func<TEntity,bool>>`表达式委托。

这样的一个表达式可以自动地识别某个实体类的属性，并且根据这些属性通过Linq对IQuerable进行模糊查询，例如：
```cs
public class Post : IEntity<Guid>,IHasCreationTime,IHasModificationTime
{
    public Guid Id { get; set; }
    [SearchKeyword]
    public string Title { get; set; }
    [SearchKeyword]
    public string Markdown { get; set; }
    public Guid? CategoryId { get; set; }
    public Category? Category { get; set; }
    public ICollection<Tag> Tags { get; set; }
    
    public DateTime CreationTime { get; set; }
    public DateTime? LastModificationTime { get; set; }
    
    private Post() { }//EFCore

    public Post(string title, string markdown = "", Category? category = null, ICollection<Tag>? tags = null)
    {
        Id = SequentialGuidGenerator.Create();
        Title = title;
        Markdown = markdown;
        Category = category;
        Tags = tags ?? new List<Tag>();
    }
}

public Task<List<Post>> SearchAsync(string keyword)
{
    return dbContext.Posts.Where(BuildKeywordSearchExpression(keyword)).ToListAsync();
}
```
方法中将根据实体属性上的特性自动生成 `post => post.Title.Contains(keyword) || post.Markdown.Contains(keyword)` 表达式，并通过`IQuerable`和EFCore生成SQL语句对实体进行查询。
```sql
info: 2023/4/24 23:30:37.812 RelationalEventId.CommandExecuted[20101] (Microsoft.EntityFrameworkCore.Database.Command)
Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
SELECT [t].[Id], [t].[CategoryId], [t].[CreationTime], [t].[LastModificationTime], [t].[Markdown], [t].[Title]
FROM [T_Posts] AS [t]
WHERE (CASE
    WHEN [t].[Title] LIKE N'%net%' THEN CAST(1 AS bit)
    ELSE CAST(0 AS bit)
END | CASE
    WHEN [t].[Markdown] LIKE N'%net%' THEN CAST(1 AS bit)
    ELSE CAST(0 AS bit)
END) = CAST(1 AS bit)

```


参考文章
> - https://masuit.com/1795?t=vou0ts7ora4g#%CE%BB%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%A0%91%E7%9A%84%E9%AB%98%E7%BA%A7%E7%94%A8%E6%B3%95
> - https://cloud.tencent.com/developer/article/1817790  
