---
title: Introduction to Rx.NET
weight: 2
---

{{< callout type="info" >}}
本书原作者为 [Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/) 和 [Lee Campbell](https://leecampbell.com/) 。原书发布在 [IntroToRx.NET](https://www.introtorx.com/) 网站上，由 Hugo HU 翻译，最后校对日期为 2023.12.11 ，如果发现译文有错误或译文落后于源文本请通过本仓库 [issues](https://github.com/librehugohu/librehugohu.github.io/issues) 反馈或通过 [E-Mail](mailto:librehugohu@outlook.com) 联系我改正。
{{< /callout >}}

## Rx简介

响应式编程不是什么新概念。任何形式的UI开发都必须包含响应事件的代码。像 [Smalltalk](https://en.wikipedia.org/wiki/Smalltalk)、[Delphi](https://en.wikipedia.org/wiki/Delphi_(software)) 和 .NET 语言都有流行的响应式或事件驱动编程模型。诸如 [CEP (Complex Event Processing)](https://en.wikipedia.org/wiki/Complex_event_processing) 和 [CQRS (Command Query Responsibility Segregation)](https://en.wikipedia.org/wiki/Command_Query_Responsibility_Segregation) 这样的架构模型将事件作为其组成的基本部分。在任何必须处理正在发生的事情的程序中，响应式编程都是一个有用的概念。

> 在任何必须处理**正在发生的事情**的程序中，响应式编程都是一个有用的概念。

事件驱动模型允许在不破坏封装或使用昂贵的轮询技术的前提下调用代码。有许多常见的方法可以实现这一点，包括[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)、[直接在语言内部公开的事件（例如C#）](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/events/)或通过委托注册的其他形式的回调。Rx（Reactive Extensions 的缩写，“响应式扩展”，以下统称 ”Rx“）利用 LINQ 扩展了回调隐喻（callback metaphor），从而可以查询事件序列和管理并发性。

.NET 运行时库包含 IObservable<T> 和 IObserver<T> 接口，它们代表了十多年来响应式编程的核心概念。.NET 的响应式扩展 (Rx.NET) 实际上是这些接口的实现库。 Rx.NET 首次出现于 2010 年，之后 Rx 库也可用于其他语言，并且这种编程方式在 JavaScript 中变得尤其流行。

本书将通过 C# 介绍 Rx。这些概念是通用的，因此其他 .NET 语言（例如 VB.NET 和 F#）的使用者将能够提取这些概念并将其翻译为他们的特定语言。

Rx.NET 只是一个库，最初由 Microsoft 创建，但现在是一个完全通过社区努力支持的开源项目。（Rx 的现任首席维护者 [Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/) 也是本书最新版本的作者，实际上也是这句话的作者。）

如果您以前从未使用过 Rx，它将改变您设计和构建软件的方式。它为计算中的一个基本重要概念提供了经过深思熟虑的抽象：事件序列。这些事件序列与列表或数组一样重要，但在 Rx 之前，库或语言几乎没有直接支持，而且那里的支持往往相当临时（ad hoc），并且建立在薄弱的理论基础上。 Rx 改变了这一点。微软的这一发明能够被一些传统上对微软并不特别友好的开发者社区全心全意地采用，足以证明其基础设计的质量。

本书旨在教您：

* 关于 Rx 定义的类型
* 关于 Rx 提供的扩展方法以及如何使用它们
* 如何管理事件源的订阅
* 如何在编码之前可视化数据“序列”并简述你的解决方案
* 如何使并发对你有利并避免常见的陷阱
* 如何组合、聚合和转换流
* 如何测试你的 Rx 代码
* 使用 Rx 时的一些常见最佳实践

学习 Rx 的最好方法就是使用它。阅读本书中的理论只会帮助您熟悉 Rx，但要完全理解它，您应该用它来构建东西。因此，我们热烈鼓励您基于本书中的示例进行构建。

## 致谢

首先，我（Ian Griffiths）应该明确指出，这个修订版建立在原作者李·坎贝尔（Lee Campbell）的优秀作品之上。我很感激他慷慨地允许 Rx.NET 项目使用他的内容，从而使这个新版本得以诞生。

我还要感谢那些使这本书面世的人们。感谢 [endjin](https://www.introtorx.com/chapters/endjin.com) 的所有人，尤其是 [Howard van Rooijen](https://endjin.com/who-we-are/our-people/howard-van-rooijen/) 和 [Matthew Adams](https://endjin.com/who-we-are/our-people/matthew-adams/)，他们不仅为本书的更新提供资金，而且还为 Rx.NET 本身的持续开发提供资金。 （也感谢您雇用我！）。还要感谢 [Felix Corke](https://www.linkedin.com/in/blackspike/) 在本书网络版的设计元素方面所做的工作。除了作者 [Lee Campbell](https://leecampbell.com/) 之外，对本书第一版至关重要的还有：James Miles、Matt Barrett、[John Marks](http://johnhmarks.wordpress.com/)、Duncan Mole、Cathal Golden、Keith Woods、Ray Booysen、Olivier DeHeurles、[Matt Davey](http://mdavey.wordpress.com/)、[Joe Albahari](http://www.albahari.com/) 和 Gregory Andrien。特别感谢微软团队的辛勤工作，为我们带来了 Rx；[Jeffrey Van Gogh](https://www.linkedin.com/in/jeffrey-van-gogh-145673/), [Wes Dyer](https://www.linkedin.com/in/wesdyer/), [Erik Meijer](https://en.wikipedia.org/wiki/Erik_Meijer_%28computer_scientist%29) & [Bart De Smet](https://www.linkedin.com/in/bartdesmet/)。还要感谢那些在 Rx.NET 不再受到 Microsoft 直接支持并成为基于社区的开源项目后继续致力于 Rx.NET 工作的人们。很多人都参与其中，在这里列出每个贡献者是不切实际的，但我想特别感谢 Bart De Smet（再次强调，因为他在微软内部从事其他工作很久之后还在继续从事开源 Rx 的工作）以及 Claire Novotny、Daniel Weber、David Karnok、Brendan Forster、Ani Betts 和 Chris Pulman。我们还感谢 Richard Lander 和 .NET 基金会帮助身在 [endjin](https://endjin.com/) 的我们成为 Rx.NET 项目的新管理员，使其能够继续蓬勃发展。

如果您对有关 Rx 起源的更多信息感兴趣，您可能会从 [https://reaqtive.net/](https://reaqtive.net/) 站点上的 “A Little History of Reaqtor” 中获得启发。

本书所针对的版本是 System.Reactive 版本6.0。本书的源代码可以在 [https://github.com/dotnet/reactive/tree/main/Rx.NET/Documentation/IntroToRx](https://github.com/dotnet/reactive/tree/main/Rx.NET/Documentation/IntroToRx) 找到。如果您在本书中发现任何错误或其他问题，请在 [https://github.com/dotnet/reactive/](https://github.com/dotnet/reactive/) 上发布问题。如果您开始认真使用 Rx.NET，您可能会发现 [Reactive X slack](https://www.introtorx.com/chapters/reactivex.slack.com) 是一个有用的资源。

那么，启动 Visual Studio，让我们开始吧。
