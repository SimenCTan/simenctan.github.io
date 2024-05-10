---
title: HttpClient的常用场景及问题分析
date: 2023-05-10 22:30:22 +0800
categories: [.NET]
tags: [c#]     # TAG names should always be lowercase
mermaid: true
authors: [1,2]
---
在一个.NET应用一般都需要通过HTTP调用一个外部API,在.NET中发送HTTP请求的简单方式是使用HttpClient,尤其是在支持JSON负载和响应的方法中，但在使用HttpClient很容易被误用。

### 每次请求创建一个新实例
```csharp
public class GitHubService
{
  private readonly GitHubSettings _settings;
  public GitHubService(IOptions<GitHubSettings> settings)
  {
      _settings = settings.Value;
  }

  public async Task<GitHubUser?> GetUserAsync(string username)
  {
      var client = new HttpClient();

      client.DefaultRequestHeaders.Add("Authorization", _settings.GitHubToken);
      client.DefaultRequestHeaders.Add("User-Agent", _settings.UserAgent);
      client.BaseAddress = new Uri("https://api.github.com");

      GitHubUser? user = await client
          .GetFromJsonAsync<GitHubUser>($"users/{username}");

      return user;
  }
}
```
上面的代码可能会出什么问题呢？HttpClient实例应该是长期存在的，并在应用程序的生命周期中重复使用,如果服务器负载过高而你的应用程序不断创建新的连接，可能会导致可用端口耗尽。当再次创建实例时可能引发程序异常。

### 使用IHttpClientFactory智能创建HttpClient
使用IHttpClientFactory来创建HttpClient实例，而不是自己管理HttpClient的生命周期,只需调用CreateClient方法，然后使用返回的HttpClient实例来发送你的HTTP请求。HttpClient的昂贵部分是实际的消息处理器`HttpMessageHandler`，每个`HttpMessageHandler`都有一个可以重用的内部HTTP连接池,IHttpClientFactory会缓存HttpMessageHandler，并在创建新的HttpClient实例时重用它,这里需要注意的一点是，由IHttpClientFactory创建的HttpClient实例应该是短期存在的,使用IHttpClientFactory优化代码
```csharp
public class GitHubService
{
  private readonly GitHubSettings _settings;
  private readonly IHttpClientFactory _factory;
  public GitHubService(
    IOptions<GitHubSettings> settings,
    IHttpClientFactory factory)
  {
      _settings = settings.Value;
      _factory = factory;
  }

  public async Task<GitHubUser?> GetUserAsync(string username)
  {
      var client = _factory.CreateClient();

      client.DefaultRequestHeaders.Add("Authorization", _settings.GitHubToken);
      client.DefaultRequestHeaders.Add("User-Agent", _settings.UserAgent);
      client.BaseAddress = new Uri("https://api.github.com");

      GitHubUser? user = await client
          .GetFromJsonAsync<GitHubUser>($"users/{username}");

      return user;
  }
}
```
### 使用命名客户端
使用IHttpClientFactory将解决大部分手动创建HttpClient的问题。然而我们仍然需要每次从CreateClient方法获取新的HttpClient时配置默认的请求参数。这时候可以使用命名客户端来减少代码重复。你可以通过调用AddHttpClient方法并传入所需的名称来配置一个命名的客户端。AddHttpClient接受一个委托，你可以使用它来配置HttpClient实例的默认参数。
```csharp
services.AddHttpClient("github", (serviceProvider, client) =>
{
  var settings = serviceProvider
  .GetRequiredService<IOptions<GitHubSettings>>().Value;
  client.DefaultRequestHeaders.Add("Authorization", settings.GitHubToken);
  client.DefaultRequestHeaders.Add("User-Agent", settings.UserAgent);
  client.BaseAddress = new Uri("https://api.github.com");
});
```
主要的区别是你现在必须通过传递客户端的名称来获取客户端,但是使用HttpClient看起来更简单:
```csharp
public class GitHubService
{
  private readonly IHttpClientFactory _factory;
  public GitHubService(IHttpClientFactory factory)
  {
      _factory = factory;
  }

  public async Task<GitHubUser?> GetUserAsync(string username)
  {
      var client = _factory.CreateClient("github");

      GitHubUser? user = await client
          .GetFromJsonAsync<GitHubUser>($"users/{username}");

      return user;
  }
}
```
### 配置一个类型化的客户端
使用命名客户端的缺点是每次都需要通过传入一个名称来解析一个HttpClient,有一种更好的方法可以通过配置一个类型化的客户端来实现相同的行为。你可以通过调用AddClient<TClient>方法并配置将消费HttpClient的服务来实现这一点。在底层，这仍然是使用一个命名的客户端，其中的名称与类型名称相同,这也会以瞬态生命周期注册GitHubService。
```csharp
services.AddHttpClient<GitHubService>((serviceProvider, client) =>
{
  var settings = serviceProvider
  .GetRequiredService<IOptions<GitHubSettings>>().Value;
  client.DefaultRequestHeaders.Add("Authorization", settings.GitHubToken);
  client.DefaultRequestHeaders.Add("User-Agent", settings.UserAgent);

  client.BaseAddress = new Uri("https://api.github.com");
});
```
在GitHubService中你注入并使用已应用所有配置的类型化HttpClient实例,不再需要处理IHttpClientFactory和手动创建HttpClient实例。

### 在单例服务中使用类型化的客户端
如果你在一个单例服务中注入一个类型化的客户端，你可能会遇到一个问题。由于类型化的客户端是瞬态的，将它注入到一个单例服务中会导致它在单例服务的生命周期中被缓存,这将阻止类型化的客户端对DNS变化作出反应,推荐的方法是使用SocketsHttpHandler作为主要的处理器，并配置PooledConnectionLifetime。由于SocketsHttpHandler将处理连接池，你可以通过将HandlerLifetime设置为Timeout.InfiniteTimeSpan来在IHttpClientFactory级别禁用回收
```csharp
services.AddHttpClient<GitHubService>((serviceProvider, client) =>
{
  var settings = serviceProvider
  .GetRequiredService<IOptions<GitHubSettings>>().Value;
  client.DefaultRequestHeaders.Add("Authorization", settings.GitHubToken);
  client.DefaultRequestHeaders.Add("User-Agent", settings.UserAgent);
  client.BaseAddress = new Uri("https://api.github.com");
}.ConfigurePrimaryHttpMessageHandler(() =>
{
  return new SocketsHttpHandler()
  {
    PooledConnectionLifetime = TimeSpan.FromMinutes(15)
  };
})
.SetHandlerLifetime(Timeout.InfiniteTimeSpan);
```

### 参考
- [httpclient-guidelines](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)
- [Use IHttpClientFactory to implement resilient HTTP requests](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)
- [the-right-way-to-use-httpclient-in-dotnet](https://www.milanjovanovic.tech/blog/the-right-way-to-use-httpclient-in-dotnet)
