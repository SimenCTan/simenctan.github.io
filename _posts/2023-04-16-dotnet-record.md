---
title: csharp Record 类型
date: 2023-04-16 21:12:22 +0800
categories: [.NET]
tags: [csharp]     # TAG names should always be lowercase
mermaid: true
authors: [1,2]
---
record是一种引用类型，旨在简化和增强对不可变数据的建模和操作。C# 9引入了record type,csharp 10扩展了record增加了record struct，提供了一种更简洁、更强大的方式来处理数据。本文将深入探讨C#记录类型的背景、特性、record struct的特点和用法以及与传统类（class）的区别。

##  record的背景
在C#中，我们经常需要创建用于表示数据的类。这些类通常包含一组属性，用于描述对象的状态。然而，创建和使用这些类可能需要编写大量的样板代码，如属性、构造函数、相等性比较和哈希码等。为了简化这个过程，C#引入了记录类型。记录类型的背景可以追溯到函数式编程语言中的不可变数据结构。不可变数据结构具有许多优点，如线程安全、易于推理和测试、以及更好的性能。通过引入记录类型，C#使得不可变数据的建模和操作更加简单和直观。

## record的特性
###  不可变性
记录类型是不可变的，一旦创建，就不能修改其属性的值。这种不可变性有助于编写线程安全的代码，并简化了数据的使用和传递。
```csharp
Console.WriteLine("Hello, World!");
var person = new Person("You", "Test");
Console.WriteLine(person);

public record Person(string FirstName,string LastName);
```
###  自动生成成员
记录类型自动生成了一些有用的成员，如相等性比较、哈希码和ToString方法。这些成员基于记录类型的属性值进行计算，使得比较和打印记录类型对象变得非常方便。
```csharp
var person = new Person("You", "Test");
var personOther = new Person("You", "Test");
Console.WriteLine(person.Equals(personOther));
Console.WriteLine(person.GetHashCode());
Console.WriteLine(person.ToString());

public record Person(string FirstName,string LastName);
```

###  模式匹配和解构
记录类型与C#的模式匹配功能相结合，可以实现更强大的模式匹配和解构操作。模式匹配可以用于比较记录类型的属性值、提取属性值以及进行条件分支等操作。
```csharp
if (person is Person { FirstName: "You" }) // 使用属性模式匹配
{
    Console.WriteLine("Hello,Yod");
}
var (f, l) = person; // 使用解构操作
Console.WriteLine($"{f} {l}");

public record Person(string FirstName,string LastName);
```

### 继承
当一个记录类型被继承时，子类会继承父类的属性和自动生成的成员，包括相等性比较、哈希码和ToString方法。子类还可以添加自己的属性和成员。
```csharp
Person student = new Student(12, "You", "Test");
Console.WriteLine(student.Equals(person));

public record Person(string FirstName,string LastName);

public record Student(int Number,string FirstName, string LastName) : Person(FirstName,LastName);
```

## record struct的特点和用法
csharp 10引入了记录结构（record struct），它是记录类型的一种变体，作为值类型使用。记录结构具有与记录类型相似的特点，但由于是值类型，它具有一些额外的优势,记录结构是值类型，存储在栈上，因此比引用类型的记录类型更轻量级。这使得记录结构在内存使用和性能方面具有优势。记录结构也是不可变的，一旦创建，就不能修改其属性的值。它们同样自动生成了相等性比较、哈希码和ToString方法，使得使用和操作记录结构对象变得简单和方便。
```csharp
var point1 = new Point(1, 2);
var point2 = new Point(1, 2);
Console.WriteLine(point1.Equals(point2));
Console.WriteLine(point1.GetHashCode());
Console.WriteLine(point1.ToString());

public record struct Point(int X,int Y);
```
记录结构可以像记录类型一样使用属性、模式匹配和解构操作，而且由于是值类型，可以更高效地使用和传递。然而，需要注意的是，记录结构适用于较小的数据结构，不适合存储大量的数据。

##  record与class的区别
尽管记录类型与类在某些方面相似，但它们也有一些区别  
1. 记录类型是不可变的，而类可以是可变的。记录类型的属性是只读的，一旦创建，就不能修改其属性的值（某些情况下也可以修改record属性的值）。
2. 记录类型自动生成了一些成员，如相等性比较、哈希码和ToString方法。而在类中，需要手动实现这些成员。
3. 记录类型是基于值的相等性比较，而类是基于引用的相等性比较。这意味着记录类型的相等性比较是基于属性值，而不是对象引用。
4. 记录类型可以被继承，子类会继承父类的属性和自动生成的成员。这使得记录类型可以实现代码的重用和扩展。而类也可以被继承，但需要手动定义继承关系和实现相关成员。
