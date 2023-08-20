---
title: Blazor 文件上传
date: 2023-02-01 12:30:22 +0800
categories: [.NET]
tags: [blazor]     # TAG names should always be lowercase
mermaid: true
---
使用 InputFile组件将浏览器文件数据读入 .NET 代码，在用户选择文件后发生 OnChange (change) 事件时， InputFile 组件执行 LoadFiles 方法参数 InputFileChangeEventArgs 提供对所选文件列表和每个文件的详细信息的访问
```csharp
<InputFile OnChange="@LoadFiles" multiple/>
@code {
  private List<IBrowserFile> BrowserFiles = new();
  private int maxAllowFiles=3;
  private long maxFileSize = 1024 * 1024 * 5;
  private void LoadFiles(InputFileChangeEventArgs e)
  {
      try
      {
          BrowserFiles.AddRange(e.GetMultipleFiles(maxAllowFiles));
      }
      catch (InvalidOperationException ex)
      {
          _logger.LogError($"only allow {maxAllowFiles} files upload; message{ex.Message}");
      }
      Console.WriteLine($"{BrowserFiles.Count} files upload");
      BrowserFiles.Clear();
  }
}
```
调用 IBrowserFile.OpenReadStream，并从返回的流中读取文件内容。OpenReadStream 强制采用其 Stream 的最大大小（以字节为单位）。 读取一个或多个大于 512,000 字节 (500 KB) 的文件会引发异常。 此限制可防止开发人员意外地将大型文件读入到内存中。 如果需要，可以使用 OpenReadStream 上的 maxAllowedSize 参数指定更大的大小，如果超过这个大小就报出错误
```csharp
private void LoadFiles(InputFileChangeEventArgs e)
{
    try
    {
        BrowserFiles.AddRange(e.GetMultipleFiles(maxAllowFiles));
    }
    catch (InvalidOperationException ex)
    {
        _logger.LogError($"only allow {maxAllowFiles} files upload; message{ex.Message}");
    }
    if (BrowserFiles.Count > 0)
    {
        foreach (var file in BrowserFiles)
        {
            try
            {
                var fileContent = file.OpenReadStream(maxAllowedSize: maxFileSize);
            }
            catch (Exception ex)
            {
                _logger.LogError($"{file.Name} not uploaded (Err: 6): {ex.Message}");
            }
        }
    }
    Console.WriteLine($"{BrowserFiles.Count} files upload");
```
## 上传文件到AzureBlob
Azure Storage Blobs 暂不支持wasm所以无法从blazor wasm应用中直接上传文件到azure blob上得通过相应api上传文件。通过命令 dotnet new webapi —name FileUpload 创建webapi项目引用Azure blob，在Program.cs文件中注入BlobContainerClient
```csharp
// Add services to the container.
builder.Services.Configure<AzureStorageOption>(builder.Configuration.GetSection("AzureStorage"));
AzureStorageOption AzureStorageOption = new();
builder.Configuration.GetSection("AzureStorage").Bind(AzureStorageOption);
builder.Services.AddScoped(_ =>
{
    var blobContainerClient = new BlobContainerClient(AzureStorageOption.StorageConnectionString, AzureStorageOption.ContainerName);
    blobContainerClient.CreateIfNotExists();
    return blobContainerClient;
});
```
新增Controller 在FileController.cs 文件中增加uploadfile接口
```csharp
[ApiController]
[Route("api/[Controller]/[action]")]
public class FileController : Controller
{
    private readonly BlobContainerClient blobContainerClient;
    private readonly ILogger<FileController> logger;
    private readonly AzureStorageOption storageOption;

    public FileController(BlobContainerClient blobContainerClient,
        ILogger<FileController> logger,
        IOptionsMonitor<AzureStorageOption> optionsMonitor)
    {
        this.blobContainerClient = blobContainerClient;
        this.logger = logger;
        storageOption = optionsMonitor.CurrentValue;
    }

    [HttpPost]
    [ActionName("UploadFile")]
    public async Task<IActionResult> UploadFile()
    {
        var formCollection = await Request.ReadFormAsync();
        var file = formCollection.Files.First();
        if (file.Length <= 0) return BadRequest();
        if (!storageOption.Format.Contains(Path.GetExtension(file.FileName).TrimStart('.'),StringComparison.OrdinalIgnoreCase))
        {
            return BadRequest();
        }

        try
        {
            var blobFile = blobContainerClient.GetBlobClient(file.FileName);
            await blobFile.DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots);
            using var fileStream = file.OpenReadStream();
            await blobFile.UploadAsync(fileStream, new BlobHttpHeaders { ContentType = file.ContentType });
            return Ok(blobFile.Uri.ToString());
        }
        catch (Exception ex)
        {
            logger.LogError(ex.Message);
            return StatusCode(500, ex.Message);
        }
    }
}
```
在webapi Program.cs 中配置cors策略允许不同域的应用程序可以调用api
```csharp
builder.Services.AddCors(policy => {
    policy.AddPolicy("CorsPolicy",
                 opt => opt.SetIsOriginAllowed(s => true)
                .AllowAnyMethod()
                .AllowAnyHeader());
});
```
在blazor wasm应用中注入的HttpClient设置BaseAddress
`builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri("https://localhost:7124/api/")
});`
blazor wasm FileOperation.razor 组件中允许用户从客户端上传文件,在用户界面中显示由客户端提供的不可信/不安全的文件名，不受信任/不安全的文件名由Razor自动进行HTML编码，以便在用户界面中安全显示。
```csharp
@code {
    private List<IBrowserFile> BrowserFiles = new();
    private int maxAllowFiles=3;
    private long maxFileSize = 1024 * 1024 * 5;
    public string? imgUrl { get; set; }
    private async Task LoadFiles(InputFileChangeEventArgs e)
    {
        try
        {
            BrowserFiles.AddRange(e.GetMultipleFiles(maxAllowFiles));
        }
        catch (InvalidOperationException ex)
        {
            _logger.LogError($"only allow {maxAllowFiles} files upload; message{ex.Message}");
        }
        if (BrowserFiles.Count > 0)
        {
            var file = BrowserFiles.First();
            using var fileContent = file.OpenReadStream(maxFileSize);
            MultipartFormDataContent httpContent = new();
            httpContent.Headers.ContentDisposition = new ContentDispositionHeaderValue("form-data");
            httpContent.Add(new StreamContent(fileContent), "file", file.Name);
            var response = await HttpClient.PostAsync("File/UploadFile", httpContent);
            imgUrl = await response.Content.ReadAsStringAsync();
            //foreach (var file in BrowserFiles)
            //{
            //    try
            //    {
            //        var fileContent = file.OpenReadStream(maxAllowedSize: maxFileSize);
            //    }
            //    catch (Exception ex)
            //    {
            //        _logger.LogError($"{file.Name} not uploaded (Err: 6): {ex.Message}");
            //    }
            //}
        }
        Console.WriteLine($"{BrowserFiles.Count} files upload");
        BrowserFiles.Clear();
        await Task.CompletedTahe
```
## 文件显示
选择文件上传到azure blob上返回文件的url路径，在img标签中显示图片内容
```html
@if(imgUrl!=null)
{
    <div>
        <img src="@imgUrl" class="image-preview" />
    </div>
}
```
![upload-file-2-azure](/assets/img/upload-file-2-azure.png)

#### 参考
[blazor-file-upload](https://learn.microsoft.com/en-us/aspnet/core/blazor/file-uploads)
