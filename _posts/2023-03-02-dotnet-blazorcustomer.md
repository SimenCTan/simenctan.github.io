---
title: Blazor 自定义组件
date: 2023-03-02 09:30:22 +0800
categories: [.NET, C#]
tags: [blazor]     # TAG names should always be lowercase
mermaid: true
---
组件是在razor文件中C#和HTML标记的组合，组件文件名使用 Pascal 风格命名，在编译应用时HTML标记和C#呈现逻辑转换为组件类，生成的类的名称与文件名匹配。组件的@code块中主要处理
- 属性和字段初始化表达式。
- 由父组件和路由参数传递的自变量的参数值。
- 用于用户事件处理、生命周期事件和自定义组件逻辑的方法。
HTML标记和C#代码位于同一个文件中，也可以使用带有分部类的代码隐藏文件从C#代码中拆分HTML和Razor标记，例如CounterPartialClass.razor和CounterPartialClass.razor.cs
```html
@page "/counter-partial-class"

<PageTitle>partial class component</PageTitle>
<h1>test</h1>
<p role="status">Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>
```
```C#
namespace BlazorNativeFile.Pages;
public partial class CounterPartialClass
{
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```
### 组件参数
#### 通过父组件传递参数
组件参数将数据传递给组件，使用组件类中包含 特性的公共 C# 属性进行定义,应将组件参数声明为自动属性，这意味着它们不应在其 或 set 访问器中包含自定义逻辑,如果子组件属性的 set 访问器包含导致父组件重新呈现的逻辑，则会导致一个无限的呈现循环,定义组件ProductionCard.razor
```C#
<div>
    <img class="rounded" src="@imageUrl" alt="@imageAlt"/>
    <div class="mt-2">
        <div>
            <div class="text-xs text-slate-600 uppercase font-bold tracking-wider">@eyeBrow</div>
            <div class="font-bold text-slate-700 leading-snug">
                <a href="@textUrl" class="hover:underline">@textUrl</a>
            </div>
            <div class="mt-2 text-sm text-slate-600">@Price</div>
        </div>
    </div>
</div>

@code{
    [Parameter]
    public string imageUrl{get;set;}=default!;

    [Parameter]
    public string imageAlt{get;set;}=default!;

    [Parameter]
    public string? eyeBrow {get;set;}

    [Parameter]
    public string? textUrl {get;set;}

    [Parameter]
    public int Price {get;set;}
}
```
#### 通过路由传递参数
blazor组件不仅可以通过父组件传递参数也可以通过路由传递参数，在@page指令的路由模板中指定路由参数，路由器使用路由参数来填充相应的组件参数。
```c#
@page "/counter/{routeParameter?}"
...
@code {
	private int currentCount = 0;
	[Parameter]
	public string? RouteParameter{get;set;}
	private void IncrementCount()
	{
			Console.WriteLine(RouteParameter);
			currentCount++;
	}
}
```
#### 传递泛型参数
如果blazor组件参数是泛型类型参数可以用@typeparam 指令声明
```C#
@typeparam TEntity where TEntity:struct
...
@code{
	 [Parameter]
   public IEnumerable<TEntity>? Entities {get;set;}
}
```
在父组件中使用带有泛型参数的组件
```html
<ProductionCard
    imageUrl="https://images.unsplash.com/photo-1452784444945-3f422708fe5e?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=512&q=80"
    imageAlt="test"
    eyeBrow="Private Villa"
    textUrl="Relaxing All-Inclusive Resort in Cancun"
    Price=299
    Entities="@(new List<int>{1,2,3,4})"
    TEntity="int">
    </ProductionCard>
```
#### 通过CascadingValue组件传递参数
祖先组件使用 Blazor 框架的 CascadingValue 组件提供级联值，该组件包装组件层次结构的子树，并向其子树中的所有组件提供单个值，与常规组件参数类似，当级联值改变时，接受级联参数的组件会重新呈现。在组件MainLayout.razor中添加
```html
<CascadingValue Value="@BtnClass">
		<article class="content px-4">
				@Body
		</article>
</CascadingValue>
@code{
private string BtnClass {get;init;}="btn-success";
}
在组件CounterPartialClass.razor中添加

@code{
  [CascadingParameter]
  protected string BtnClass {get;set;}
}
```
### 组件数据绑定
组件使用 @bindRazor 指令特性提供与字段、属性或 Razor 表达式值的数据绑定功能。
```html
<p>
  <label>
    <code>yyyy-MM-dd</code>format:
    <input @bind="startDate" @bind:format="yyyy-MM-dd" />
  </label>
</p>
```
### 组件事件处理
blazor组件使用 @on=""Razor 语法在 Razor 组件标记中指定委托事件处理程序
- 占位符是（例如 click）
- 占位符是 C# 委托事件处理程序 对于事件处理支持返回 Task 的异步委托事件处理程序，委托事件处理程序会自动触发 UI 呈现，因此无需手动调用 StateHasChanged，记录异常。
```html
<p>
  <label>
      New title
      <input @bind="newHeading" />
  </label>
	<button @onclick="UpdateHeading">
			Update heading
	</button>
</p>
@code{
private async Task UpdateHeading()
{
		currentHeading = $"{newHeading}!!!";
		await Task.Delay(2000);
}
}
```
### 组件的生命周期
Razor组件处理一组同步和异步生命周期方法中的blazor组件生命周期事件，可以替代生命周期方法以在组件初始化和呈现期间对组件执行其他操作。 
>组件在生命周期中从呈现到释放各会发生什么事件
1. 创建组件的实例，执行属性注入，运行 SetParametersAsync
2. 调用 OnInitialized如果返回不完整的 Task，则将等待 Task，然后重新呈现组件
3. 调用 OnParametersSet。 如果返回不完整的 Task，则将等待 Task，然后重新呈现组件
4. 呈现所有同步工作和完整的 Task
![lifecycle-event](/assets/img/lifecycle-event.png)
>在组件内处理事件的生命周期
1. 运行事件处理程序
2. 如果返回不完整的 Task，则将等待 Task，然后重新呈现组件
3. 呈现所有同步工作和完整的 Task
![lifecycle-onevent](/assets/img/lifecycle-onevent.png)

>组件ui呈现的生命周期
1. 避免对组件进行进一步的呈现操作在第一次呈现后ShouldRender为false
2. 生成呈现树差异并呈现组件
3. 等待 DOM 更新
4. 调用 OnAfterRender
![lifecycle-ui](/assets/img/lifecycle-ui.png)

#### 参考
[ASP.NET Core Razor components](https://learn.microsoft.com/en-us/aspnet/core/blazor/components)
