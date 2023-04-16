---
title: "基于Abp框架的博客项目：使用Abp框架创建解决方案"
date: 2023-04-13T18:24:04+08:00
tags: ["Abp框架"]
categories: [".NET"]
series: ["基于Abp框架的博客项目"]
series_order: 1
---


## 开始

该系列文章记录ABP框架的学习过程，将使用ASP .NET Core ABP框架开发一个博客项目：数据库选用SQL Server + EFCore，前端框架选用Blazor WebAssembly，ABP框架的版本为7.1，项目将使用[领域驱动设计](https://docs.abp.io/zh-Hans/abp/latest/Domain-Driven-Design)分层。


[ABP快速入门文档](https://docs.abp.io/zh-Hans/abp/latest/Tutorials/Todo/Index?UI=Blazor&DB=EF)
官方文档是最好的教程

---

## 安装ABP框架
首先使用.NET CLI安装ABP框架：
``` shell
dotnet tool install -g Volo.Abp.Cli
```

## 安装Redis
参考 [Redis官方文档](https://redis.io/docs/getting-started/installation/) 安装Redis客户端，  
或是访问 http://redis.com/try-free/ 可使用 Redis Cloud 免费提供的服务器（虽然只有30M容量）。

---

## 安装NPM
访问 [Node.js官网](https://nodejs.org/en/download/) 下载并安装Node.js客户端。

---

## 创建ABP解决方案

切换到终端到项目文件夹，然后用ABP CLI创建项目：
```shell
abp new Azusa.AbpBlog -u blazor --separate-auth-server
```

> - 项目名为 *Azusa.AbpBlog*  
> - -u 指定前端框架，该项目使用 Blazor WASM   
> - --separate-auth-server 参数用于将Identity Server应用程序与API主机应用程序分隔开. 如果未指定, 则服务器上将只有一个端点.  
> - 详细的命令见[ABP CLI 文档](https://docs.abp.io/zh-Hans/abp/latest/CLI)

执行完后目录下将会出现项目解决方案，解决方案结构为：

- `Azusa.AbpBlog.Application`  
应用层，包含应用服务的接口实现。  
引用 `.Application.Contracts` 项目，因为接口在那里。  
引用.Domain项目，它需要使用领域对象。

- `Azusa.AbpBlog.Application.Contracts`  
应用层契约(Contract)，主要包含应用服务的抽象接口，以及应用层的数据传输对象DTO，它用于分离应用层的接口和实现. 这种方式可以将接口项目做为约定包共享给客户端。  
引用 `.Domain.Shared` 项目。

- `Azusa.AbpBlog.AuthServer`  
独立的身份验证服务器项目，该项目有独立的配置文件以及用户界面，前端项目调用该服务器进行用户认证和登录，获取到 Bearer Token 以后再访问API服务器。该项目使用 IdentityServer ，还提供了独立的用户界面 (ASP .NET Core MVC)。  
引用 `.Domain` 项目  
引用 `.EntityFrameworkCore` 项目

- `Azusa.AbpBlog.Blazor`  
前端项目，提供用户界面。  
引用 `.Application.Contracts` 项目，获取DTO  
引用 `.HttpApi.Client` 项目


- `Azusa.AbpBlog.DbMigrator`  
控制台应用程序，可以使用它创建数据库，应用数据库迁移，初始化种子数据。  
引用 `.Application.Contracts` 项目，因为它需要访问权限定义在初始化种子数据时为管理员用户赋予所有权限。  
引用 `.EntityFrameworkCore.DbMigrations` 项目，因为它要进行迁移。

- `Azusa.AbpBlog.Domain`  
领域层，它主要包含实体,聚合根,领域服务,值对象,仓储接口和解决方案的其他领域对象。  
引用 `.Domain.Shared` 项目。

- `Azusa.AbpBlog.Domain.Shared`  
公共项目，包含常量，枚举或其他对象。  
不依赖于其他项目，其他项目直接或间接地引用该项目。

- `Azusa.AbpBlog.EntityFrameworkCore`  
EFCore的项目，定义了`DbContext`类，并对领域层中声明的仓储类接口进行实现。  
因为EFCore已经实现了仓储模式，在这里重新封装一次其实没有好处，在应用层直接引用EFCore项目会更加简洁，不过既然要符合ABP框架的领域驱动设计，再次封装也无妨。

- `Azusa.AbpBlog.EntityFrameworkCore.DbMigrations`  
这个项目不会自动创建，之后的文章会提到，该项目专门用于整合DbContext类的迁移。  
引用 `.EntityFrameworkCore` 项目。

- `Azusa.AbpBlog.HttpApi`  
用于定义API控制器，不过通常ABP框架可以根据应用层自动创建API控制器。  
引用 `.Application.Contracts` ，因为它需要应用服务的接口(interface)。

- `Azusa.AbpBlog.HttpApi.Client`  
定义C#客户端代理使用解决方案的HTTP API项目。  
引用 `.Application.Contracts` 项目，因为它需要使用应用服务接口和DTO。  

- `Azusa.AbpBlog.HttpApi.Host`  
后端 Web API 启动项目，它有独立的配置文件。  
依赖于大部分项目。

> [项目结构文档](https://docs.abp.io/zh-Hans/abp/latest/Startup-Templates/Application#%E9%BB%98%E8%AE%A4%E7%BB%93%E6%9E%84)

---

## 运行解决方案
### 修改配置文件
修改以下项目中`appsettings.json`里的连接字符串。  
`Azusa.AbpBlog.AuthServer`的连接字符串，该数据库负责认证服务。  
`Azusa.AbpBlog.HttpApi.Host`的连接字符串，主要的数据库。  
`Azusa.AbpBlog.DbMigrator`的连接字符串，该项目用于生成数据库迁移。  
项目使用的是SQL Server Express数据库。  
``` json
"ConnectionStrings": {
    "Default": "Server=ALICIA\\SQLEXPRESS;User Id=Azusa;Password=12345678;Database=db_AbpBlog;Trusted_Connection=True;TrustServerCertificate=True"
},
```

分层项目使用 Redis 作为缓存服务器，首先需要安装Redis客户端。  

在`Azusa.AbpBlog.HttpApi.Host`项目  
以及`Azusa.AbpBlog.AuthServer`项目中的`appsettings.json`文件进行配置，  
默认情况下，ABP框架使用本机地址：
``` json
"Redis": {
"Configuration": "127.0.0.1"
},
```
***如果使用远程服务器，需要修改该配置。***
    
### 数据库迁移  

**初次迁移**：
> .dbMigator 应用程序在首次运行时自动创建初始迁移.
>
> 如果你使用的是 Visual Studio, 你可以跳到 运行 dbMigrator 部分。  
但是, 其他 IDE (例如 Rider) 在首次运行时可能会遇到问题, 因为它会添加初始迁移并编译项目。  
在这种情况下, 请在 .dbMigration 项目的文件夹中打开命令行终端, 然后运行以下命令:
>
>   ``` shell
>    dotnet run
>    ```
> 下次, 你可以像往常一样在 IDE 中运行它.


**执行迁移**  
运行Azusa.AbpBlog.DbMigrator项目，它会自动执行迁移。  
迁移执行成功后，会输出以下内容：  
![迁移成功](./%E6%89%B9%E6%B3%A8%202023-04-13%20231155.png)  

### 运行项目  
**运行`Azusa.AbpBlog.HttpApi.Host`项目**  
执行该项目会启动Web API项目，默认访问 [https://localhost:44320/swagger/index.html](https://localhost:44320/swagger/index.html) 查看Swagger UI。  
![运行效果](./%E6%89%B9%E6%B3%A8%202023-04-14%20000121.png)  

**运行`Azusa.AbpBlog.AuthServer`项目**  
启动用户认证服务，默认访问 [https://localhost:44326](https://localhost:44326) 查看用户认证页面。
![运行效果](./%E6%89%B9%E6%B3%A8%202023-04-14%20083427.png)

**运行`Azusa.AbpBlog.Blazor`项目**  
启动Blazor客户端，默认访问 [https://localhost:44373/](https://localhost:44373/) 查看客户端页面。
![运行效果](./%E6%89%B9%E6%B3%A8%202023-04-14%20000741.png)  

点击页面右上角的 *login* 按钮进行登录，该操作将会访问 `.AuthServer` 项目的 API。  
![登录](./%E6%89%B9%E6%B3%A8%202023-04-14%20085758.png)
此页面是 `AuthServer` 项目提供的，可以看到页面的URL并不是Blazor项目的：
![URL](./%E6%89%B9%E6%B3%A8%202023-04-14%20093028.png)  
默认的用户名和密码为`admin`和`1q2w3E*`。

### 异常和解决方案  
- 运行.AuthServer项目时抛出 `AbpException: Could not find the bundle file '/libs/abp/core/abp.css' for the bundle 'LeptonXLite.Global'!` 异常：  

    查看这篇文章可解决： https://stackoverflow.com/questions/70236519/invalidoperationexception-cannot-find-compilation-library-location-for-package
- 运行.AuthServer项目时抛出`AbpException:Could not find the bundle file '/libs/abp/core/abp.css' for the bundle 'LeptonXLite.Global'!`异常：  

    项目缺少 [Node.js](https://nodejs.org) 依赖，先检查Node.js是否正常安装并添加至环境变量，确认安装后，在解决方案目录使用打开命令提示符输入:  
    ``` shell
    abp install-libs
    ```
    他将自动调用 Node.js 下载项目依赖的 NPM 包，下载完毕后重新运行项目即可。

---  

至此，项目的创建和运行应当全部完成。