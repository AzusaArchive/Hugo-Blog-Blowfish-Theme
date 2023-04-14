---
title: "基于Abp框架的博客项目：实体和数据库迁移"
date: 2023-04-14T20:56:41+08:00
tags: [Abp框架]
categories: [".NET"]
series: ["基于Abp框架的博客项目"]
series_order: 2
---

上一节安装了Abp框架并创建了解决方案，并成功运行了项目。  
本篇将会从定义实体类开始，使用 EFCore 连接数据库，并用 Code First 模式配置实体类，创建数据库DbContext并对数据库进行迁移，项目将采用SQL Server数据库。  

---

## 实体创建
### 需求分析
博客系统最基本的实体有 `文章(Post)`， `类别(Category)`， `标签(Tag)`，为方便今后的登录验证，先创建个`用户(BlogUser)`实体。
- `文章(Post)`，即最基本的博客文章。
- `类别(Category)`，文章的类别，一篇文章只能有一个类别，一个类别有多个文章。
- `标签(Tag)`，文章带有的标签，与文章为多对多关系。
- `用户(BlogUser)`，顾名思义。

实体存放在领域层下，所以在`Azusa.AbpBlog.Domain`项目下新建文件夹`Entities`，接着创建`Blog`文件夹，存放博客系统的实体，创建`Identity`文件夹，存放用户实体。

### Abp框架中的实体
以 `文章(Post)` 为例，创建一个Abp框架实体：
``` cs
using System;
using System.Collections.Generic;
using Azusa.AbpBlog.Identity;
using Volo.Abp.Auditing;
using Volo.Abp.Domain.Entities;

namespace Azusa.AbpBlog.Entities.Blog;

public class Post : IEntity<Guid>,IHasCreationTime,IMustHaveCreator<BlogUser>,IHasModificationTime,IHasDeletionTime
{
    public Guid Id { get; }
    
    public string Title { get; set; }
    public string Markdown { get; set; }
    public Category Category { get; set; }
    public ICollection<Tag> Type { get; set; }
    
    public DateTime CreationTime { get; }
    public Guid CreatorId { get; }
    public BlogUser Creator { get; }
    public DateTime? LastModificationTime { get; }
    public bool IsDeleted { get; }
    public DateTime? DeletionTime { get; }
    
    public object[] GetKeys()
    {
        return new object[]{Id};
    }
}
```
Abp框架提供了许多实体基类和接口，这里实现了：
- `IEntity`: 实现`Id`属性和`GetKeys`方法并使用`Guid`作为主键类型，`GetKeys`方法返回一个主键数组用于排列（Abp考虑到复合主键所以返回数组）。  
- `IHasCreationTime`: 实现`创建时间(CreationTime)`属性。  
- `IMustHaveCreator<BlogUser>`: 实现创建者`Id(CreatorId)`以及其实体类导航属性`Creator`（适用于EFCore），创建者的实体类型为`BlogUser`。
- `IHasModificationTime`: 实现`最近修改时间(LastModificationTime)`属性。
- `IHasDeletionTime`: 实现**软删除**，`IsDeleted`属性以及`DeletionTime`属性。
如果实体继承或是实现了这些Abp框架提供的基类或是接口，Abp框架就会尽可能的自动管理这些属性。  

#### 关于软删除
在使用Abp框架提供的仓储接口对实现软删除接口进行删除时，会自动将`IsDeleted`属性置为`true`，而不是删除实体，更多关于Abp框架中软删除的实现，参考[数据过滤文档](https://docs.abp.io/zh-Hans/abp/latest/Data-Filtering)   


#### 关于领域驱动设计
在Abp框架中定义了领域驱动设计所用的`聚合根(AggregateRoot)`基类和接口，它们提供了如并发控制，审计等各种实现聚合根所需的各种方法，并且Abp框架默认会自动为聚合根创建并注入[仓储](https://docs.abp.io/zh-Hans/abp/latest/Repositories)。该项目相对简单，为避免繁琐，实体不实现聚合根接口，而是手动地实现一些需要的，Abp提供的审计接口，并且手动创建该实体的**仓储**。关于Abp的聚合根和实体，参考[实体文档](https://docs.abp.io/zh-Hans/abp/latest/Repositories)。  

> *"ABP框架不强制你应用任何DDD规则或模式.但是,当你准备应用的DDD规则或模式时候,ABP会让这变的可能而且更简单.文档同样遵循这个原则."*

关于更多实体信息以及基类和接口也参考[实体文档](https://docs.abp.io/zh-Hans/abp/latest/Entities)  

### 编写实体类
待续...
