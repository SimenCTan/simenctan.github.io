---
title: 掌握异步编程中的ConfigureAwait
date: 2024-02-07 16:30:22 +0800
categories: [.NET]
tags: [infrastructure]     # TAG names should always be lowercase
mermaid: true
authors: [1,2]
---

### 理解ConfigureAwait
`ConfigureAwait`是在异步编程中用于控制异步操作继续运行的上下文的方法。通过使用`ConfigureAwait(true)`或`ConfigureAwait(false)`，开发人员可以指定异步操作的继续运行是否需要在原始上下文中执行，从而影响性能和避免潜在的死锁问题。
```csharp
// 使用ConfigureAwait(true)示例
public async Task<int> ExampleMethod()
{
    // 异步操作
    await Task.Delay(100).ConfigureAwait(true);

    // 继续运行在原始上下文中
    return 1;
}

// 使用ConfigureAwait(false)示例
public async Task<int> ExampleMethod()
{
    // 异步操作
    await Task.Delay(100).ConfigureAwait(false);

    // 继续运行在不需要同步上下文的情况下
    return 1;
}
```
在适当的情况下使用ConfigureAwait(false)可以确保异步操作在不需要同步上下文的情况下运行，从而提高效率。

### 使用ConfigureAwait(false)的好处
使用ConfigureAwait(false)可以在不需要同步上下文的情况下提高性能，特别是在以下情况下
- 避免UI线程阻塞：在UI应用程序中，使用ConfigureAwait(false)可以确保异步操作不会阻塞UI线程，从而保持应用程序的响应性。
- 提高并发性能：通过在不需要同步上下文的情况下执行异步操作，可以更好地利用线程池资源，提高并发性能。
  - 将回调排队而不是直接调用它是有成本的，因为这涉及到额外的工作（通常还涉及额外的分配），而且还意味着我们无法使用我们通常希望在运行时中使用的某些优化如果 await 后的代码实际上不需要在原始上下文中运行，使用 ConfigureAwait(false) 可以避免所有这些成本：不必要地排队、可以利用所有它能够调动的优化避免不必要的线程静态访问。
- 避免死锁。考虑一个使用 await 在某个网络下载的结果上的库方法。您调用此方法并且同步阻塞等待它完成，例如通过使用返回的 Task 对象的 .Wait() 或 .Result 或 .GetAwaiter().GetResult()。现在考虑当您调用它时，当前的同步上下文是一个限制可以在其上运行的操作数量为 1 的上下文这是一个只有一个可以使用的线程的上下文，例如 UI 线程隐式地限制了并发数。因此，您在该线程上调用该方法，然后阻塞等待操作完成。该操作启动网络下载并等待它。由于默认情况下等待 Task 会捕获当前的同步上下文，因此当网络下载完成时，它会将将调用剩余操作的回调排队回到同步上下文中。但是，可以处理排队的回调的唯一线程当前被您的代码阻塞等待操作完成。而且该操作直到回调被处理才会完成。死锁！即使上下文不限制并发数为 1，但当资源以任何方式受限时，也会发生这种情况。想象一下相同的情况，只是使用限制为 4 而且不仅仅是对操作进行一次调用，我们将 4 个调用排队到该上下文中，每个调用都会进行调用并阻塞等待它完成。现在，我们仍然在等待异步方法完成时阻塞了所有资源，而唯一能够让这些异步方法完成的是它们的回调能够被已经完全占用的这个上下文处理。再次，死锁！如果库方法使用了 ConfigureAwait(false)，它将不会将回调排队回到原始上下文，从而避免死锁情况。
```csharp
// 示例：使用ConfigureAwait(false)防止死锁并提高响应性
public async Task<int> ExampleMethod()
{
    // 异步操作1
    await Task.Run(async () =>
    {
        // 异步操作2
        await Task.Delay(100).ConfigureAwait(false);
    }).ConfigureAwait(false);

    // 异步操作3
    await Task.Delay(200).ConfigureAwait(false);

    return 1;
}
```
在上面的示例中，通过在异步操作1和异步操作3中使用ConfigureAwait(false)，可以确保这些操作在不需要同步上下文的情况下执行，从而避免潜在的死锁问题并提高应用程序的响应性。
### ConfigureAwait(false)的最佳实践
应用程序级代码通常使用`ConfigureAwait(true)`此行为是默认配置；通用库代码通常使用`ConfigureAwait(false)`
- 在编写应用程序时，通常希望使用默认行为（这就是为什么它是默认行为）。如果应用模型/环境（例如 Windows Forms、WPF、ASP.NET Core 等）发布了自定义的同步上下文，那几乎肯定有一个非常好的理由：它提供了一个让关心同步上下文的代码与应用模型/环境适当交互的方式。因此，无论应用模型是否实际发布了同步上下文，您都应该在 Windows Forms 应用程序中编写事件处理程序，在 xunit 中编写单元测试，在 ASP.NET MVC 控制器中编写代码，如果存在该同步上下文，您都应该使用该同步上下文。这意味着默认/ConfigureAwait(true)。您简单地使用 await，如果存在原始上下文，则关于回调/继续被发布回到原始上下文的正确事情就会发生。这导致了一般性指导原则：如果您正在编写应用程序级代码，请勿使用ConfigureAwait(false)。
```csharp
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime");
    downloadBtn.Content = text;
}
```
设置 downloadBtn.Content = text 需要在原始上下文中完成。如果代码违反了这一准则，并且在不应该使用时使用了ConfigureAwait(false)：
```csharp
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime").ConfigureAwait(false); // bug
    downloadBtn.Content = text;
}
```
将对于依赖于HttpContext.Current使用ConfigureAwait(false)，然后尝试使用 HttpContext.Current 可能会导致问题。

