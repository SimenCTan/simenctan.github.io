---
title: Blazor 模型绑定与校验
date: 2023-03-12 19:10:22 +0800
categories: [.NET]
tags: [blazor]     # TAG names should always be lowercase
mermaid: true
---
Blazor 框架支持 WebForm 并使用 EditForm 绑定到组件上进行模型验证。验证的过程则是：EditForm 基于分配的模型实例创建了EditContext，用作窗体中其他组件的EditForm。EditContext 跟踪有关编辑进程的元数据，其中包括已修改的字段和当前的验证消息；通过提供OnValidSubmit事件让其在提交含有效字段的窗体时运行处理窗体提交，OnInvalidSubmit 事件让其在提交含无效字段的窗体时运行，使用OnSubmit事件让其在不考虑窗体字段验证状态的情况下运行，通过调用事件处理程序方法中的 EditContext.Validate 来验证窗体如果 Validate返回true则窗体有效。
## EditForm模型绑定验证
定义待验证的模型Starship 包含了数据注释的多个属性
```csharp
using System.ComponentModel.DataAnnotations;
public class Starship
{
    [Required]
    [StringLength(16, ErrorMessage = "Identifier too long (16 character limit).")]
    public string? Identifier { get; set; }
    public string? Description { get; set; }

    [Required]
    public string? Classification { get; set; }
    [Range(1, 100000, ErrorMessage = "Accommodation invalid (1-100000).")]
    public int MaximumAccommodation { get; set; }

    [Required]
    [Range(typeof(bool), "true", "true",
			ErrorMessage = "This form disallows unapproved ships.")]
    public bool IsValidatedDesign { get; set; }

    [Required]
    public DateTime ProductionDate { get; set; }
}
```
通过以下内容接受和验证用户输入
```html
<EditForm Model="@starship" OnValidSubmit="@HandleValidSubmit">
<DataAnnotationsValidator></DataAnnotationsValidator>
<ValidationSummary></ValidationSummary>
<p>
<label>
Identifier: <InputText @bind-Value="starship.Identifier"></InputText>
</label>
</p>
<p>
<label>
Description (optional): <InputTextArea @bind-Value="starship.Description"></InputTextArea>
</label>
</p>
<p>
<label>
Primary Classification: <InputSelect @bind-Value="starship.Classification">
<option value="">Select classification ...</option>
<option value="Exploration">Exploration</option>
<option value="Diplomacy">Diplomacy</option>
<option value="Defense">Defense</option>
</InputSelect>
</label>
</p>
<p>
<label>
Maximum Accommodation:<InputNumber @bind-Value="starship.MaximumAccommodation"></InputNumber>
</label>
</p>
<p>
<label>
Engineering Approval:<InputCheckbox @bind-Value="starship.IsValidatedDesign"></InputCheckbox>
</label>
</p>
<p>
<label>
Production Date:<InputDate @bind-Value="starship.ProductionDate"></InputDate>
</label>
</p>
<button type="submit">Submit</button>
</EditForm>
@code {
private Starship starship = new() { ProductionDate = DateTime.UtcNow };
private void HandleValidSubmit()
{
Logger.LogInformation("HandleValidSubmit called");
// Process the valid form
}
```
其中DataAnnotationsValidator 组件将数据注释验证附加到级联 EditContext。 启用数据注释验证需要DataAnnotationsValidator组件。 若要使用不同于数据注释的验证系统，请用自定义实现替换 DataAnnotationsValidator组件 ValidationSummary 组件用于汇总所有验证消息, EditForm 基于分配的 Starship 实例创建 EditContext (Model="@starship") 并处理有效的窗体
![editorfrom](/assets/img/blazor-forms-validation-editorfrom.png)
## EditForm基本验证
在基本窗体验证场景中，EditForm 实例可以使用声明的 EditContext 和 ValidationMessageStore 来校验，EditContext组件的OnValidationRequested 事件处理程序执行自定义验证逻辑，处理程序的结果会更新 ValidationMessageStore 实例 在组件中增加代码
```html
<EditForm EditContext="editContext" OnValidSubmit="@HandleValidSubmit">
<label>
	Type 1:
	<InputCheckbox @bind-Value="holodeck.Type1" />
</label>
<label>
	Type 2:
	<InputCheckbox @bind-Value="holodeck.Type2" />
</label>
<button type="submit">Update</button>
<ValidationMessage For="() => holodeck.Options" />
</EditForm>
@code{
public class Holodeck
{
public bool Type1 { get; set; }
public bool Type2 { get; set; }
public bool Options => Type1 || Type2;
}

private EditContext? editContext;
private Holodeck holodeck = new();
private ValidationMessageStore? messageStore;
protected override void OnInitialized()
{
		editContext=new EditContext(holodeck);
		editContext.OnValidationRequested+=HandleValidationRequested;
		messageStore=new(editContext);
}
private void HandleValidationRequested(object? sender,ValidationRequestedEventArgs args)
{
	messageStore?.Clear();
	// Custom validation logic
	if (!holodeck.Options)
	{
			messageStore?.Add(() => holodeck.Options, "Select at least one.");
	}
}
private void HandleValidSubmit()
{
	Logger.LogInformation("HandleValidSubmit called: Processing the form");
	// Process the form
}
public void Dispose()
{
if (editContext is not null)
{
	editContext.OnValidationRequested -= HandleValidationRequested;
}
}
}
```
HandleValidationRequested 处理程序方法通过在验证窗体之前调用 ValidationMessageStore.Clear 来清除任何现有的验证消息
## 自定义验证 CSS 类属性
在校验失败时使用自定义的css来提示用户，在wwwroot/css/app.css (Blazor WebAssembly) 或 wwwroot/css/site.css (Blazor Server) css文件中添加自定的样式或则使用框架Bootstrap和tailwindcss定义的css； 创建一个从 FieldCssClassProvider 派生的类，用于检查字段验证消息，并应用相应的有效或无效样式。
```csharp
using System.Linq;
using Microsoft.AspNetCore.Components.Forms;

namespace BlazorAssmeblyNoHost.Options;
public class CustomFieldClassProvider : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext, in FieldIdentifier fieldIdentifier)
    {
        var isValid = !editContext.GetValidationMessages(fieldIdentifier).Any();
        return isValid ? "validField" : "invalidField";
    }
}
```
重写OnInitialized()方法设置自定义CSS属性提供器
```csharp
@code{
protected override void OnInitialized()
{
editContext=new EditContext(holodeck);
editContext.SetFieldCssClassProvider(new CustomFieldClassProvider());
editContext.OnValidationRequested+=HandleValidationRequested;
messageStore=new(editContext);
}
}
```
#### 参考
[Blazor forms and validation](https://docs.microsoft.com/en-us/aspnet/core/blazor/forms-validation)
