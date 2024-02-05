---
title: 1.1 为什么选择 Rx .NET？
weight: 1
---

Rx 是一个用于处理事件流的 .NET 库。你为什么需要它？

## 为什么需要 Rx？

因为用户想要及时的信息。如果您正在等待快递送达，您可能更愿意看到快递员的实时位置信息而不是像“预计 2 小时内送货上门”这样粗略的信息；金融程序依赖于持续更新的数据流；人们希望手机和计算机能够及时向我们提供各种重要通知。如果没有实时信息，某些应用程序根本无法运行。比如在线协作工具和多人游戏绝对依赖于数据的快速分发和传递。

简而言之，我们的系统需要在感兴趣的事情发生时做出反应。

实时信息流是计算机系统中普遍存在的基本元素。尽管如此，它们在编程语言中往往是二等公民。大多数语言通过数组之类的东西支持数据序列，它假设数据位于内存中，可供我们的代码在空闲时读取。如果您的应用程序需要处理事件，数组可能更适合保存历史数据而不是表示程序运行时发生的事件。尽管流式数据（streamed data）在计算领域是一个相当古老的概念，但它往往很笨重，通过与我们的编程语言的类型系统集成很差的 API 来实现抽象。

这很糟糕。实时数据对于很多应用程序至关重要。它应该像列表、字典和集合一样易于使用。