- 相比之下通用库是“通用目的”的一部分，部分原因是它们不关心它们被使用的环境。您可以从 Web 应用程序、客户端应用程序或测试中使用它们，这并不重要，因为库代码对可能在其中使用的应用程序模型是不可知的。因为它是不可知的，所以它也不会执行任何需要以特定方式与应用程序模型交互的操作，例如它不会访问 UI 控件，因为通用库对 UI 控件一无所知。由于我们不需要在任何特定环境中运行代码，我们可以通过使用ConfigureAwait(false) 避免强制将继续/回调返回到原始上下文，并获得它带来的性能和可靠性好处。这导致了一般性指导原则：如果您正在编写通用库代码，请使用ConfigureAwait(false)。这就是为什么，例如，您会看到 .NET Core 运行时库中的每个（或几乎每个）await 在每个 await 上都使用ConfigureAwait(false)。

与所有指导原则一样，当然也会有例外情况，有些地方不适用。例如，在通用库中的较大的例外情况之一（或至少需要考虑的类别）是当这些库具有需要调用的委托的 API。在这种情况下，库的调用者正在传递可能由库调用的应用程序级代码，这实际上使得库的“通用目的”假设无效。例如，考虑 LINQ 的 Where 方法的异步版本，例如 public static async IAsyncEnumerable<T> WhereAsync(this IAsyncEnumerable<T> source, Func<T, bool> predicate)。这里的 predicate 是否需要在调用者的原始同步上下文中被调用？这取决于 WhereAsync 的实现来决定，这也是它可能选择不使用ConfigureAwait(false) 的原因。
即使在这些特殊情况下，一般性指导仍然有效，并且是一个非常好的起点：如果您正在编写通用库/应用程序模型不可知代码，请使用ConfigureAwait(false)，否则不要使用。

### ConfigureAwait FAQ
关于ConfigureAwait常见使用场景问题总结
#### ConfigureAwait(false) 是否保证回调不会在原始上下文中运行？
ConfigureAwait(false) 不保证await 之后的代码不会在原始上下文中运行。它确保回调不会排队回到原始上下文。如果您等待一个在等待时已经完成的任务，则无论是否使用了ConfigureAwait(false)，紧接着的代码都将继续在当前上下文中的当前线程上执行。

#### 只在第一个await使用ConfigureAwait(false)而不在其余await中使用
通常不会这么做，如果await task.ConfigureAwait(false)涉及一个在等待时已经完成的任务（这实际上非常常见），那么ConfigureAwait(false)将毫无意义，因为线程继续执行此后方法中的代码并且仍然处于与之前相同的环境中。除非你确定第一个 await 总是异步完成，并且被等待的任务会在没有自定义 SynchronizationContext 或 TaskScheduler 的环境中调用其回调。一个明显的例外是像.NET运行时库中的CryptoStream，它希望确保其潜在的计算密集型代码不会作为调用者同步调用的一部分运行，因此它使用自定义的awaiter来确保第一个 await 之后的所有代码都在线程池线程上运行。然而，即使在这种情况下，你会注意到下一个 await 仍然使用 ConfigureAwait(false)；从技术上讲这并不是必要的，但这样做会使代码审查变得更容易，因为否则每次查看这段代码时都不需要分析为什么省略了 ConfigureAwait(false)

#### 可以使用Task.Run来避免使用ConfigureAwait(false)吗？
可以，如果你在 `Task.Run` 中写入以下代码：
```csharp
Task.Run(async delegate
{
    await SomethingAsync(); // 不会看到原始上下文
});
```
那么在 `SomethingAsync()` 调用上使用 `ConfigureAwait(false)` 将是一个空操作，因为传递给 `Task.Run` 的委托将在一个线程池线程上执行，堆栈中没有更高级的用户代码，因此 `SynchronizationContext.Current` 将返回 null。此外，`Task.Run` 隐式使用 `TaskScheduler.Default`，这意味着在委托内部查询 `TaskScheduler.Current` 也会返回 `Default`。这意味着无论是否使用了 `ConfigureAwait(false)`，`await` 的行为都将相同。它也不会对此 lambda 内部的代码做出任何保证。如果你有以下代码：
```csharp
Task.Run(async delegate
{
    SynchronizationContext.SetSynchronizationContext(new SomeCoolSyncCtx());
    await SomethingAsync(); // 将会针对 SomeCoolSyncCtx
});
```
那么 `SomethingAsync` 内部的代码实际上将看到 `SynchronizationContext.Current` 作为那个 `SomeCoolSyncCtx` 实例，并且这个 `await` 和 `SomethingAsync` 内部的任何未配置的 `await` 都会回到它。因此，要使用这种方法，你需要了解你正在排队的所有代码可能会做什么，以及它的行为是否可能会阻碍你的行为。
这种方法还需要创建/排队一个额外的任务对象，这可能对你的应用程序或库是否敏感于性能有所影响。

### 参考
- [configureawait faq](https://devblogs.microsoft.com/dotnet/configureawait-faq/#comments)
- [ConfigureAwait](https://github.com/Fody/ConfigureAwait)
