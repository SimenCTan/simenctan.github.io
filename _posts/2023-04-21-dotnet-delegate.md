---
title: C# Delegate
date: 2023-04-16 21:12:22 +0800
categories: [.NET]
tags: [C#]     # TAG names should always be lowercase
mermaid: true
authors: [1,2]
---
委托是一项强大的功能，通过将方法封装为对象并进行传递，提供了代码的灵活性和可扩展性，使开发人员能过轻松实现松耦合、代码重用和异步执行，从而编写更高效和易于维护的代码。
## 委托静态方法
C#委托可用于调用静态方法和实例方法。在处理静态方法时，委托提供了一种封装方法并将其作为对象传递的方式。这样可以实现解耦并增强代码的可重用性。
在`Delegates.cs`中定义委托 `StaticDelegate`
```csharp
public delegate void StaticDelegate(string message);
```
在`Program.cs`中委托静态方法
```csharp
StaticDelegate staticDelegate = Console.WriteLine;
staticDelegate("Hello static delegate");
```
在上面的代码片段中，我们定义了一个委托StaticDelegate，它指向Console.WriteLine方法。然后我们创建委托的实例并使用所需的消息调用它。

## 委托实例方法
委托还可用于封装和调用类的实例方法这使得应用程序可以实现松耦合并将关注点分离,在示例中我们定义了一个委托`InstanceDelegate`和类`Animal`，把委托`InstanceDelegate`指向类`Animal`的实例方法。在这里我们创建了`Animal`的实例并将PrintLiveTime分配给委托最后我们调用委托委托又调用了MPrintLiveTime方法
```csharp
public delegate void InstanceDelegate(int years);
public class Animal
{
    public virtual void PrintLiveTime(int years)
    {
        Console.WriteLine($"live {years} years old");
    }
}

var animal = new Animal();
StaticDelegate staticDelegate = Console.WriteLine;
InstanceDelegate instanceDelegate = animal.PrintLiveTime;
staticDelegate("Hello static delegate");
instanceDelegate(10);
```

## 多播委托
C#委托支持多播，意味着它们可以引用多个方法并按顺序调用它们。这个特性在需要对事件或操作进行多个方法调用时非常有用
```csharp
var animal = new Animal();
StaticDelegate staticDelegate = Console.WriteLine;
staticDelegate += PrintMultiDelegate;
InstanceDelegate instanceDelegate = animal.PrintLiveTime;
staticDelegate("Hello static delegate");
instanceDelegate(10);

static void PrintMultiDelegate(string message)
{
    Console.WriteLine($"Multi Delegate {message}");
}
```
在示例中通过使用+=运算符，我们将多个方法添加到委托中当调用委托时，两个方法按顺序执行。

## 委托的协变和逆变
C#支持委托类型的协变和逆变，允许在委托之间进行更灵活的赋值。协变允许将返回派生类型的委托分配给返回基类型的委托，而逆变允许将接受基类型参数的委托分配给接受派生类型参数的委托。
```csharp
public delegate Animal AnimalFactory();
public delegate void AnimalConsumer(Animal animal);
AnimalFactory factory = () => new Dog();
AnimalConsumer consumer = (Animal animal) => Console.WriteLine(animal.GetType().Name);

Animal animalft = factory();
consumer(animalft);
```
在示例中我们定义了两个委托：AnimalFactory返回一个Animal对象，AnimalConsumer接受一个Animal对象作为参数。我们将一个创建Dog对象的lambda表达式分配给AnimalFactory委托展示了协变。然后我们将一个接受Animal对象的lambda表达式分配给AnimalConsumer委托展示了逆变。

## 内置委托Func、Action和Predicate
C#提供了三个内置委托类型：Func、Action和Predicate。这些委托类型消除了为常见场景定义自定义委托的需要。
### Func
Func是一个泛型委托，可以指向具有最多16个输入参数和一个返回值的方法。`Func<int, int, int> sum = (a, b) => a + b;`
### Action
Action是一个泛型委托，可以指向具有最多16个输入参数和无返回值的方法。`Action<int, int> print = (a, b) => Console.WriteLine(a + b);`
### Predicate
Predicate是一个泛型委托，表示接受一个参数并返回布尔值的方法。
`Predicate<int> isEven = n => n % 2 == 0;`

## 委托的应用
### 回调
委托的一种常见用途是创建回调方法。在这种情况下，委托作为参数传递给方法，然后在该方法中调用以指示进度或完成。
```csharp
LongRunningMethond(i => Console.WriteLine($"progess {i}% complete"));
static void LongRunningMethond(Action<int> reportProgress)
{
    for (var i=1; i <= 10; i++)
    {
        reportProgress(i * 10);
        Task.Delay(100);
    }
}
```
在示例中将lambda 表达式与委托一起使用。Lambda表达式是一种编写匿名方法的简洁方法，LongRunningMethod通过调用委托来报告其进度

### 事件处理
委托通常与事件一起使用，以允许调用方法来响应某些操作。
```csharp
public class EventPublisher
{
    public event EventHandler? EventOccurred;

    public void DoSomething()
    {
        // 执行某些操作
        // 触发事件
        OnEventOccurred("Event occurred!");
    }

    protected virtual void OnEventOccurred(string message)
    {
        // 检查是否有订阅者，并调用事件处理程序
        EventOccurred?.Invoke(message);
    }
}

public class EventSubscribe
{
    public void HandleEvent(string message)
    {
        Console.WriteLine("Event handled: " + message);
    }
}

var publisher = new EventPublisher();
var subscriber = new EventSubscribe();

// 订阅事件
publisher.EventOccurred += subscriber.HandleEvent;

// 执行操作，触发事件
publisher.DoSomething();
```
在上面的示例中，我们定义了一个委托类型`EventHandler`，它接受一个字符串参数。然后，我们创建了一个`EventPublisher`类，其中声明了一个事件`EventOccurred`，其类型为`EventHandle`r委托。在`DoSomething`方法中，我们执行某些操作并调用`OnEventOccurred`方法来触发事件。在`OnEventOccurred`方法中，我们检查是否有订阅者，并通过调用委托的Invoke方法来触发事件。我们创建了一个`EventSubscriber`类，其中定义了一个`HandleEvent`方法来处理事件。在`Program.cs`中，我们创建了一个EventPublisher实例和一个EventSubscriber实例。然后，我们使用+=运算符将EventSubscriber的HandleEvent方法订阅到EventPublisher的事件上。最后，我们调用DoSomething方法来执行操作并触发事件。当事件触发时，EventSubscriber的HandleEvent方法将被调用，并打印出事件的消息。

