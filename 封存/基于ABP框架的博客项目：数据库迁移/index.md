﻿---
title: "基于Abp框架的博客项目：实体和数据库迁移"
date: 2023-04-14T20:56:41+08:00
tags: [Abp框架,"CSharp"]
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

---

### Abp框架中的实体
以 `文章(Post)` 为例，创建一个Abp框架实体：
``` cs
public class Post : Entity<Guid>,IHasCreationTime,IMustHaveCreator<BlogUser>,IHasModificationTime,IHasDeletionTime
{
    public string Title { get; set; }
    public string Markdown { get; set; }
    public Guid? CategoryId { get; set; }
    public Category? Category { get; set; }
    public ICollection<Tag> Tags { get; set; }
    
    public DateTime CreationTime { get; set; }
    public Guid CreatorId { get; set; }
    public BlogUser Creator { get; set; }
    public DateTime? LastModificationTime { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletionTime { get; set; }
    
    private Post() { }//EFCore

    public Post(Guid id, BlogUser creator, string title, string markdown = "", Category? category = null, ICollection<Tag>? tags = null):base(id)
    {
        Title = title;
        Markdown = markdown;
        Category = category;
        Tags = tags ?? new List<Tag>();
        Creator = creator;
    }
}
```
Abp框架提供了许多实体基类和接口，这里实现了：
- `Entity<Guid>`: 实体基类，实现了`Id`属性和`GetKeys`方法并使用`Guid`作为主键类型，`GetKeys`方法返回一个主键数组用于排列（Abp考虑到复合主键所以返回数组）。  
- `IHasCreationTime`: 实现`创建时间(CreationTime)`属性。  
- `IMustHaveCreator<BlogUser>`: 实现创建者`Id(CreatorId)`以及其实体类导航属性`Creator`（适用于EFCore），创建者的实体类型为`BlogUser`。
- `IHasModificationTime`: 实现`最近修改时间(LastModificationTime)`属性。
- `IHasDeletionTime`: 实现**软删除**，`IsDeleted`属性以及`DeletionTime`属性。
> - 如果实体继承或是实现了这些Abp框架提供的基类或是接口，并且后续的仓储以及业务逻辑的编写都***按照Abp框架的约定的话*** ，Abp框架就会尽可能的自动管理这些属性。  

