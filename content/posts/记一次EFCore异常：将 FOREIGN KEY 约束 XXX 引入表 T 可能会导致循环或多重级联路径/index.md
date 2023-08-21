---
title: "记一次EFCore异常：将 FOREIGN KEY 约束 XXX 引入表 T 可能会导致循环或多重级联路径"
date: 2023-07-04T10:07:32+08:00
tags: ["EFCore","C#", "Bug修复"]
categories: [".NET"]
series: []
---

## 概述
在EFCore应用迁移(`Update Database`)时，由于实体配置有误而出现异常：  
***Introducing FOREIGN KEY constraint may cause cycles or multiple cascade paths***  
***(将FOREIGN KEY约束XXX引入表T可能会导致循环或多重级联路径)***

## 原因
在配置EFCore实体时，某个实体同时外键依赖了多个实体并配置了级联关系，而依赖的实体直接也有同样的级联关系。这样将会在根实体被删除时导致多重级联路径。以下面的代码为例：

```csharp
public class User
{
    ...
}

public class Post
{
    public User User { get; set; }
    ...
}

public class PostConfig : IEntityTypeConfiguration<Post>
{
    public void Configure(EntityTypeBuilder<Post> builder)
    {
        builder.HasOne(p => p.User).WithMany().OnDelete(DeleteBehavior.Cascade);
    }
}

public class PostReply
{
    public User User { get; set; }
    public Post Post { get; set }
    ...
}

public class PostReplyConfig : IEntityTypeConfiguration<PostReply>
{
    public void Configure(EntityTypeBuilder<PostReply> builder)
    {
        ...
        builder.HasOne(p => p.Post).WithMany(p => p.Replies).OnDelete(DeleteBehavior.Cascade);
        builder.HasOne(p => p.User).WithMany().OnDelete(DeleteBehavior.Cascade);
    }
}   
```

PostReply实体配置了两个一对多外键依赖`User`和`Post`，并将删除模式改为了级联删除。  
而`Post`实体同时也配置了与`User`实体的一对多关系，删除模式也是级联删除。

此时，如果删除了一个`User`实体，数据库将会：
- 直接级联删除其对应的`Post`实体，然后间接地删除`Post`实体对应的`PostReply`实体。 `User -> Post -> PostReply`
- 直接级联删除其对应的`PostReply`实体。 `User -> PostReply`。

可以发现，此时对`PostReply`实体的级联删除路径有两条，这将会触发异常。

> 将 FOREIGN KEY 约束 'FK_PostReply_User_UserId' 引入表 'PostReply' 可能会导致循环或多重级联路径。请指定 ON DELETE NO ACTION 或 ON UPDATE NO ACTION，或修改其他 FOREIGN KEY 约束。

## 解决方法
比较好的解决方法是将`PostReply`实体配置中的任意一个一对多关系的删除模式改为非级联的，比如`SetNull`。  
这里推荐修改对`User`实体的一对多关系，这样当外键`Post`被删除时，实体也能正确的被级联删除。  
修改后的配置：
```csharp
public class PostReplyConfig : IEntityTypeConfiguration<PostReply>
{
    public void Configure(EntityTypeBuilder<PostReply> builder)
    {
        ...
        builder.HasOne(p => p.Post).WithMany(p => p.Replies).OnDelete(DeleteBehavior.Cascade);
        builder.HasOne(p => p.User).WithMany().OnDelete(DeleteBehavior.SetNull);
    }
}  
```

遇到异常的开发者可能在配置`PostReply`实体时会这样想：“当`User`实体被删除时，我想要其对应的`Post`和`PostReply`都被级联删除。所以应当在配置一对多关系时为`Post`外键和`PostReply`外键都指定级联删除行为。”  

但此时，`User`实体和`Post`实体已经含有一对多关系，并且配置了级联删除。对于`PostReply`实体，只需要配置其`Post`外键的级联关系即可。`User`外键的级联关系是多余的。

所以当遇到此类异常时，只需要把出现异常实体的外键关系链上“**距离较远**”的实体的级联关系都删除，只保留“**最近**”的那一个即可。
如上面的例子中，`PostReply`实体的外键关系链为`User -> Post -> PostReply`，其中`User`和`Post`的级联路径发生了冲突，此时需要删除**较远端**的实体`User`，保留**最近**的实体`Post`。

>- StackOverFlow例子:<https://stackoverflow.com/questions/17127351/introducing-foreign-key-constraint-may-cause-cycles-or-multiple-cascade-paths>  
>- 官方文档-级联删除:<https://learn.microsoft.com/zh-cn/ef/core/saving/cascade-delete>