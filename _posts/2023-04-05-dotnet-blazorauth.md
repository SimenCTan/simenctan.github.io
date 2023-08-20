---
title: Blazor WebAssembly 认证
date: 2023-04-05 20:10:22 +0800
categories: [.NET]
tags: [blazor]     # TAG names should always be lowercase
mermaid: true
---
Blazor WebAssembly 支持通过Microsoft.AspNetCore.Components.WebAssembly.Authentication库使用OIDC对应用进行身份验证和授权，出于以下功能和安全原因选择了以JWT)为基础的基于令牌的身份验证而不是基于cookie的身份验证
1. 使用基于令牌的协议可以减小攻击面，因为并非所有请求中都会发送令牌
2. 服务器终结点不要求针对跨站点请求伪造 (CSRF) 进行保护，因为会显式发送令牌
3. 令牌的生命周期更短这限制了攻击时间窗口，还可随时撤销令牌
4. OAuth 和 OIDC 的令牌不依赖于用户代理行为以确保应用安全
5. 基于令牌的协议（例如 OAuth 和 OIDC）允许用同一组安全特征对托管和独立应用进行验证和授权

## IdentityServer认证Blazor wasm
使用命令dotnet new blazorwasm -au Individual -ho --name IdentityAuthen 将独立或托管的 Blazor WebAssembly应用配置为使用现有的外部Identity服务器进行授权认证。在IdentityAuthen.Server项目中会引用包,其中Identity是关于身份验证的包，EntityFrameworkCore是关于数据库访问的包，在IdentityAuthen.Server项目Program.cs文件中会配置认证授权的代码
```C#
// Add services to the container.
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite(connectionString));
builder.Services.AddDatabaseDeveloperPageExceptionFilter();

builder.Services.AddDefaultIdentity<ApplicationUser>(options => options.SignIn.RequireConfirmedAccount = true)
    .AddEntityFrameworkStores<ApplicationDbContext>();

builder.Services.AddIdentityServer()
    .AddApiAuthorization<ApplicationUser, ApplicationDbContext>();

builder.Services.AddAuthentication()
    .AddIdentityServerJwt();

builder.Services.AddControllersWithViews();
builder.Services.AddRazorPages();
```
AddApiAuthorization 帮助器方法针对 ASP.NET Core场景配置IdentityServer。 Identity Server 是一个功能强大且可扩展的框架，用于处理应用安全问题 AddIdentityServerJwt 帮助器方法将应用的策略方案配置为默认身份验证处理程序。 该策略配置为允许 Identity 处理路由到 Identity URL 空间 /Identity 中任何子路径的所有请求，此外次方法还可以

1. 向具有默认 API 范围的 IdentityServer 注册 API API 资源
2. 配置中间件以验证 IdentityServer为应用颁发的令牌
配置IdentityAuthen.Server项目的cors允许客户端的域名访问，并配置回调的url
```C#
builder.Services.AddIdentityServer()
	.AddApiAuthorization<ApplicationUser, ApplicationDbContext>(options => {
		// Clients
		var spaClient = ClientBuilder
				.SPA("IdentityAuthen.Client")
				.WithRedirectUri("https://localhost:7276/authentication/login-callback")
				.WithLogoutRedirectUri("https://localhost:7276/authentication/logged-out")
				.Build();
		spaClient.AllowedCorsOrigins = new[]
		{
				"https://localhost:7276"
		};

		options.Clients.Add(spaClient);
	});
  app.UseCors(cors => cors.WithOrigins("https://localhost:7276").AllowAnyMethod().AllowAnyHeader().AllowCredentials());
```
在IdentityAuthen.Client项目中会自动接收 Microsoft.AspNetCore.Components.WebAssembly.Authentication 包的包引用此包提供了一组基元，可帮助应用验证用户身份并获取令牌以调用受保护的API，在索引页 (wwwroot/index.html) 包含一个脚本，用于在 JavaScript 中定义 AuthenticationService，AuthenticationService 处理 OIDC 协议的低级别详细信息，应用从内部调用脚本中定义的方法以执行身份验证操作。
`<script src="_content/Microsoft.AspNetCore.Components.WebAssembly.Authentication/
    AuthenticationService.js"></script>`
在App.razor组件中CascadingAuthenticationState组件管理向应用的其余部分公开AuthenticationState；AuthorizeRouteView 组件确保当前用户有权访问给定页面或以其他方式呈现 RedirectToLogin 组件 启动IdentityAuthen.Server授权服务器和客户端IdentityAuthen.Client运行效果如下
![blazor-identityserver-authorize](/assets/img/blazor-identityserver-authorize.png)
#### 参考
[ASP.NET Core Blazor WebAssembly app with Identity Server](https://docs.microsoft.com/en-us/aspnet/core/blazor/security/webassembly/hosted-with-identity-server)