#### 关于软删除
在使用Abp框架提供的仓储接口对实现软删除接口进行删除时，会自动将`IsDeleted`属性置为`true`，而不是删除实体，更多关于Abp框架中软删除的实现，参考[数据过滤文档](https://docs.abp.io/zh-Hans/abp/latest/Data-Filtering)   


#### 关于领域驱动设计
在Abp框架中定义了领域驱动设计所用的`聚合根(AggregateRoot)`基类和接口，它们提供了如并发控制，审计等各种实现聚合根所需的各种方法，并且Abp框架默认会自动为聚合根创建并注入[仓储](https://docs.abp.io/zh-Hans/abp/latest/Repositories)。该项目相对简单，为避免繁琐，实体使用贫血模型，不实现Abp提供的聚合根接口，而是手动地实现一些需要的、Abp提供的审计接口。关于Abp的聚合根和实体，参考[实体文档](https://docs.abp.io/zh-Hans/abp/latest/Repositories)。  

> *"ABP框架不强制你应用任何DDD规则或模式.但是,当你准备应用的DDD规则或模式时候,ABP会让这变的可能而且更简单.文档同样遵循这个原则."*


#### 关于Guid类型主键
Guid类型很好地避免了主键重复的情况，但因为Guid的无序性，数据库根据Guid主键创建的索引会在插入时带来严重的性能影响(因为插入新记录可能需要对现有记录进行重新排序)。  
Abp框架提供了解决方案：[Guid生成器](https://docs.abp.io/zh-Hans/abp/latest/Guid-Generation)，框架提供了`IGuidGenerator`接口，它可以生成连续的Guid值，这对数据库的聚集索引非常重要，所以不要用`Guid.NewGuid`方法生成Guid值，而是用Abp框架的Guid生成器。  
> - `IGuidGenerator`接口默认会被Abp框架注册到依赖注入，它的实现类是`SequentialGuidGenerator`。
> - 使用Abp的应用服务基类(`ApplicationService`)时无需注入`IGuidGenerator`，Abp框架已经将其定义在`ApplicationService`的`GuidGenerator`属性中，调用它的`Create`方法即可。
> - 后续如果使用了Abp的应用服务基类（比如`CrudAppService`)，继承自`Entity<>`基类的实体Id将会自动被Abp框架生成，但如果实体仅只是实现了`IEntity<>`则需要开发者手动在应用服务中生成Guid。
> - 关于更多实体信息以及基类和接口参考[基类和接口的审计属性](https://docs.abp.io/zh-Hans/abp/latest/Entities#%E5%9F%BA%E7%B1%BB%E5%92%8C%E6%8E%A5%E5%8F%A3%E7%9A%84%E5%AE%A1%E8%AE%A1%E5%B1%9E%E6%80%A7)  

---

### 编写实体类
按照Abp框架实体类的规则，编写其他实体类：  

```cs
public class Category : Entity<Guid>
{
    public string Name { get; set; }

    public ICollection<Post> Posts { get; set; }

    private Category() { }//EFCore

    public Category(Guid id, string name) : base(id)
    {
        Name = name;

        Posts = new List<Post>();
    }
}
```

```cs
public class Tag : Entity<Guid>
{
    public string Name { get; set; }
    
    public ICollection<Post> Posts { get; set; }

    private Tag() { }//EFCore
    public Tag(Guid id, string name):base(id)
    {
        Name = name;
        
        Posts = new List<Post>();
    }
}
```

```cs
public class BlogUser : FullAuditedEntity<Guid>
{
    public string UserName { get; set; }
    public string PasswordHash { get; set; }
    public string Email { get; set; }
    public string? Avatar { get; set; }

    private BlogUser() { }//EFCore
    public BlogUser(Guid id, string userName, string password, string email, string? avatar = null) : base(id)
    {
        UserName = userName;
        PasswordHash = SecurePasswordHasher.Hash(password);
        Email = email;
        Avatar = avatar;
    }
}
```

---

## 使用EFCore进行数据库迁移
### 在DbContext中添加实体
首先在`Azusa.AbpBlog.EntityFrameworkCore`项目的EntityFrameworkCore文件夹中，找到`AbpBlogDbContext`类（这个类是根据项目名命名的），Abp框架已经自动生成了EFCore需要的DbContext类。在类中添加刚才创建的实体：
```cs
...
public class AbpBlogDbContext :
    AbpDbContext<AbpBlogDbContext>,
    IIdentityDbContext,
    ITenantManagementDbContext
{
    /* Add DbSet properties for your Aggregate Roots / Entities here. */
    public DbSet<Post> Posts { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Tag> Tags { get; set; }
    public DbSet<BlogUser> BlogUsers { get; set; }
...

### 配置表映射
在`AbpBlogDbContext`的`OnModelCreating`方法中添加实体配置：
```cs
protected override void OnModelCreating(ModelBuilder builder)
{
    ...
    /* Configure your own tables/entities inside here */
    //从程序集中获取实体配置
    builder.ApplyConfigurationsFromAssembly(GetType().Assembly);
    ...
}
```
项目将实体的配置单独写在类中，在`Azusa.AbpBlog.EntityFrameworkCore`项目中创建`EntityConiguration`文件夹，用于存放配置EFCore的实体映射配置。  
> - [EFCore配置实体](https://learn.microsoft.com/zh-cn/ef/core/modeling/)

PostConfig.cs
```cs
public class PostEntityConfig : IEntityTypeConfiguration<Post>
{
    public void Configure(EntityTypeBuilder<Post> builder)
    {
        builder.ToTable("T_Posts");
        builder.Property(p => p.Title).HasMaxLength(64);
        //Category一对多关系
        builder.HasOne(p => p.Category).WithMany(c => c.Posts);
        //Tag多对多关系
        builder.HasMany(p => p.Tags).WithMany(t => t.Posts);
        //BlogUser一对多关系
        builder.HasOne(p => p.Creator).WithMany().HasForeignKey(p => p.CreatorId);
    }
}
```

CategoryEntityConfig.cs
```cs
public class CategoryEntityConfig : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.ToTable("T_Categories");
        builder.Property(c => c.Name).HasMaxLength(64);
        builder.HasIndex(c => c.Name).IsUnique();
        builder.HasMany(c => c.Posts).WithOne(p => p.Category);
    }
}
```

TagEntityConfig.cs
```cs
public class TagEntityConfig : IEntityTypeConfiguration<Tag>
{
    public void Configure(EntityTypeBuilder<Tag> builder)
    {
        builder.ToTable("T_Tags");
        builder.Property(t => t.Name).HasMaxLength(64);
        builder.HasIndex(t => t.Name).IsUnique();
        builder.HasMany(t => t.Posts).WithMany(p => p.Tags);
    }
}
```

BlogUserEntityConfig.cs
```cs
public class BlogUserEntityConfig : IEntityTypeConfiguration<BlogUser>
{
    public void Configure(EntityTypeBuilder<BlogUser> builder)
    {
        builder.ToTable("T_BlogUsers");
        builder.Property(user => user.UserName).HasMaxLength(24);
        builder.HasIndex(user => user.UserName).IsUnique();
        builder.Property(user => user.PasswordHash).HasMaxLength(200);
        builder.Property(user => user.Email).HasMaxLength(200);
    }
}
```

---

### 配置种子数据
在`.Domain`项目中创建`AbpBlogDataSeedContributor`类。
```cs
public class AbpBlogDataSeedContributor: IDataSeedContributor,ITransientDependency
{
    private readonly IPostRepository _postRepos;
    private readonly IRepository<BlogUser, Guid> _userRepos;
    private readonly IRepository<Category, Guid> _categoryRepos;
    private readonly IGuidGenerator _guidGenerator;
    
    public AbpBlogDataSeedContributor(IPostRepository postRepos, IRepository<BlogUser, Guid> userRepos, IGuidGenerator guidGenerator, IRepository<Category, Guid> categoryRepos)
    {
        _postRepos = postRepos;
        _userRepos = userRepos;
        _guidGenerator = guidGenerator;
        _categoryRepos = categoryRepos;
    }

    public async Task SeedAsync(DataSeedContext context)
    {
        if (await _postRepos.GetCountAsync() <= 0)
        {
            var user = await _userRepos.FindAsync(user => user.UserName == "AzusaAdmin");
            if (user == null)
            {
                user = new BlogUser(_guidGenerator.Create(), "AzusaAdmin", "VanitasVanitatum",
                    "2817212736@qq.com");
                await _userRepos.InsertAsync(user,autoSave:true);
            }
            
            await _postRepos.InsertAsync(new Post(_guidGenerator.Create(), user, "Hello World", "初始化数据"),autoSave:true);
        }

        if (await _categoryRepos.FindAsync(category => category.Name == "其他") is null)
        {
            await _categoryRepos.InsertAsync(new Category(_guidGenerator.Create(), "其他"));
        }
        
    }
}
```

Abp框架提供了[种子数据系统](https://docs.abp.io/zh-Hans/abp/latest/Data-Seeding)用来创建数据库的**初始数据**，创建一个`IDataSeedContributor`接口的实现类来配置种子数据，框架将会自动将其加入依赖注入并在`.DbMigrator`项目运行并应用数据库迁移时执行，由此来创建数据库的初始数据。

上面的类在`SeedAsync`方法中配置了种子数据的生成，在名为 *"AzusaAdmin"* 的用户不存在时自动创建，并且创建一个初始化文章(`Post`)实体和初始化类型(`Category`)实体。类中注入了相应表的**仓储**以及上述的`IGuidGenerator`接口，Abp已经自动将常用的服务和默认**仓储**加入了依赖注入容器。

---

### 生成数据库迁移
配置完后，生成`AbpBlogDbContext`的数据库迁移。   
```shell
add-migration Add_Post_Category_Tag -Context AbpBlogDbContext
```
使用Visual Studio包管理器命令行或是其他工具进行EFCore数据库迁移。

> - 注意，使用包管理器命令行生成EFCore迁移需要先安装 `Microsoft.EntityFrameworkCore.Tools` 包，Abp框架默认会在创建项目的时候预安装。

然后运行`.DbMigrator`项目，Abp框架自动将DbContext加入了依赖，并且通过[设计时工厂](https://learn.microsoft.com/zh-cn/ef/core/cli/dbcontext-creation?tabs=dotnet-core-cli)根据`.DbMigrator`项目下的`appsettings.json`配置文件对DbContext进行配置。该项目会**自动应用存在的数据迁移脚本**，并且根据刚才的配置创建**种子数据**。
> - 如果使用`Update-Database`或是其他迁移工具进行迁移，并不会产生**种子数据**。

![数据库表](./%E6%89%B9%E6%B3%A8%202023-04-16%20234657.png)  
执行完后生成了一堆表，大部分是Abp框架的配置，日志，Identity框架，多租户所用的表，其实用不上。
(在之后可能会对项目和表进行简化吧...

> - [EFCore迁移文档](https://learn.microsoft.com/zh-cn/ef/core/managing-schemas/migrations/?tabs=dotnet-core-cli)

---

至此，项目基本实体以及数据库迁移应当创建完毕。