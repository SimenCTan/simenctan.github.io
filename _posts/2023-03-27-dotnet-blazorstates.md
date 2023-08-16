---
title: Blazor 状态管理
date: 2023-03-27 22:10:22 +0800
categories: [.NET, C#]
tags: [blazor]     # TAG names should always be lowercase
mermaid: true
---
Blazor Server是有状态的应用框架大多数情况下应用保持与服务器的连接,用户的状态保留在线路中的服务器内存中.在Blazor WebAssembly应用中创建的用户状态会保存在浏览器的内存中,那么在什么情况下保持用户状态,主要在以下情况下保持用户状态
1. 呈现的 UI 中组件实例的层次结构及其最新的呈现输出
2. 组件实例中的字段和属性的值
3. 依赖关系注入 (DI) 服务实例中保留的数据

### Blazor应用状态管理
Blazor Server是有状态的应用框架大多数情况下应用保持与服务器的连接，用户的状态保留在线路中的服务器内存中，还可以通过JavaScript互操作调用在浏览器的内存集的 JavaScript 变量中找到用户状态。在Blazor Server中如果用户遇到暂时的网络连接丢失问题，Blazor 会尝试将用户重新连接到具有其原始状态的原始线路，在以下情况下可能原始线路会断开。
1. 超时后或在服务器面临内存压力时，服务器必须释放断开连接的线路
2. 在负载均衡的多服务器部署环境中，不再需要单个服务器处理整个请求量时，它可能会失败或被自动删除。在用户尝试重新连接时，用户的原始服务器处理请求可能会变得不可用
3. 用户可能会关闭并重新打开其浏览器或重载页面，这会删除浏览器内存中保留的所有状态

### 服务器存储状态
对于跨多个用户和设备的永久数据持久性，应用可以使用服务器端存储用户状态：可以在关系数据库存储、Redis中存储键值对以及Blob 存储

### URL中存储状态
对于表示导航状态的暂时性数据，例如已查看实体的ID，分页网格中的当前页码，通过URL保存当前所在页和当前查看实体的ID，以便在用户刷新浏览器之后依然能获取到这些暂时性的数据
### 浏览器存储状态
用户正在主动创建的暂时性数据，通用存储位置是浏览器的localStorage和sessionStorage。
1. localStorage 的应用范围限定为浏览器的窗口，如果用户重载页面或关闭并重新打开浏览器则状态保持不变。如果用户打开多个浏览器选项卡，则状态跨选项卡共享。 数据保留在 localStorage 中，直到被显式清除为止
2. sessionStorage 的应用范围限定为浏览器的选项卡。如果用户重载该选项卡，则状态保持不变。 如果用户关闭该选项卡或该浏览器，则状态丢失。 如果用户打开多个浏览器选项卡，则每个选项卡都有自己独立的数据版本 在_Imports.razor 中引入命名空间@using Microsoft.AspNetCore.Components.Server.ProtectedBrowserStorage，在Counter.razor组件中使用ProtectedSessionStorage来存储本次对话需要保存的数据，以便当用户刷新浏览器的时候依然能读取到之前的数据
```C#
@page "/counter"
@inject ProtectedSessionStorage ProtectedSessionStorage
<PageTitle>Counter</PageTitle>
<h1>Counter</h1>
<p role="status">Current count: @currentCount</p>
<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>
@code {
	private int currentCount = 0;
	protected override async Task OnParametersSetAsync()
	{
		var result = await ProtectedSessionStorage.GetAsync<int>("count");
		currentCount=result.Success?result.Value:0;
	}
	private async Task IncrementCount()
	{
		currentCount++;
		await ProtectedSessionStorage.SetAsync("count",currentCount);
	}
}
```
![session-storage](/assets/img/blazor-session-storage.png)
>在使用浏览器存储时的注意事项
1.与使用服务器端数据库类似，加载和保存数据都是异步的
2.与服务器端数据库不同，在预呈现期间，存储不可用，因为在预呈现阶段，请求的页面在浏览器中不存在
3.保留状态的位置对于 Blazor Server 应用，持久存储几千字节的数据是合理的。 超出几千字节后，你就须考虑性能影响，因为数据是跨网络加载和保存的
4.用户可以查看或篡改数据。 ASP.NET Core 数据保护可以降低风险。 例如，ASP.NET Core 受保护的浏览器存储使用 ASP.NET Core 数据保护

### 内存中状态容器服务
嵌套组件通常使用ASP.NET Core数据绑定中所述的链式绑定来绑定数据，嵌套组件和非嵌套组件可使用已注册的内存中状态容器来共享对数据的访问，自定义状态容器类可以使用可分配的 Action，来向应用不同部分中的组件通知状态更改，定义StateContainer类。
```C#
namespace BlazorServerApp.Services;
public class StateContainer
{
	private string? savedString;
	public string Property
	{
		get => savedString ?? string.Empty;
		set
		{
				savedString = value;
				NotifyStateChanged();
		}
	}
	public event Action? OnChange;
	private void NotifyStateChanged() => OnChange?.Invoke();
}
```
注入状态容器服务services.AddScoped<StateContainer>(); 增加Nested.razor组件
```html
@implements IDisposable
@inject StateContainer StateContainer
<h2>Nested component</h2>
<p>Nested component Property: <b>@StateContainer.Property</b></p>
<p>
	<button @onclick="ChangePropertyValue">
			Change the Property from the Nested component
	</button>
</p>
@code{
	protected override void OnInitialized()
	{
		StateContainer.OnChange+=StateHasChanged;
	}
	private void ChangePropertyValue()
	{
		StateContainer.Property =
		$"New value set in the Nested component: {DateTime.Now}";
	}

	public void Dispose()
	{
		StateContainer.OnChange -= StateHasChanged;
	}
}
```
在Counter.razor组件中增加对Nested组件的引用
```html
<h1>State Container Example component</h1>
<p>State Container component Property: <b>@StateContainer.Property</b></p>
<p>
    <button @onclick="ChangePropertyValue">
        Change the Property from the State Container Example component
    </button>
</p>
<Nested />
@code{
...
public void Dispose()
{
	StateContainer.OnChange-=StateHasChanged;
}
protected override void OnInitialized()
{
	StateContainer.OnChange += StateHasChanged;
}
private void ChangePropertyValue()
{
	StateContainer.Property = "New value set in the State " +
			$"Container Example component: {DateTime.Now}";
}
}
```
![](/assets/img/blazor-state-container.png)
组件实现IDisposable并且OnChange委托在Dispose方法中取消订阅这些方法是在释放组件时由框架调用的。
#### 参考
[ASP.NET Core Blazor state management](https://docs.microsoft.com/en-us/aspnet/core/blazor/state-management)