[.NET 响应式扩展](https://github.com/dotnet/reactive)（简称 Rx.NET 或 Rx，以 [System.Reactive NuGet 包](https://www.nuget.org/packages/System.Reactive/) 的形式提供）将实时数据源提升为一等公民。Rx 不需要任何特殊的编程语言支持。它利用 .NET 的类型系统来表示数据流，因此 .NET 语言（例如 C#、F# 和 VB.NET）都可以像使用集合类型一样自然地使用它。

（顺便说一下语法：虽然短语“Reactive Extensions”是复数，但当我们将其简化为 Rx.NET 或 Rx 时，我们将其视为单数名词。这是不一致的，但说“Rx are...”听起来很奇怪。）

例如，C# 提供集成查询功能，我们可以使用它来查找列表中满足某些条件的所有条目。如果我们有一些 `List<Trade> trades` 变量，我们可以这样写：

```C#
var bigTrades =
    from trade in trades
    where trade.Volume > 1_000_000;
```

借助 Rx，我们可以对实时数据使用完全相同的代码。此时 `trades` 变量是 `IObservable<Trade>` 类型。`IObservable<T>` 是 Rx 中的基本抽象。它本质上是 `IEnumerable<T>` 的实时版本。在这种情况下 `bigTrades` 也将是 `IObservable<Trade>` 类型，它是一个代表 `trade.Volume` 超过一百万的交易的实时数据源。最重要的是它可以即时报告每一笔此类型的交易——这就是我们所说的“实时”数据源。

Rx 是一个功能强大的高效开发工具。它使开发人员能够使用所有 .NET 开发人员熟悉的语言功能来处理实时事件流。它采用声明式方法（a declarative approach），通常能让我们用更少的代码更优雅地表达复杂的行为（相比于不使用 Rx 的情况）。

Rx 构建于 LINQ（语言集成查询）之上，因此我们可以使用上面展示的 LINQ 风格查询语法（您也可以用某些 .NET 开发人员偏爱的显式函数调用方式）。 LINQ 在 .NET 中被广泛用于数据访问（比如在 Entity Framework Core 中）和处理内存中的集合（使用 LINQ to Objects），这意味着经验丰富的 .NET 开发人员在使用 Rx 时会有一种熟悉感。至关重要的是，LINQ 是一种高度可组合的设计：您可以按照您喜欢的任何组合将运算符连接在一起，以简单的方式表达潜在的复杂处理。这种可组合性源于其设计的数学基础，您可以根据需要进一步了解，但这并不是使用 LINQ 的先决条件。对其背后的数学原理不感兴趣的开发人员可以直接享受 LINQ 提供的便利：LINQ 提供商（如 Rx）提供了一系列构建模块，这些模块可以通过无穷无尽的组合方式连接在一起，而且一切都能正常工作。

LINQ 在处理大量数据方面成绩斐然。微软在一些内部系统的实现中广泛使用了它，包括支持数千万活跃用户的服务。

## 什么时候适合使用 Rx ？

Rx 专为处理事件序列而设计，这意味着它有一些专用领域。接下来的部分将介绍其中一些最合适的场景、不太合适但仍值得考虑的情况以及可以使用 Rx 但替代方案可能更好的情况。

### 与 Rx 完美契合的使用场景

Rx 非常适合表示来自代码外部且需要代码对其做出响应的事件，例如：

* 集成事件（integration events），例如来自消息总线的广播、来自 WebSockets API 的推送事件、通过 MQTT 或其他低延迟中间件（例如 [Azure 事件网格](https://azure.microsoft.com/en-gb/products/event-grid/)、[Azure 事件中心](https://azure.microsoft.com/en-gb/products/event-hubs/)和 [Azure 服务总线](https://azure.microsoft.com/en-gb/products/service-bus/)）接收的消息、通用的事件表示形式（例如 [cloudevents](https://cloudevents.io/)）。

* 来自监控设备的遥测，例如自来水厂在公共基础设施中安装的流量传感器、宽带提供商在其网络设备中集成的监控和诊断功能

* 来自移动系统的位置数据，例如来自船舶的 [AIS](https://github.com/ais-dotnet/) 消息或汽车遥测

* 操作系统事件，例如文件系统活动或 WMI 事件

* 道路交通信息，例如事故通知或汽车平均速度变化通知

* 与[复杂事件处理 (Complex Event Processing, CEP)](https://en.wikipedia.org/wiki/Complex_event_processing) 引擎集成

* UI 事件，例如鼠标移动或按钮单击

Rx 也适合对域事件（domain event）进行建模。这些域事件可能是上述某些事件导致的结果，但经过处理后，会产生更直接代表应用概念的事件。它们可能包括：

* 域对象的属性或状态更改，例如“已更新的订单状态”或“已接受的注册”

* 对域对象集合的更改，例如“已创建的新注册”

事件还可以代表从传入事件（或在发生一段时间后才进行分析的历史数据）中得出的见解（insight），例如：

* 宽带用户可能在不知情的情况下成为 DDoS 攻击的参与者

* 两艘远洋船只进行了一种通常与非法活动相关的移动模式（例如，在远离陆地的海面上长时间并排航行，这段时间长到足以转移船上的货物或人员）

* 数控铣床 MFZH12 的 4 号轴轴承出现磨损迹象，磨损率明显高于标称轮廓

* 如果用户想准时到达半个城镇外的会议地点，应用程序会根据当前的交通状况建议他们应该在接下来的 10 分钟内离开

这三组示例展示了应用程序在处理事件时如何逐步增加信息的价值。我们从原始事件（raw event）开始，然后对其进行强化以生成特定于域的事件（domain-specific event），然后执行分析以生成应用程序用户真正关心的通知【译注：也就是“见解（insight）”】。处理事件的每个阶段都会增加新事件的质量、减少新事件的数量。如果我们将第一层级中的原始事件（raw event）直接呈现给用户，他们可能会被大量的消息淹没而无法发现真正重要的事件。但是，如果我们仅在处理检测到重要信息时向它们发送通知，这将使它们能够更高效、更准确地工作，因为我们极大地提高了信息的信噪比。

[System.Reactive 库](https://www.nuget.org/packages/System.Reactive)提供了用于构建此类增值流程（value-adding process）的工具，在该流程中，我们驯服大量原始事件源以产生高价值、实时且可操作的见解。它提供了一套运算符，使我们的代码能够声明式地表达这种处理，正如您将在后续章节中看到的那样。

Rx 还非常适合引入和管理并发以实现任务卸载（offloading）功能。也就是说，并发执行一组给定的工作，这样检测到事件的线程就可以和处理该事件的线程分开来。该技术的一个非常流行的用途是维护响应式 UI。（UI 事件处理已经成为 Rx 的一种流行用途——无论是在 .NET 中，还是在 [RxJS](https://rxjs.dev/) 中，RxJS 起源于 Rx.NET 的一个分支——人们很容易认为这就是它的用途。但它在 UI 领域的成功不应该让人们忽视它更广泛的适用性。）

如果您有一个正在尝试对实时事件建模的现有 `IEnumerable<T>`，您应该考虑使用 Rx。虽然 `IEnumerable<T>` 可以对运动中的数据进行建模（通过使用像 `yield return` 这样的惰性求值），但存在一个问题：如果使用集合的代码运行到了需要下一项（item）的时间点（比如 `foreach` 循环刚刚完成迭代）但还没有可用的项，则 `IEnumerable<T>` 只能阻塞 `MoveNext` 中的调用线程，直到数据可用，这可能会导致某些应用程序中的可伸缩性问题（scalability problem）。即使在线程阻塞是可以接受的情况下（或者如果您使用较新的 `IAsyncEnumerable<T>`，它可以利用 C# 的 `await foreach` 功能来避免在这些情况下阻塞线程），`IEnumerable<T>` 和 `IAsyncEnumerable<T>` 仍然是表示实时信息源的误导类型。这些接口代表“拉取式（pull）”编程模型：代码请求序列中的下一项。对于按照自己的时间表自然生成信息的信息源进行建模，Rx 是更自然的选择。

### 可能适合 Rx

Rx可以用来表示异步操作。.NET 的 `Task` 或 `Task<T>` 有效地表示单个事件，而 `IObservable<T>` 可表示一系列事件。（`Task<int>` 和 `IObservable<int>` 之间的关系类似于 `int` 和 `IEnumerable<int>` 之间的关系。）

这意味着有些场景既可以使用 `Task` 和 `async` 关键字也可以通过 Rx 来处理。如果在处理过程中的任何时候您需要处理多个值和单个值，则 Rx 可以同时胜任这两项工作；但 `Task` 不能很好地处理多个项目。您可以使用 `Task<IEnumerable<int>>` ，它使您能够等待（`await`）一个集合，如果可以在一个步骤中集齐集合中的所有项目，那就没问题了。这样做的限制是，一旦 `Task<T>` 产生了 `IEnumerable<int>` 结果，您的等待（`await`）就完成了，并且您将回到对 `IEnumerable<int>` 的非异步迭代。如果无法在单个步骤中获取所有数据（例如 `IEnumerable<int>` 表示来自 API 的数据，每次批量获取 100 个项作为结果），则其 `MoveNext` 每次需要等待时都必须阻塞线程。

在 Rx 诞生的最初 5 年里，它可以说是表示不一定会立刻获得可用项的集合的最佳方式。然而，.NET Core 3.0 和 C# 8 中引入的 [IAsyncEnumerable<T>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) 提供了一种在 `async` / `await` 环境中处理序列的方法（.NET Framework 和 .NET Standard 2.0 平台可通过 [Microsoft.Bcl.AsyncInterfaces NuGet 包](https://www.nuget.org/packages/Microsoft.Bcl.AsyncInterfaces/)使用相关功能）。因此，现在是否使用 Rx 往往取决于“拉取式（pull）”模型（以 `foreach` 或 `await foreach` 为代表）还是“推送式（push）”模型（代码提供回调，当项可用时由事件源调用）更适合正在建模的概念。

自 Rx 首次出现以来 .NET 添加的另一个相关功能是[通道（channels）](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels)。它们允许源生成对象并让消费者处理它们，因此粗看与 Rx 有点像。然而，Rx 的一个明显特征是它支持与大量运算符组合在一起，而通道中没有直接等效的功能。另一方面，通道提供了更多的选择来适应生产和消费率的变化。

之前，我提到过任务卸载（offloading）：使用 Rx 将工作推送到其他线程上。尽管此技术可以使 Rx 引入和管理并发性以扩展或执行并行计算，但其他专用框架（例如 [TPL（任务并行库，Task Parallel Library）数据流](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library) 或 [PLINQ](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/introduction-to-plinq)）更适合执行并行计算密集型工作。但是，TPL 数据流 通过其 [AsObserver](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.dataflow.dataflowblock.asobserver) 和 [AsObservable](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.dataflow.dataflowblock.asobservable) 扩展方法提供了与 Rx 的一些集成。因此，通常使用 Rx 将 TPL 数据流与应用程序的其余部分集成在一起。

### 与 Rx 配合不佳

Rx 的 `IObservable<T>` 不能替代 `IEnumerable<T>` 或 `IAsyncEnumerable<T>` 。不能强行让基于“拉取式（pull）”工作的数据流以“推送式（push）”运行。

此外，在某些情况下，Rx 编程模型的简单性可能会对您不利。例如，一些消息队列技术（例如 MSMQ）根据定义是顺序的，因此可能看起来很适合 Rx。但它们通常是因为支持业务处理才被选用。而 Rx 没有任何直接的方式来表达业务语义，因此在需要此类消息队列技术的场景中，您最好直接使用相关技术的 API。 （从另一方面说，像 [Reaqtor](https://reaqtive.net/) 这样的框架为 Rx 增加了耐用性和持久性的支持，因此您可以使用它将这些类型的队列系统与 Rx 集成。）

通过选择最适合工作的工具，您的代码应该更容易维护，它可能会提供更好的性能，并且您可能会获得更好的支持。

## Rx 实践

您可以快速开始一个简单的 Rx 示例。如果您安装了 .NET SDK，则可以在命令行运行以下命令：

```C#
mkdir TryRx
cd TryRx
dotnet new console
dotnet add package System.Reactive
```

或者，如果安装了 Visual Studio，请创建一个新的 .NET 控制台项目，然后使用 NuGet 包管理器添加对 System.Reactive 的引用。

此代码创建一个可观察源（ticks），每秒生成一个事件。该代码还将一个处理程序传递给该源，该处理程序将每个事件的消息写入控制台：

```C#
using System.Reactive.Linq;

IObservable<long> ticks = Observable.Timer(
    dueTime: TimeSpan.Zero,
    period: TimeSpan.FromSeconds(1));

ticks.Subscribe(
    tick => Console.WriteLine($"Tick {tick}"));

Console.ReadLine();
```

如果这看起来不是很令人激动，那是因为它是一个能创建的最简单示例，并且从本质上讲，Rx 有一个非常简单的编程模型。力量来自于组合——我们可以使用 System.Reactive 库中的构建块来描述我们将原始的低级事件转变为高价值见解的处理过程。但要做到这一点，我们必须首先了解 [Rx 的关键类型 IObservable<T> 和 IObserver<T>](../1.2-key-types/)。
