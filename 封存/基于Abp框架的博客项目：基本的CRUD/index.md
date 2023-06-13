---
title: "基于Abp框架的博客项目：基本的CRUD"
date: 2023-04-18T14:31:42+08:00
tags: [Abp框架,"CSharp"]
categories: [".NET"]
series: ["基于Abp框架的博客项目"]
series_order: 3
---
上一节成功地在Abp框架中定义了博客系统最基本的实体，并使用 EFCore 和 SQL Server 进行了数据库迁移，然后生成了数据库表。  
本节将使用Abp框架进行基本的增删改查（CRUD），包括创建自定义仓储，编写业务逻辑层，创建数据传输对象（DTO），打开 Swagger UI 查看Abp框架自动生成的 API 端点。  

---

## 创建自定义仓储
关于[Abp中的仓储](https://docs.abp.io/zh-Hans/abp/latest/Repositories) 在上一节（生成种子数据那会儿）已经有所介绍。在使用EFCore时，Abp框架会自动地为实现`IAggregateRoot(聚合根)`接口的实体以及 `AbpDbContext` 中的 `DbSet<>` 对应的实体生成仓储并添加到依赖注入容器中。在类中注入 `IRepository<>` 接口即可访问对应实体类的**默认仓储**。  

在`.EntityFrameworkCore`项目中的`AbpBlogEntityFrameworkCoreModule`类中，配置了该Abp模块的服务，可以看到在`ConfigureServices`方法中加入了DbContext依赖：
```cs
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddAbpDbContext<AbpBlogDbContext>(options =>
    {
        /* 该方法会将所有聚合根对应的仓储加入依赖注入，
        如果方法的includeAllEntities参数设为true，
        则为所有DbSet中配置实体创建仓储并加入依赖注入 */
        options.AddDefaultRepositories(includeAllEntities: true);
    });

    Configure<AbpDbContextOptions>(options =>
    {
        /* The main point to change your DBMS.
        * See also AbpBlogMigrationsDbContextFactory for EF Core tooling. */
        options.UseSqlServer();
    });
}
```

Abp框架也支持自定义仓储，或是在默认仓储接口上扩展，下面将介绍如何创建自定义仓储接口并提供实现类：
### 自定义仓储接口
首先在领域层(`.Domain`项目)下创建名为 `Abstractions` 的文件夹，用于存放抽象接口，再创建名为 `Repositories` 的文件夹，用于存放仓储接口。接着为 `Post` 实体定义一个仓储接口，继承自Abp的默认仓储：
```cs
public interface IPostRepository : IRepository<Post,Guid>
{
    Task<List<Post>> FindByKeywordAsync(string keyword);
}
```
现在，领域层的项目结构如图：  
![领域层项目结构](./%E6%89%B9%E6%B3%A8%202023-04-18%20154333.png)


### 定义实现类
仓储的实现类应当定义在基础设施/数据访问层，即项目的 `.EntityFrameworkCore` 项目，在该项目中创建文件夹 `Repositories` 用于存放仓储接口的实现类。  
创建 `PostRepository` 类，实现刚才创建的 `IPostRepository` 接口。  
```cs
public class PostRepository : IPostRepository
{
}
```

![仓储接口](./%E6%89%B9%E6%B3%A8%202023-04-18%20155234.png)  

可以看到Abp的默认仓储接口需要实现大量的方法，这些方法不必自己编写，Abp已经提供了实现类，在使用EFCore时，将类继承自 `EfCoreRepository` 类即可实现大部分方法。实际上，如果使用Abp的默认仓储，框架默认就会把对应的 `EfCoreRepository` 类作为其实现类注入到依赖中。

```cs
public class PostRepository : EfCoreRepository<AbpBlogDbContext,Post,Guid>,IPostRepository
{
}
```

实现剩余的方法：
```cs
public class PostRepository : EfCoreRepository<AbpBlogDbContext,Post,Guid>,IPostRepository
{
    public PostRepository(IDbContextProvider<AbpBlogDbContext> dbContextProvider) : base(dbContextProvider) { }
    public async Task<List<Post>> FindByKeywordAsync(string keyword)
    {
        var dbCtx = await GetDbContextAsync();
        return await dbCtx.Posts.Where(post =>
            post.Markdown.Contains(keyword) || 
            post.Title.Contains(keyword) || 
            post.Tags.Any(tag => tag.Name.Contains(keyword)) ||
            (post.Category != null && post.Category.Name.Contains(keyword))
            ).ToListAsync();
    }
}
```
通过基类的`GetDbContextAsync`方法可获取到对应的 DbContext 实例，用它访问数据库并实现自定义接口中的方法。

### 将自定义仓储添加至依赖注入
Abp框架也会自动地将自定义仓储加入依赖，并且默认的仓储`IRepository`并不会失效，默认仓储不会使用刚才创建的自定义仓储的实现类。  

> Abp框架对于不同的ORM框架会采用不同的实现类，该项目使用EFCore，所以使用的是 `EfCoreRepository` 类。  
> 如果要将默认仓储的实现替换成自定义的实现类，需要在 `AbpBlogEntityFrameworkCoreModule` 类的 `ConfigureServices` 方法中对对应实体的默认仓储实现类进行替换。  
例如：  
>    ```cs
>    context.Services.AddAbpDbContext<BookStoreDbContext>(options =>
>    {
>        options.AddDefaultRepositories();
>
>        //将Book实体的默认仓储实现类替换为BookRepository
>        options.AddRepository<Book, BookRepository>();
>    });
>
>    ```
> 访问[Abp中的仓储](https://docs.abp.io/zh-Hans/abp/latest/Repositories)和[Abp中的EFCore集成](https://docs.abp.io/zh-Hans/abp/latest/Entity-Framework-Core#%E6%B7%BB%E5%8A%A0%E8%87%AA%E5%AE%9A%E4%B9%89%E4%BB%93%E5%82%A8)来了解更多关于自定义仓储的信息。

---

## 创建数据传输对象
定义供表示层使用的[数据传输对象(DTO)](https://docs.abp.io/zh-Hans/abp/latest/Data-Transfer-Objects)。  
> - 数据传输对象(DTO)应用在*应用层*和*表示层*并用来与客户端进行数据传输。它起到隔离*领域层*和*表示层*以及隐藏重要数据的作用。DTO也对传输数据的序列化起到优化，避免了对某些实体的嵌套关系进行重复地序列化。   
> - 将客户端对某个实体**有需要**的属性放入DTO，而对于客户端不需要或者是隐私的数据则不放入DTO。  
> - 对于数据传输对象的更多信息，参阅[Abp框架中的DTO](https://docs.abp.io/zh-Hans/abp/latest/Data-Transfer-Objects)。

### 创建响应数据传输对象
首先创建用于响应客户端使用的响应数据传输对象。  
在 `.Application.Contracts` 项目中创建 `ResponseDtos` 文件夹，用于存放响应DTO，并定义数据传输类 `PostDto`：
```cs
public class PostDto : EntityDto<Guid>
{
    public string Title { get; set; }
    public string MarkDown { get; set; }
    public CategoryDto Category { get; set; }
    public IEnumerable<TagDto> Tags { get; set; }
    public DateTime CreationTime { get; set; }
    public BlogUserDto Creator { get; set; }
    public DateTime? LastModificationTime { get; set; }
    public PostDto(string title, string markDown, CategoryDto? category, IEnumerable<TagDto> tags, DateTime creationTime, BlogUserDto creator, DateTime? lastModificationTime)
    {
        Title = title;
        MarkDown = markDown;
        Category = category;
        Tags = tags;
        CreationTime = creationTime;
        Creator = creator;
        LastModificationTime = lastModificationTime;
    }
}
```
> - Abp框架定义好了一些数据传输对象使用的基类/接口以减少重复工作，上面的 `PostDto` 类就继承自了 `EntityDto<Guid>` 类，它提前定义好了主键 `Id` 属性。
> - 和实体类一样，Abp也提供了一些审计DTO基类，参考[审计DTO](https://docs.abp.io/zh-Hans/abp/latest/Data-Transfer-Objects#%E5%AE%A1%E8%AE%A1dto)。

为其余的实体定义响应DTO：
```cs
public class CategoryDto : EntityDto<Guid>
{
    public string Name { get; set; }  
    public CategoryDto(string name)
    {
        Name = name;
    } 
}
```
```cs
public class TagDto : EntityDto<Guid>
{
    public required string Name { get; set; }
    public TagDto(string name)
    {
        Name = name;
    }
}
```
```cs
public class BlogUserDto : EntityDto<Guid>
{
    public required string UserName { get; set; }
    public required string Email { get; set; }
    public required string? Avatar { get; set; }
    public BlogUserDto(string userName, string email, string? avatar)
    {
        UserName = userName;
        Email = email;
        Avatar = avatar;
    }
}
```

### 创建请求数据传输对象
对于客户端的请求（比如创建，修改）也要创建数据传输对象，同样在 `.Application.Contracts` 项目中创建 `RequestDtos` 文件夹，用于存放请求DTO，定义数据传输 `CreatePostDto`：
```cs
public class CreatePostDto
{
    public string Title { get; set; }
    public string Markdown { get; set; }
    public string? CategoryName { get; set; }
    public IEnumerable<string> TagNames { get; set; }
    public Guid CreatorId { get; set; }
    
    public CreatePostDto(string title, string markdown, string? categoryName, IEnumerable<string> tagNames, Guid creatorId)
    {
        Title = title;
        Markdown = markdown;
        CategoryName = categoryName;
        TagNames = tagNames;
        CreatorId = creatorId;
    }
}
```
请求传输对象没有`Id`属性，故不需要继承自`EntityDto`类。
创建其他的请求数据传输对象：
```cs
public class CreateCategoryDto
{
    public string Name { get; set; }
    public CreateCategoryDto(string name)
    {
        Name = name;
    }
}
```

```cs
public class CreateTagDto
{
    public string Name { get; set; }
    public CreateTagDto(string name)
    {
        Name = name;
    }
}
```

```cs
public class CreateBlogUserDto
{
    public string UserName { get; set; }
    public string Password { get; set; }
    public string Email { get; set; }
    public string? Avatar { get; set; }
    public CreateBlogUserDto(string userName, string password, string email, string? avatar)
    {
        UserName = userName;
        Password = password;
        Email = email;
        Avatar = avatar;
    }
}
```
### 请求数据对象校验
请求数据传输对象需要做数据校验，可以为DTO类标注[`System.ComponentModel.DataAnnotations`](https://learn.microsoft.com/zh-cn/dotnet/api/system.componentmodel.dataannotations?view=net-7.0)命名空间中的特性（注解）来自动进行数据校验（在Abp框架中将由Abp进行校验而不是ASP .NET Core）。个人偏爱于使用[***FluentValidation***](https://fluentvalidation.net/)数据校验库，Abp框架也添加了它的[集成](https://docs.abp.io/zh-Hans/abp/latest/FluentValidation)：  
1. 添加`Volo.Abp.FluentValidation`Nuget包到项目中
    ```shell
    Install-Package Volo.Abp.FluentValidation
    ```
    > 注意！ 命名空间为***Volo.Abp.FluentValidation***而不是***Abp.FluentValidation***，后者属于 [ASP .NET Boilerplate](https://aspnetboilerplate.com/) 框架 (Volo.Abp的前身) ，而不是本项目使用的 [***Volo.Abp***](https://abp.io/)。 本项目中Abp框架的命名空间都应当是以 `Volo.Abp` 开头的，注意区分。
2. 在当前模块类(`AbpBlogApplicationContractsModule`)中添加依赖：
    ```cs
    [DependsOn(
        //...其他依赖
        typeof(AbpFluentValidationModule)//Abp FluentValidation集成
    )]
    public class AbpBlogApplicationContractsModule : AbpModule
    ```
3. 然后按照[FluentValidation文档](https://fluentvalidation.net/)创建校验器类：
    ```cs
    public class CreatePostDtoValidator : AbstractValidator<CreatePostDto>
    {
        public CreatePostDtoValidator()
        {
            RuleFor(dto => dto.Title).MaximumLength(64).WithMessage("文章的标题过长，最大长度为64");
            RuleFor(dto => dto.Markdown).MaximumLength(20000).WithMessage("文章过长，最多为20000字");
        }
    }
    ```
    Abp框架将会自动找到这个类并在创建时进行校验。  
    > 方便起见，该项目将Dto类的校验器类于Dto类放在了同一文件中，若有其他需求可转移至别的文件。
4. 创建其他的校验器类
    ```cs
    public class CreateCategoryDtoValidator : AbstractValidator<CreateCategoryDto>
    {
        public CreateCategoryDtoValidator()
        {
            RuleFor(dto => dto.Name).NotEmpty().WithMessage("类型名不得为空");
            RuleFor(dto => dto.Name).MaximumLength(32).WithMessage("类型名过长，最多为32个字");
        }
    }
    ```

    ```cs
   public class CreateTagDtoValidator : AbstractValidator<TagDto>
    {
        public CreateTagDtoValidator()
        {
            RuleFor(dto => dto.Name).NotEmpty().WithMessage("标签名不得为空");
            RuleFor(dto => dto.Name).MaximumLength(32).WithMessage("标签过长，最多为32个字");
        }
    }
    ```

    ```cs
    public class CreateBlogUserDtoValidator : AbstractValidator<CreateBlogUserDto>
    {
        public CreateBlogUserDtoValidator()
        {
            RuleFor(dto => dto.UserName).Length(4,24).WithMessage("用户名的长度需要为4-24之间");
            RuleFor(dto => dto.Password).Length(6, 32).WithMessage("密码的长度需要为6-32之间");
            RuleFor(dto => dto.Email).EmailAddress().WithMessage("邮件格式错误");
            When(dto => dto.Avatar is not null, () =>
                RuleFor(dto => dto.Avatar).Length(2000).WithMessage("头像的URL过长"));
        }
    }
    ```
### 实体类与数据传输对象的映射
在表示层和应用层之间传递数据时会频繁地将实体类与DTO类进行映射，过程重复而又繁琐，因此Abp框架使用了[***AutoMapper***](https://automapper.org/) 库以进行自动的对象映射。 

> - Abp框架使用*AutoMapper*对DTO和实体进行映射，开发者需要提供每个实体类与其对应DTO的映射规则，Abp框架才能正常进行映射并提供CRUD操作。  
> - 更多关于Abp框架的对象映射参阅官方文档中的[*对象到对象映射*](https://docs.abp.io/zh-Hans/abp/latest/Object-To-Object-Mapping)。

根据[AutoMapper文档](https://docs.automapper.org/en/latest/Getting-started.html)配置对象映射方案，Abp框架已经在`.Application`项目中创建好了配置AutoMapper所使用的[`Profile`](https://docs.automapper.org/en/latest/Configuration.html?highlight=profile#profile-instances)类，只需在这个类中添加对象映射即可：
```cs
public AbpBlogApplicationAutoMapperProfile()
{
    /* You can configure your AutoMapper mapping configuration here.
        * Alternatively, you can split your mapping configurations
        * into multiple profile classes for a better organization. */

    CreateMap<Post, PostDto>();
    CreateMap<Category, CategoryDto>();
    CreateMap<Tag, TagDto>();
    CreateMap<BlogUser, BlogUserDto>();

    CreateMap<CreatePostDto, Post>();
    CreateMap<CreateCategoryDto, Category>();
    CreateMap<CreateTagDto, Tag>();
    CreateMap<CreateBlogUserDto, BlogUser>();
}
```

---

## 定义应用程序服务
### 定义应用服务接口
在`.Application.Contracts`项目中创建`AppServices`文件夹，在此存放应用程序服务接口，也就是业务逻辑接口。  
创建`Blog`文件夹，并定义`IPostAppService`接口：
```cs
public interface IPostAppService : ICrudAppService<PostDto,Guid,PagedAndSortedResultRequestDto,CreatePostDto>
{}
```
上述的代码创建了一个`Post`实体的应用服务接口，并扩展了Abp框架提供的`ICrudAppService<>`接口，该接口定义了常用的CRUD方法，在类型参数中填上实体对应的DTO，该接口就能提供该DTO对应的CRUD方法。  
例如上面的代码在`ICrudAppService`接口中提供了`PostDto`、`Guid`类型主键、`PagedAndSortedResultRequestDto`和`CreatePostDto`，该接口提供了`GetAsync`,`GetListAsync`,`CreateAsync`,`UpdateAsync`和`DeleteAsync`方法。  

> - `ICrudAppService`接口有多个泛型变体，提供不同种类的DTO，它定义的方法就不一样。
> - `PagedAndSortedResultRequestDto`是Abp提供的DTO，它提供了用于分页查询和排序的属性，这个DTO是通用的。  
> - `ICrudAppService`接口实现了`IApplicationService`接口，Abp将会识别继承`IApplicationService`的接口并根据**接口定义的方法名**和相应的[**约定**](https://docs.abp.io/zh-Hans/abp/latest/API/Auto-API-Controllers#%E4%BE%8B%E5%AD%90)生成对应的 API 控制器。  
> - 开发者也可以不使用`ICrudAppService`接口，而是自己定义接口以及方法，只要接口实现了`IApplicationService`接口并且方法的名字符合[**约定**](https://docs.abp.io/zh-Hans/abp/latest/API/Auto-API-Controllers#%E4%BE%8B%E5%AD%90) ，Abp框架就可以自动生成API控制器。  
> - Abp框架生成API的配置是可以自定义的，在`.HttpApi.Host`项目的`AbpBlogHttpApiHostModule`类中定义了Abp根据应用服务生成API控制器的方法：  
>   ```cs
>   private void ConfigureConventionalControllers()
>   {
>       Configure<AbpAspNetCoreMvcOptions>(options =>
>       {
>           options.ConventionalControllers.Create(typeof(AbpBlogApplicationModule).Assembly);
>       });
>   }
>   ```
> - 更多关于Abp框架自动生成API控制器的信息参阅官方文档中的[自动API控制器](https://docs.abp.io/zh-Hans/abp/latest/API/Auto-API-Controllers)

定义其他实体的应用服务接口：
```cs
public interface ICategoryAppService : ICrudAppService<CategoryDto,Guid,PagedAndSortedResultRequestDto,CreateCategoryDto>
{
}
```

```cs
public interface ITagAppService:ICrudAppService<TagDto,Guid,PagedAndSortedResultRequestDto,CreateTagDto>
{
}
```

```cs
public interface IBlogUserAppService:ICrudAppService<BlogUserDto,Guid,PagedAndSortedResultRequestDto,CreateBlogUserDto>
{
}
```

### 实现应用服务接口
在应用层`.Application`项目中对上述接口进行实现，创建`AppServices`文件夹，然后编写实现类：

```cs
public class BlogUserAppService :
    CrudAppService<BlogUser, BlogUserDto, Guid, PagedAndSortedResultRequestDto, CreateBlogUserDto>, IBlogUserAppService
{
    private readonly IRepository<BlogUser, Guid> _userRepos;
    public BlogUserAppService(IRepository<BlogUser, Guid> repository, IRepository<BlogUser, Guid> userRepos) : base(repository)
    {
        _userRepos = userRepos;
    }
    public override async Task<BlogUserDto> CreateAsync(CreateBlogUserDto input)
    {
        if (await _userRepos.FindAsync(user => user.UserName == input.UserName) is not null)
        {
            throw new AbpValidationException(new List<ValidationResult> { new("用户名重复", new []{"Name"})});
        }
        return await base.CreateAsync(input);
    }
}
```

> - Abp也提供了`ICrudAppService`接口对应的实现类，它会自动注入相应实体的[***默认仓储***](https://docs.abp.io/zh-Hans/abp/latest/Repositories)，以及[`IObjectMapper`](https://docs.abp.io/zh-Hans/abp/latest/Object-To-Object-Mapping#iobjectmapper)对对象进行映射，然后实现`ICrudAppService`接口中对应的CRUD方法。  
> - `CrudAppService`类与`ICrudAppService`类似，都拥有不同的泛型版本，确保他们俩类型参数中的DTO一致。   
> - [`IObjectMapper`](https://docs.abp.io/zh-Hans/abp/latest/Object-To-Object-Mapping#iobjectmapper) 是Abp定义的一个可重用的对象映射接口，如果之前配置好了*AutoMapper*，那么Abp框架就会自动将该接口添加到依赖注入并且使用*AutoMapper*进行对象映射。  
> - `CrudAppService`类定义了虚方法，如果有需要修改的逻辑，可以重写类中的方法。上述代码中的`Category`实体在数据库中具有唯一约束，在插入数据库之前需要校验，因此重写了基类的`CreateAsync`方法，并添加了校验过程。
> - 代码中抛出的`AbpValidationException`将被Abp框架进行[异常处理](https://docs.abp.io/zh-Hans/abp/7.1/Exception-Handling#%E9%AA%8C%E8%AF%81%E9%94%99%E8%AF%AF)，对客户端响应***数据校验*** 相关的信息。

实现其他实体的应用服务接口：

```cs
public class TagAppService : CrudAppService<Tag, TagDto, Guid, PagedAndSortedResultRequestDto, CreateTagDto>,
    ITagAppService
{
    private readonly IRepository<Tag, Guid> _tagRepos;

    public TagAppService(IRepository<Tag, Guid> repository, IRepository<Tag, Guid> tagRepos) : base(repository)
    {
        _tagRepos = tagRepos;
    }

    public override async Task<TagDto> CreateAsync(CreateTagDto input)
    {
        if (await _tagRepos.FindAsync(tag => tag.Name == input.Name) is not null)
        {
            throw new AbpValidationException(new List<ValidationResult> { new("标签名重复", new []{"Name"})});
        }

        return await base.CreateAsync(input);
    }
}
```

```cs
public class BlogUserAppService :
    CrudAppService<BlogUser, BlogUserDto, Guid, PagedAndSortedResultRequestDto, CreateBlogUserDto>, IBlogUserAppService
{
    private readonly IRepository<BlogUser, Guid> _userRepos;
    public BlogUserAppService(IRepository<BlogUser, Guid> repository, IRepository<BlogUser, Guid> userRepos) : base(repository)
    {
        _userRepos = userRepos;
    }
    public override async Task<BlogUserDto> CreateAsync(CreateBlogUserDto input)
    {
        if (await _userRepos.FindAsync(user => user.UserName == input.UserName) is not null)
        {
            throw new AbpValidationException(new List<ValidationResult> { new("用户名重复", new []{"Name"})});
        }
        return await base.CreateAsync(input);
    }
}
```


```cs
public class PostAppServices : CrudAppService<Post,PostDto,Guid,PagedAndSortedResultRequestDto,CreatePostDto> ,IPostAppService
{
    private readonly IRepository<BlogUser, Guid> _userRepos;
    private readonly IRepository<Category, Guid> _categoryRepos;
    private readonly IRepository<Tag, Guid> _tagRepos;
    private readonly IRepository<Post, Guid> _postRepos;
    public PostAppServices(IRepository<Post, Guid> repository, IRepository<BlogUser, Guid> userRepos, IRepository<Category, Guid> categoryRepos, IRepository<Tag, Guid> tagRepos, IRepository<Post, Guid> postRepos) : base(repository)
    {
        _userRepos = userRepos;
        _categoryRepos = categoryRepos;
        _tagRepos = tagRepos;
        _postRepos = postRepos;
    }

    public override async Task<PostDto> CreateAsync(CreatePostDto input)
    {
        var validationErrors = new List<ValidationResult>();
        
        //获取对应的用户，分类和标签实体
        var creator = await _userRepos.FindAsync(user => user.Id == input.CreatorId);
        var category = await _categoryRepos.FindAsync(category => category.Name == input.CategoryName);
        var tags = new List<Tag>();
        foreach (var name in input.TagNames)
        {
            var tag = await _tagRepos.FindAsync(tag => tag.Name == name);
            if (tag is null)
                validationErrors.Add(new ValidationResult($"标签[{name}]不存在",new []{nameof(input.TagNames)}));
            else
                tags.Add(tag);
        }
        
        //校验
        if (creator is null)
            validationErrors.Add(new ValidationResult("用户不存在",new []{nameof(input.CreatorId)}));
        if (category is null)
            validationErrors.Add(new ValidationResult($"类型[{input.CategoryName}]不存在", new[] { nameof(input.CategoryName) }));

        //不通过，则抛出异常
        if (!validationErrors.IsNullOrEmpty())
            throw new AbpValidationException(validationErrors);

        //映射对应实体，并使用获取到的实体进行赋值
        var post = ObjectMapper.Map<CreatePostDto, Post>(input);
        post.Category = category;
        post.Creator = creator!;
        post.Tags = tags;

        //插入表，返回Dto
        await _postRepos.InsertAsync(post);
        return ObjectMapper.Map<Post,PostDto>(post);
    }
}
```
> - `Post`实体的校验比较复杂，实现类重写了`CreateAsync`方法，并且手动对对象进行映射，校验，并且插入数据库。


---

## Swagger UI
Abp框架使用了SwashBukle.AspNetCore运行***Swagger UI***，运行`.HttpApi.Host`项目，打开Swagger，就可以看到刚才定义的AppService已经被Abp框架自动生成为 API 接口：  
![Swagger](./%E6%89%B9%E6%B3%A8%202023-04-20%20234155.png "Swagger UI")

测试一下API接口，应用服务，对象映射，数据校验以及仓储是否正常运作：  
![测试接口](./%E6%89%B9%E6%B3%A8%202023-04-22%20020124.png "测试新增")  
![测试接口](./%E6%89%B9%E6%B3%A8%202023-04-22%20020320.png "AutoMapper以及FluentValidation正常地工作了")
![测试接口](./%E6%89%B9%E6%B3%A8%202023-04-22%20022858.png "测试正确的数据")
![测试接口](./%E6%89%B9%E6%B3%A8%202023-04-22%20022817.png "成功地添加了实体")

其他接口还需自行测试

---

> - 如果开发者按照Abp框架的**约定**来编写代码，大部分基本的业务逻辑Abp框架都会**自动**实现。但遵守Abp的约定**并不是强制**的，开发者可以**重写**Abp的虚方法，**自主实现**Abp提供的接口，或是**自定义并手动注册服务**。但越是按照Abp框架的约定来编写代码，框架就越是**自动化**，提供更高的开发效率。

至此，项目已经可以使用Abp框架进行基本的增删改查。