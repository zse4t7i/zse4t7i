---
title: 1.3 创建可观察序列
weight: 3
---

在上一章中，我们了解了两个基本的 Rx 接口，`IObservable<T>` 和 `IObserver<T>`。我们还学习了如何通过直接实现 `IObserver<T>` 接口或使用 `System.Reactive` 包提供的实现来接收事件。在本章中，我们将学习如何创建 `IObservable<T>` 源来表示应用程序中使用的源事件。

我们先来了解一下如何直接实现 `IObservable<T>` 接口（这种做法在实践中不太常见），之后再学习如何使用 `System.Reactive` 提供的实现。

## 一个非常基本的 **IObservable<T>** 实现

下面是一个可以生成数字序列的 `IObservable<int>` 实现：

```C# {linenos=table,hl_lines=[],linenostart=1,filename="MySequenceOfNumbers.cs"}
public class MySequenceOfNumbers : IObservable<int>
{
    public IDisposable Subscribe(IObserver<int> observer)
    {
        observer.OnNext(1);
        observer.OnNext(2);
        observer.OnNext(3);
        observer.OnCompleted();
        return System.Reactive.Disposables.Disposable.Empty;
        // 返回一个释放时啥也不干的 `IDisposable`
        // `Disposable.Empty` 属性后面还会提及
    }
}
```

我们可以通过构造该类的实例并订阅它的方式来测试它：

```C#
var numbers = new MySequenceOfNumbers();
numbers.Subscribe(
    number => Console.WriteLine($"Received value: {number}"),
    () => Console.WriteLine("Sequence terminated"));
```

这会产生以下输出：

```
Received value 1
Received value 2
Received value 3
Sequence terminated
```

虽然 `MySequenceOfNumbers` 从技术上来说确实是 `IObservable<int>` 的正确实现，但它有点太简单了，没什么用处。因为我们通常在有感兴趣的事件时使用 Rx，但上面这个例子根本不是真正的响应式——它只是立即生成一组固定的数字。此外，该实现是阻塞的——它甚至不会从 `Subscribe` 返回，直到它生成完所有值。此示例只是简单展示了源如何向订阅者提供事件，但如果我们就是打算像上面的例子一样表示预定的数字序列，我们不如使用 `IEnumerable<T>` 实现，例如 `List<T>` 或数组。

## 在 Rx 中表示文件系统事件

让我们看一些更现实的案例。这是 .NET 的 `FileSystemWatcher` 的包装，将文件系统更改通知呈现为 `IObservable<FileSystemEventArgs>` 。


{{< callout type="info" >}}
  注意：这不一定是 Rx `FileSystemWatcher` 包装器的最佳设计。`FileSystemWatcher` 为几种不同类型的文件更改提供事件，其中之一的 `Renamed` 提供详细信息作为 `RenamedEventArgs` ，它派生自 `FileSystemEventArgs`，因此将所有内容压缩为单个事件流确实可行，但这对于想要访问重命名事件详细信息的应用程序来说会很不方便。更严重的设计问题在于它无法报告来自 `FileSystemWatcher.Error` 的多个事件。此类错误可能是暂时的且可恢复的，在这种情况下应用程序可能希望继续运行，但由于该类选择用一个 `IObservable<T>` 来表示所有可能的事件，并且它通过调用观察者的 `OnError` 方法来报告错误，此时 Rx 的规则迫使我们终止。想让它正常工作的话可以考虑使用 Rx 的 `Retry` 运算符，它可以在发生错误后自动重新订阅，但最好提供一个单独的 `IObservable<ErrorEventArgs>`，以便我们可以以非终止的方式报告错误。然而，这种额外的复杂性并不总是有必要的。这种设计的简单性意味着它将非常适合某些应用。正如软件设计的常见方式一样，没有一种放之四海而皆准的方法。
{{< /callout >}}


```C# {linenos=table,hl_lines=[45,67],linenostart=1,filename="RxFsEvents.cs"}
// 通过一个 RX 可观察序列来表示文件系统更改
// 注意：此示例过于简单，仅供学习。
//      它不能有效地处理多个订阅者、也没有使用 `IScheduler`、
//      还会在第一个错误发生后立即终止。
public class RxFsEvents : IObservable<FileSystemEventArgs>
{
    private readonly string folder;

    public RxFsEvents(string folder)
    {
        this.folder = folder;
    }

    public IDisposable Subscribe(IObserver<FileSystemEventArgs> observer)
    {
        // 在有多个订阅者的情况下会很低效。
        FileSystemWatcher watcher = new(this.folder);

        // `FileSystemWatcher` 的文档没有说明它会在哪个线程上引发事件（除非使用它的
        // `SynchronizationObject`，该对象和 Windows Forms 交互良好，但我们这里
        // 不方便使用），也没有承诺会等我们处理完一个事件后再发送下一个事件。

        // Mac，Windows，和 Linux 的实现各不相同，所以依赖文档没有保证的特性很不明智
        // （碰巧，.NET 7 的 Win32 实现会等到每个事件处理程序返回后才发送下一个事件，
        // 所以目前我们暂时能在 Windows 上忽略这个问题。此外，Linux 实现是用一个线程
        // 来完成这项工作的，但源代码中有一条注释说这可能需要改进——这也是只依赖文档中
        // 明确规定的行为的另一个原因）。

        // 因此，我们的首要问题是确保遵守 `IObserver<T>` 的规则。首先，我们需要确保每次
        // 只对观察者进行一次调用。一个更好的方案是使用 Rx 的 `IScheduler`，但由于
        // 我们还没有学到那儿，因此这里我们需要单独声明一个对象来使用 `lock` 语句。
        object sync = new();

        // 更微妙的地方在于，`FileSystemWatcher` 的文档没有明确说明在它报告了一个错误后
        // 我们是否还会接收到一些文件系统改变事件。由于没有关于线程的承诺，因此有可能存在
        // 竞争条件，导致我们在 `FileSystemWatcher` 报错后试图处理它发出的事件。因此，
        // 我们需要记住是否已经调用了 `OnError`，以确保在这种情况下不会违反
        // `IObserver<T>` 的规则。
        bool onErrorAlreadyCalled = false;

        void SendToObserver(object _, FileSystemEventArgs e)
        {
            lock (sync)
            {
                if (!onErrorAlreadyCalled)
                {
                    observer.OnNext(e); 
                }
            }
        }

        watcher.Created += SendToObserver;
        watcher.Changed += SendToObserver;
        watcher.Renamed += SendToObserver;
        watcher.Deleted += SendToObserver;

        watcher.Error += (_, e) =>
        {
            lock (sync)
            {
                // `FileSystemWatcher` 可能报告多个错误，但我们只能向
                // `IObservable<T>` 报告其中一个。

                // 译注：
                // 第67行高亮的 if 语句的条件部分应该和上面高亮的第 45 行一样为
                // if (!onErrorAlreadyCalled)
                // 否则第一次调用该方法时无法进入 if 语句。
                if (onErrorAlreadyCalled)
                {
                    observer.OnError(e.GetException());
                    onErrorAlreadyCalled = true; 
                    watcher.Dispose();
                }
            }
        };

        watcher.EnableRaisingEvents = true;

        return watcher;
    }
}
```

情况很快变得复杂起来。这个例子说明 `IObservable<T>` 实现需要遵守 `IObserver<T>` 规则。这通常是一件好事：它将并发性方面的杂乱问题集中在一个地方。订阅 `RxFsEvents` 的任何 `IObserver<FileSystemEventArgs>` 都不必担心并发问题，因为它可以依赖 `IObserver<T>` 规则，该规则保证了它一次只需处理一件事。要不是为了遵守这些规则，`RxFsEvents` 类还能更简单，但你就不得不在处理事件的代码中来解决可能的并发问题。处理并发问题本身已经够麻烦了，一旦并发问题涉及到多种类型就更难处理了。而 Rx 的 `IObserver<T>` 规则可以防止这种情况发生。

{{< callout type="info" >}}
  注意：这是 Rx 的一个重要功能，这些规则有利于简化观察者的工作，其重要性随着事件源或事件处理代码的复杂度增加而提升。
{{< /callout >}}

该示例代码还存在其他一些问题（除了已经提到的 API 设计问题）：比如当 `IObservable<T>` 实现产生模拟现实生活异步活动（例如文件系统更改）的事件时，应用程序通常需要某种方式来控制接收通知的线程。例如，UI 框架往往有线程亲和性要求，一般需要在特定线程上才能更新用户界面。Rx 提供了将通知重定向到不同调度程序（scheduler）的机制，我们可以用该机制解决问题，但我们通常希望能够为此类观察者提供一个 `IScheduler`，并让它通过 `IScheduler` 传递通知。我们将在后面的章节中讨论调度程序。

另一个问题是它不能有效地处理多个订阅者。理论上来说你可以多次调用 `IObservable<T>.Subscribe` ，但如果你真这么干了，那么每次都会创建一个新的 `FileSystemWatcher`。这事可能比你想象的更容易发生。假设我们有一个 `RxFsEvents` 的实例，并且想要以不同的方式处理不同类别的事件，那我们可以用 `Where` 运算符来定义可观察源，以我们想要的方式分割事件：

```C#
IObservable<FileSystemEventArgs> configChanges =
    fs.Where(e => Path.GetExtension(e.Name) == ".config");
IObservable<FileSystemEventArgs> deletions =
    fs.Where(e => e.ChangeType == WatcherChangeTypes.Deleted);
```

还记得我们在 [1.2.4.4](../1.2-key-types/#%e8%ae%a2%e9%98%85%e7%9a%84%e7%94%9f%e5%91%bd%e5%91%a8%e6%9c%9f%e5%92%8c%e7%bb%84%e5%90%88) 中学习的内容吗？当你对 `Where` 运算符返回的 `IObservable<T>` 对象调用 `Subscribe` 方法时，它将隐式调用其上游可观察序列的 `Subscribe` 方法。因此在本例中，如果你既调用了 `configChanges.Subscribe` 又调用了 `deletions.Subscribe`，那上游的 `fs.Subscribe` 就会被调用两次。假设 `fs` 是 `RxFsEvents` 类型的实例，则每次调用都会构造自己的 `FileSystemEventWatcher` ，这是很低效的。

Rx 提供了几种方法来解决这个问题。它提供了专门设计的、适用于不支持多订阅者的 `IObservable<T>` 的运算符，并将其包装在适配器中，示例如下：

```C#
IObservable<FileSystemEventArgs> fs =
    new RxFsEvents(@"c:\temp")
    .Publish()
    .RefCount();
```

但这个知识点太超前了（这些运算符将在 [发布运算符](../../part-3/3.6-publishing-operators/) 一章中介绍）。如果你想构建一种本质上对多订阅者友好的类型，你真正要做的就是跟踪所有订阅者并循环通知每个订阅者。如下是文件系统观察器的修改版本：

```C# {linenos=table,hl_lines=[],linenostart=1,filename="RxFsEventsMultiSubscriber.cs"}
public class RxFsEventsMultiSubscriber : IObservable<FileSystemEventArgs>
{
    private readonly object sync = new();
    private readonly List<Subscription> subscribers = new();
    private readonly FileSystemWatcher watcher;

    public RxFsEventsMultiSubscriber(string folder)
    {
        this.watcher = new FileSystemWatcher(folder);

        watcher.Created += SendEventToObservers;
        watcher.Changed += SendEventToObservers;
        watcher.Renamed += SendEventToObservers;
        watcher.Deleted += SendEventToObservers;

        watcher.Error += SendErrorToObservers;
    }

    public IDisposable Subscribe(IObserver<FileSystemEventArgs> observer)
    {
        Subscription sub = new(this, observer);
        lock (this.sync)
        {
            this.subscribers.Add(sub); 

            if (this.subscribers.Count == 1)
            {
                // 初次订阅时需要初始化 `FileSystemWatcher`
                watcher.EnableRaisingEvents = true;
            }
        }

        return sub;
    }

    private void Unsubscribe(Subscription sub)
    {
        lock (this.sync)
        {
            this.subscribers.Remove(sub);

            if (this.subscribers.Count == 0)
            {
                watcher.EnableRaisingEvents = false;
            }
        }
    }

    void SendEventToObservers(object _, FileSystemEventArgs e)
    {
        lock (this.sync)
        {
            foreach (var subscription in this.subscribers)
            {
                subscription.Observer.OnNext(e);
            }
        }
    }

    void SendErrorToObservers(object _, ErrorEventArgs e)
    {
        Exception x = e.GetException();
        lock (this.sync)
        {
            foreach (var subscription in this.subscribers)
            {
                subscription.Observer.OnError(x);
            }

            this.subscribers.Clear();
        }
    }

    private class Subscription : IDisposable
    {
        private RxFsEventsMultiSubscriber? parent;

        public Subscription(
            RxFsEventsMultiSubscriber rxFsEventsMultiSubscriber,
            IObserver<FileSystemEventArgs> observer)
        {
            this.parent = rxFsEventsMultiSubscriber;
            this.Observer = observer;
        }
        
        public IObserver<FileSystemEventArgs> Observer { get; }

        public void Dispose()
        {
            this.parent?.Unsubscribe(this);
            this.parent = null;
        }
    }
}
```

在这个例子中，无论 `Subscribe` 被调用多少次，都只会创建一个 `FileSystemWatcher` 实例。注意，为了让 `Subscribe` 能返回 `IDisposable`，我们需要引入一个嵌套类 `Subscription`。而在本章的第一个案例 `MySequenceOfNumbers.cs` 中，我们不需要这么做是因为它在返回前就已经终止了序列，因此处于方便考虑，我们让它返回一个由 Rx 提供的 `Disposable.Empty` 属性（如果你必须返回一个 `IDisposable`、但又没啥需要释放的资源，返回一个 `Disposable.Empty` 不失为一种好选择）。在 `RxFsEvents.cs` 里，我返回了 `FileSystemWatcher` 本身（这么做是可以的，因为 `FileSystemWatcher.Dispose` 可以关闭 `watcher` 对象，并且每个订阅者都会声明一个自己的 `FileSystemWatcher`）。不过考虑到我们已经改进了 `RxFsEvents.cs`，在 `RxFsEventsMultiSubscriber.cs` 中单个 `FileSystemWatcher` 支持多个观察者，所以我们需要在观察者取消订阅时做更多工作。

当 `Subscribe` 返回的 `Subscription` 实例被释放时，它会将自己从订阅者列表中删除，以确保它不会再收到任何通知。如果它是最后一位订阅者，它还会将 `FileSystemWatcher` 的 `EnableRaisingEvents` 设置为 `false`，这样 `FileSystemWatcher` 就不会再监视指定目录。

`RxFsEventsMultiSubscriber.cs` 看起来比 `RxFsEvents.cs` 真实多了！一个真正的随时可能产生事件的事件源（这正是 Rx 的优势所在），并且它现在可以智能地处理多个订阅者。然而我们通常不会这样写代码——这个类完全可以独立工作，它甚至不需要引用 `System.Reactive` 包，因为它引用的跟 Rx 相关的类型仅仅是 `IObservable<T>` 和 `IObserver<T>`，这两种类型都内置于 .NET 运行时库中。而在实践中，我们通常会使用 `System.Reactive` 中的助手类（helper），因为它们可以替我们做很多工作。

例如，假设我们只关心 `Changed` 事件。我们可以这样写：

```C#  {linenos=table,hl_lines=[],linenostart=1,filename="FromEventPattern.cs"}
FileSystemWatcher watcher = new (@"c:\temp");
IObservable<FileSystemEventArgs> changes = Observable
    .FromEventPattern<FileSystemEventArgs>(watcher, nameof(watcher.Changed))
    .Select(ep => ep.EventArgs);
watcher.EnableRaisingEvents = true;
```

If we were aiming to write a fully-featured wrapper for FileSystemWatcher suitable for many different scenarios, it might be worth writing a specialized IObservable<T> implementation as shown earlier. (We could easily extend this last example to watch all of the events. We'd just use the FromEventPattern once for each event, and then use Observable.Merge to combine the four resulting observables into one. The only real benefit we're getting from a full custom implementation is that we can automatically start and stop the FileSystemWatcher depending on whether there are currently any observers.) But if we just need to represent some events as an IObservable<T> so that we can work with them in our application, we can just use this simpler approach.
此处我们使用了 `System.Reactive` 库中 `Observable` 类的 `FromEventPattern` 助手类，它可以从任何常规的 .NET 事件中创建 `IObservable<T>`（“常规的 .NET 事件”指事件处理程序采用两个参数：第一个参数是 `object` 类型，表示事件发送者；第二个参数是 `EventArgs` 派生类型，包含事件相关的信息）。这不像 `RxFsEventsMultiSubscriber.cs` 案例那么灵活，它只关注并报告某一个文件系统事件，而且我们必须手动启动 `FileSystemWatcher`（必要的话还得手动停止）。但对于某些应用程序来说这就够了，而且需要编写的代码要少得多。

如果我们的目标是为 `FileSystemWatcher` 编写一个适用于各种场景的全功能包装器，那也许值得专门编写一个 `IObservable<T>` 实现，就像前面展示的 `RxFsEventsMultiSubscriber.cs` 案例一样；但如果我们只是想将一些事件作为 `IObservable<T>` 表示，以便我们可以在自己的程序中使用它们，我们可以使用这种更简单的方法（同样，我们也可以轻松扩展 `FromEventPattern` 示例来观察所有事件：我们只需对每个事件调用一次 `FromEventPattern`，然后用 `Observable.Merge` 将四个可观察序列组合成一个。我们从完整的自定义实现中获得的唯一真正的好处是我们可以根据当前是否有观察者来自动启动和停止 `FileSystemWatcher`）。

In practice, we almost always get System.Reactive to implement IObservable<T> for us. Even if we want to take control of certain aspects (such as automatically starting up and shutting down the FileSystemWatcher in these examples) we can almost always find a combination of operators that enable this. The following code uses various methods from System.Reactive to return an IObservable<FileSystemEventArgs> that has all the same functionality as the fully-featured hand-written RxFsEventsMultiSubscriber above, but with considerably less code.
在实践中，我们几乎总是让 System.Reactive 为我们实现 IObservable<T> 。即使我们想要控制某些方面（例如在这些示例中自动启动和关闭 FileSystemWatcher ），我们几乎总能找到实现此目的的运算符组合。以下代码使用 System.Reactive 中的各种方法返回 IObservable<FileSystemEventArgs> ，该 IObservable<FileSystemEventArgs> 具有与上面全功能手写 RxFsEventsMultiSubscriber 相同的功能，但具有相当多的功能更少的代码。

```C# {linenos=table,hl_lines=[],linenostart=1,filename="ObserveFileSystem.cs"}
IObservable<FileSystemEventArgs> ObserveFileSystem(string folder)
{
    return 
        // Observable.Defer enables us to avoid doing any work
        // until we have a subscriber.
        Observable.Defer(() =>
            {
                FileSystemWatcher fsw = new(folder);
                fsw.EnableRaisingEvents = true;

                return Observable.Return(fsw);
            })
        // Once the preceding part emits the FileSystemWatcher
        // (which will happen when someone first subscribes), we
        // want to wrap all the events as IObservable<T>s, for which
        // we'll use a projection. To avoid ending up with an
        // IObservable<IObservable<FileSystemEventArgs>>, we use
        // SelectMany, which effectively flattens it by one level.
        .SelectMany(fsw =>
            Observable.Merge(new[]
                {
                    Observable.FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
                        h => fsw.Created += h, h => fsw.Created -= h),
                    Observable.FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
                        h => fsw.Changed += h, h => fsw.Changed -= h),
                    Observable.FromEventPattern<RenamedEventHandler, FileSystemEventArgs>(
                        h => fsw.Renamed += h, h => fsw.Renamed -= h),
                    Observable.FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
                        h => fsw.Deleted += h, h => fsw.Deleted -= h)
                })
            // FromEventPattern supplies both the sender and the event
            // args. Extract just the latter.
            .Select(ep => ep.EventArgs)
            // The Finally here ensures the watcher gets shut down once
            // we have no subscribers.
            .Finally(() => fsw.Dispose()))
        // This combination of Publish and RefCount means that multiple
        // subscribers will get to share a single FileSystemWatcher,
        // but that it gets shut down if all subscribers unsubscribe.
        .Publish()
        .RefCount();
}
```

I've used a lot of methods there, most of which I've not talked about before. For that example to make any sense, I clearly need to start describing the numerous ways in which the System.Reactive package can implement IObservable<T> for you.
我在那里使用了很多方法，其中大部分我以前没有谈过。为了使该示例有意义，我显然需要开始描述 System.Reactive 包为你实现 IObservable<T> 的多种方式。

Simple factory methods 简单的工厂方法
Due to the large number of methods available for creating observable sequences, we will break them down into categories. Our first category of methods create IObservable<T> sequences that produce at most a single result.
由于可用于创建可观察序列的方法有很多，我们将它们分为几类。我们的第一类方法创建最多产生一个结果的 IObservable<T> 序列。

Observable.Return
One of the simplest factory methods is Observable.Return<T>(T value), which you've already seen in the Quiescent example in the preceding chapter. This method takes a value of type T and returns an IObservable<T> which will produce this single value and then complete. In a sense, this wraps a value in an IObservable<T>; it's conceptually similar to writing new T[] { value }, in that it's a sequence containing just one element. You could also think of it as being the Rx equivalent of Task.FromResult, which you can use when you have a value of some type T, and need to pass it to something that wants a Task<T>.
最简单的工厂方法之一是 Observable.Return<T>(T value) ，你已经在上一章的 Quiescent 示例中看到过它。此方法采用 T 类型的值并返回 IObservable<T> ，它将生成此单个值，然后完成。从某种意义上说，这将一个值包装在 IObservable<T> 中；它在概念上类似于编写 new T[] { value } ，因为它是一个仅包含一个元素的序列。你还可以将其视为 Task.FromResult 的 Rx 等价物，当你有某种类型的值 T 时可以使用它，并且需要将其传递给需要 Task<T> 。

IObservable<string> singleValue = Observable.Return<string>("Value");
I specified the type parameter for clarity, but this is not necessary as the compiler can infer the type from argument provided:
为了清楚起见，我指定了类型参数，但这不是必需的，因为编译器可以从提供的参数推断类型：

IObservable<string> singleValue = Observable.Return("Value");
Return produces a cold observable: each subscriber will receive the value immediately upon subscription. (Hot and cold observables were described in the preceding chapter.)
Return 产生一个冷可观察值：每个订阅者将在订阅后立即收到该值。 （热可观测量和冷可观测量已在前一章中描述。）

Observable.Empty 可观察.空
Sometimes it can be useful to have an empty sequence. .NET's Enumerable.Empty<T>() does this for IEnumerable<T>, and Rx has a direct equivalent in the form of Observable.Empty<T>(), which returns an empty IObservable<T>. We need to provide the type argument because there's no value from which the compiler can infer the type.
有时，有一个空序列可能很有用。 .NET 的 Enumerable.Empty<T>() 对 IEnumerable<T> 执行此操作，而 Rx 有 Observable.Empty<T>() 形式的直接等效项，它返回空的 IObservable<T> 。我们需要提供类型参数，因为编译器无法从中推断出类型的值。

IObservable<string> empty = Observable.Empty<string>();
In practice, an empty sequence is one that immediately calls OnCompleted on any subscriber.
实际上，空序列是立即调用任何订阅者的 OnCompleted 的序列。

In comparison with IEnumerable<T>, this is just the Rx equivalent of an empty list, but there's another way to look at it. Rx is a powerful way to model asynchronous processes, so you could think of this as being similar to a task that completes immediately without producing any result—so it has a conceptual resemblance to Task.CompletedTask. (This is not as close an analogy as that between Observable.Return and Task.FromResult, because in that case we're comparing an IObservable<T> with a Task<T>, whereas here we're comparing an IObservable<T> with a Task—the only way for a task to complete without producing anything is if we use the non-generic version of Task.)
与 IEnumerable<T> 相比，这只是空列表的 Rx 等价物，但还有另一种看待它的方式。 Rx 是一种对异步流程进行建模的强大方法，因此你可以将其视为类似于立即完成而不产生任何结果的任务，因此它在概念上类似于 Task.CompletedTask 。 （这不像 Observable.Return 和 Task.FromResult 之间的类比那么接近，因为在这种情况下，我们将 IObservable<T> 与 Task<T> 与 Task 进行比较 - 完成任务而不产生任何内容的唯一方法是使用 Task 。）

Observable.Never 可观察。从不
The Observable.Never<T>() method returns a sequence which, like Empty, does not produce any values, but unlike Empty, it never ends. In practice, that means that it never invokes any method (neither OnNext, OnCompleted, nor OnError) on subscribers. Whereas Observable.Empty<T>() completes immediately, Observable.Never<T> has infinite duration.
Observable.Never<T>() 方法返回一个序列，与 Empty 一样，不会产生任何值，但与 Empty 不同，它永远不会结束。实际上，这意味着它永远不会调用订阅者的任何方法（既不是 OnNext 、 OnCompleted 也不是 OnError ）。 Observable.Empty<T>() 会立即完成，而 Observable.Never<T> 则具有无限的持续时间。

IObservable<string> never = Observable.Never<string>();
It might not seem obvious why this could be useful. I gave one possible use in the last chapter: you could use this in a test to simulate a source that wasn't producing any values, perhaps to enable your test to validate timeout logic.
为什么这可能有用似乎并不明显。我在上一章中给出了一个可能的用途：你可以在测试中使用它来模拟不产生任何值的源，也许可以使你的测试能够验证超时逻辑。

It can also be used in places where we use observables to represent time-based information. Sometimes we don't actually care what emerges from an observable; we might care only when something (anything) happens. (We saw an example of this "observable sequence used purely for timing purposes" concept in the preceding chapter, although Never wouldn't make sense in that particular scenario. The Quiescent example used the Buffer operator, which works over two observable sequences: the first contains the items of interest, and the second is used purely to determine how to cut the first into chunks. Buffer doesn't do anything with the values produced by the second observable: it pays attention only to when values emerge, completing the previous chunk each time the second observable produces a value. And if we're representing temporal information it can sometimes be useful to have a way to represent the idea that some event never occurs.)
它也可以用在我们使用可观察量来表示基于时间的信息的地方。有时我们实际上并不关心可观察到的结果；我们只关心可观察到的结果。我们可能只关心某事（任何事情）发生时。 （我们在前一章中看到了“纯粹用于计时目的的可观察序列”概念的示例，尽管 Never 在该特定场景中没有意义。 Quiescent 示例使用 Buffer 运算符，它作用于两个可观察的序列：第一个包含感兴趣的项目，第二个纯粹用于确定如何将第一个序列切割成块。 Buffer 不'不要对第二个可观察量产生的值做任何事情：它只关注值何时出现，每次第二个可观察量产生值时完成前一个块。如果我们表示时间信息，有时有一个有用的表示某些事件从未发生的想法的方式。）

As an example of where you might want to use Never for timing purposes, suppose you were using some Rx-based library that offered a timeout mechanism, where an operation would be cancelled when some timeout occurs, and the timeout is itself modelled as an observable sequence. If for some reason you didn't want a timeout, and just want to wait indefinitely, you could specify a timeout of Observable.Never.
作为你可能想要使用 Never 进行计时的示例，假设你正在使用一些提供超时机制的基于 Rx 的库，其中发生超时时操作将被取消，并且超时本身被建模为一个可观察的序列。如果由于某种原因你不希望超时，而只想无限期等待，则可以指定超时 Observable.Never 。

Observable.Throw 可观察.抛出
Observable.Throw<T>(Exception) returns a sequence that immediately reports an error to any subscriber. As with Empty and Never, we don't supply a value to this method (just an exception) so we need to provide a type parameter so that it knows what T to use in the IObservable<T> that it returns. (It will never actually a produce a T, but you can't have an instance of IObservable<T> without picking some particular type for T.)
Observable.Throw<T>(Exception) 返回一个立即向任何订阅者报告错误的序列。与 Empty 和 Never 一样，我们不向此方法提供值（只是一个例外），因此我们需要提供一个类型参数，以便它知道 T 中使用。 （它实际上永远不会产生 T ，但是如果不为 T 选择某种特定类型，就无法拥有 IObservable<T> 的实例。）

IObservable<string> throws = Observable.Throw<string>(new Exception()); 
Observable.Create 可观察.创建
The Create factory method is more powerful than the other creation methods because it can be used to create any kind of sequence. You could implement any of the preceding four methods with Observable.Create. The method signature itself may seem more complex than necessary at first, but becomes quite natural once you are used to it.
Create 工厂方法比其他创建方法更强大，因为它可用于创建任何类型的序列。你可以使用 Observable.Create 实现上述四种方法中的任何一种。方法签名本身起初可能看起来比必要的更复杂，但一旦习惯了它就会变得非常自然。

```C#
// Creates an observable sequence from a specified Subscribe method implementation.
public static IObservable<TSource> Create<TSource>(
    Func<IObserver<TSource>, IDisposable> subscribe)
{...}
public static IObservable<TSource> Create<TSource>(
    Func<IObserver<TSource>, Action> subscribe)
{...}
```

You provide this with a delegate that will be executed each time a subscription is made. Your delegate will be passed an IObserver<T>. Logically speaking, this represents the observer passed to the Subscribe method, although in practice Rx puts a wrapper around that for various reasons. You can call the OnNext/OnError/OnCompleted methods as you need. This is one of the few scenarios where you will work directly with the IObserver<T> interface. Here's a simple example that produces three items:
你为其提供一个委托，该委托将在每次订阅时执行。你的委托将被传递一个 IObserver<T> 。从逻辑上讲，这表示传递给 Subscribe 方法的观察者，尽管实际上 Rx 由于各种原因对其进行了包装。你可以根据需要调用 OnNext / OnError / OnCompleted 方法。这是你将直接使用 IObserver<T> 界面的少数场景之一。这是一个生成三个项目的简单示例：

```C#
private IObservable<int> SomeNumbers()
{
    return Observable.Create<int>(
        (IObserver<string> observer) =>
        {
            observer.OnNext(1);
            observer.OnNext(2);
            observer.OnNext(3);
            observer.OnCompleted();

            return Disposable.Empty;
        });
}
```

Your delegate must return either an IDisposable or an Action to enable unsubscription. When the subscriber disposes their subscription in order to unsubscribe, Rx will invoke Dispose() on the IDisposable you returned, or in the case where you returned an Action, it will invoke that.
你的代理必须返回 IDisposable 或 Action 才能取消订阅。当订阅者为了取消订阅而处置其订阅时，Rx 将在你返回的 IDisposable 上调用 Dispose() ，或者在你返回 Action 的情况下，它会调用 Dispose() 。会调用它。

This example is reminiscent of the MySequenceOfNumbers example from the start of this chapter, in that it immediately produces a few fixed values. The main difference in this case is that Rx adds some wrappers that can handle awkward situations such as re-entrancy. Rx will sometimes automatically defer work to prevent deadlocks, so it's possible that code consuming the IObservable<string> returned by this method will see a call to Subscribe return before the callback in the code above runs, in which case it would be possible for them to unsubscribe inside their OnNext handler.
这个例子让人想起本章开头的 MySequenceOfNumbers 例子，因为它立即产生一些固定值。这种情况下的主要区别在于，Rx 添加了一些包装器，可以处理诸如重入之类的尴尬情况。 Rx 有时会自动推迟工作以防止死锁，因此使用此方法返回的 IObservable<string> 的代码可能会在上面代码中的回调运行之前看到对 Subscribe 的调用返回，在这种情况下，他们可以在其 OnNext 处理程序内取消订阅。

The following sequence diagram shows how this could occur in practice. Suppose the IObservable<int> returned by SomeNumbers has been wrapped by Rx in a way that ensures that subscription occurs in some different execution context. We'd typically determine the context by using a suitable scheduler. (The SubscribeOn operator creates such a wrapper.) We might use the TaskPoolScheduler in order to ensure that the subscription occurs on some task pool thread. So when our application code calls Subscribe, the wrapper IObservable<int> doesn't immediately subscribe to the underlying observable. Instead it queues up a work item with the scheduler to do that, and then immediately returns without waiting for that work to run. This is how our subscriber can be in possession of an IDisposable representing the subscription before Observable.Create invokes our callback. The diagram shows the subscriber then making this available to the observer.
下面的序列图显示了这在实践中是如何发生的。假设 SomeNumbers 返回的 IObservable<int> 已被 Rx 包装，以确保订阅发生在某个不同的执行上下文中。我们通常会使用合适的调度程序来确定上下文。 （ SubscribeOn 运算符创建这样的包装器。）我们可以使用 TaskPoolScheduler 来确保订阅发生在某个任务池线程上。因此，当我们的应用程序代码调用 Subscribe 时，包装器 IObservable<int> 不会立即订阅底层可观察对象。相反，它会使用调度程序将一个工作项排队来执行此操作，然后立即返回，而不等待该工作运行。这就是我们的订阅者如何在 Observable.Create 调用我们的回调之前拥有代表订阅的 IDisposable 。该图显示了订阅者然后将其提供给观察者。

A sequence diagram with 6 participants: Subscriber, Rx IObservable Wrapper, Scheduler, Observable.Create, Rx IObserver Wrapper, and Observer. It shows the following messages. Subscriber sends "Subscribe()" to Rx IObservable Wrapper. Rx IObservable Wrapper sends "Schedule Subscribe()" to Scheduler. Rx IObservable Wrapper returns "IDisposable (subscription)" to Subscriber. Subscriber sends "Set subscription IDisposable" to Observer. Scheduler sends "Subscribe()" to Observable.Create. Observable.Create sends "OnNext(1)" to Rx IObserver Wrapper. Rx IObserver Wrapper sends "OnNext(1)" to Observer. Observable.Create sends "OnNext(2)" to Rx IObserver Wrapper. Rx IObserver Wrapper sends "OnNext(2)" to Observer. Observer sends "subscription.Dispose()" to Rx IObservable Wrapper. Observable.Create sends "OnNext(3)" to Rx IObserver Wrapper. Observable.Create sends "OnCompleted()" to Rx IObserver Wrapper.

The diagram shows the scheduler call Subscribe on the underlying observable after this, and that will mean the call back we passed to Observable.Create<int> will now run. Our callback calls OnNext, but it is not passed the real observer: instead it is passed another Rx-generated wrapper. That wrapper initially forwards calls directly onto the real observer, but our diagram shows that when the real observer (all the way over on the right) receives the its second call (OnNext(2)) it unsubscribes by calling Dispose on the IDisposable that was returned when we subscribed to the Rx IObservable wrapper. The two wrappers here—the IObservable and IObserver wrappers—are connected, so when we unsubscribe from the IObservable wrapper, it tells the IObserver wrapper that the subscription is being shut down. This means that when our Observable.Create<int> callback calls OnNext(3) on the IObserver wrapper, that wrapper does not forward it to the real observer, because it knows that that observer has already unsubscribed. (It also doesn't forward the OnCompleted, for the same reason.)
该图显示了在此之后对底层可观察对象的调度程序调用 Subscribe ，这意味着我们传递给 Observable.Create<int> 的回调现在将运行。我们的回调调用 OnNext ，但它没有传递给真正的观察者：而是传递给另一个 Rx 生成的包装器。该包装器最初将调用直接转发到真实的观察者，但我们的图表显示，当真实的观察者（一直在右侧）收到第二个调用（ OnNext(2) ）时，它通过调用 Dispose 位于我们订阅 Rx IObservable 包装器时返回的 IDisposable 上。这里的两个包装器 - IObservable 和 IObserver 包装器 - 是连接的，因此当我们取消订阅 IObservable 包装器时，它会告诉 IObserver 订阅正在关闭的包装器。这意味着当我们的 Observable.Create<int> 回调在 IObserver 包装器上调用 OnNext(3) 时，该包装器不会将其转发给真正的观察者，因为它知道该观察者有已经取消订阅。 （出于同样的原因，它也不会转发 OnCompleted 。）

You might be wondering how the IDisposable we return to Observable.Create can ever do anything useful. It's the return value of the callback, so we can only return it to Rx as the last thing our callback does. Won't we always have finished our work by the time we return, meaning there's nothing to cancel? Not necessarily—we might kick off some work that continues to run after we return. This next example does that, meaning that the unsubscription action it returns is able to do something useful: it sets a cancellation token that is being observed by the loop that generates our observable's output. (This returns a callback instead of an IDisposable—Observable.Create offers overloads that let you do either. In this case, Rx will invoke our callback when the subscription is terminated early.)
你可能想知道我们返回到 Observable.Create 的 IDisposable 如何能够做任何有用的事情。它是回调的返回值，因此我们只能将其返回给 Rx 作为回调的最后一件事。我们回来时不是总是已经完成工作，这意味着没有什么可以取消的吗？不一定——我们可能会开始一些在我们回来后继续进行的工作。下一个示例就是这样做的，这意味着它返回的取消订阅操作能够做一些有用的事情：它设置一个由生成可观察结果输出的循环正在观察的取消标记。 （这会返回一个回调而不是 IDisposable - Observable.Create 提供了可以让你执行任一操作的重载。在这种情况下，Rx 将在订阅提前终止时调用我们的回调。）

```C# {linenos=table,hl_lines=[],linenostart=1,filename="KeyPresses.cs"}
IObservable<char> KeyPresses() =>
    Observable.Create<char>(observer =>
    {
        CancellationTokenSource cts = new();
        Task.Run(() =>
        {
            while (!cts.IsCancellationRequested)
            {
                ConsoleKeyInfo ki = Console.ReadKey();
                observer.OnNext(ki.KeyChar);
            }
        });

        return () => cts.Cancel();
    });
```

This illustrates how cancellation won't necessarily take effect immediately. The Console.ReadKey API does not offer an overload accepting a CancellationToken, so this observable won't be able to detect that cancellation is requested until the user next presses a key, causing ReadKey to return.
这说明取消不一定会立即生效。 Console.ReadKey API 不提供接受 CancellationToken 的重载，因此此可观察量将无法检测到请求取消，直到用户下一次按下某个键，从而导致 ReadKey 返回。

Bearing in mind that cancellation might have been requested while we were waiting for ReadKey to return, you might think we should check for that after ReadKey returns and before calling OnNext. In fact it doesn't matter if we don't. Rx has a rule that says an observable source must not call into an observer after a call to Dispose on that observer's subscription returns. To enforce that rule, if the callback you pass to Observable.Create continues to call methods on its IObserver<T> after a request to unsubscribe, Rx just ignores the call. This is one reason why the IObserver<T> it passes to you is a wrapper: it can intercept the calls before they are passed to the underlying observer. However, that convenience means there are two important things to be aware of
请记住，在我们等待 ReadKey 返回时可能已请求取消，你可能认为我们应该在 ReadKey 返回之后、调用 OnNext .事实上，如果我们不这样做也没关系。 Rx 有一条规则，规定可观察源在对观察者订阅的 Dispose 调用返回后不得调用该观察者。为了强制执行该规则，如果你传递给 Observable.Create 的回调在请求取消订阅后继续调用其 IObserver<T> 上的方法，则 Rx 将忽略该调用。这就是为什么它传递给你的 IObserver<T> 是一个包装器的原因之一：它可以在调用传递给底层观察者之前拦截它们。然而，这种便利意味着有两件重要的事情需要注意

if you do ignore attempts to unsubscribe and continue to do work to produce items, you are just wasting time because nothing will receive those items
如果你确实忽略了取消订阅的尝试并继续工作来生产物品，那么你只是在浪费时间，因为没有人会收到这些物品
if you call OnError it's possible that nothing is listening and that the error will be completely ignored.
如果你调用 OnError ，则可能没有任何内容正在侦听，并且错误将被完全忽略。
There are overloads of Create designed to support async methods. This next method exploits this to be able to use the asynchronous ReadLineAsync method to present lines of text from a file as an observable source.
Create 的重载旨在支持 async 方法。下一个方法利用这一点，能够使用异步 ReadLineAsync 方法将文件中的文本行呈现为可观察源。

```C# {linenos=table,hl_lines=[],linenostart=1,filename="ReadFileLines.cs"}
IObservable<string> ReadFileLines(string path) =>
    Observable.Create<string>(async (observer, cancellationToken) =>
    {
        using (StreamReader reader = File.OpenText(path))
        {
            while (cancellationToken.IsCancellationRequested)
            {
                string? line = await reader.ReadLineAsync(cancellationToken).ConfigureAwait(false);
                if (line is null)
                {
                    break;
                }

                observer.OnNext(line);
            }

            observer.OnCompleted();
        }
    });
```

Reading data from a storage device typically doesn't happen instantaneously (unless it happens to be in the filesystem cache already), so this source will provide data as quickly as it can be read from storage.
从存储设备读取数据通常不会立即发生（除非它恰好已经在文件系统缓存中），因此该源将在从存储读取数据时尽快提供数据。

Notice that because this is an async method, it will typically return to its caller before it completes. (The first await that actually has to wait returns, and the remainder of the method runs via a callback when the work completes.) That means that subscribers will typically be in possession of the IDisposable representing their subscription before this method finishes, so we're using a different mechanism to handle unsubscription here. This particular overload of Create passes its callback not just an IObserver<T> but also a CancellationToken, with which it will request cancellation when unsubscription occurs.
请注意，因为这是一个 async 方法，所以它通常会在完成之前返回到其调用者。 （实际上必须等待的第一个 await 返回，并且该方法的其余部分在工作完成时通过回调运行。）这意味着订阅者通常会拥有 IDisposable 表示在此方法完成之前的订阅，因此我们在这里使用不同的机制来处理取消订阅。 Create 的这种特定重载不仅传递其回调 IObserver<T> ，还传递 CancellationToken ，当取消订阅发生时，它将通过该回调请求取消。

File IO can encounter errors. The file we're looking for might not exist, or we might be unable to open it due to security restrictions, or because some other application is using it. The file might be on a remote storage server, and we could lose network connectivity. For this reason, we must expect exceptions from such code. This example has done nothing to detect exceptions, and yet the IObservable<string> that this ReadFileLines method returns will in fact report any exceptions that occur. This is because the Create method will catch any exception that emerges from our callback and report it with OnError. (If our code already called OnComplete on the observer, Rx won't call OnError because that would violate the rules. Instead it will silently drop the exception, so it's best not to attempt to do any work after you call OnCompleted.)
文件 IO 可能会遇到错误。我们正在查找的文件可能不存在，或者由于安全限制或某些其他应用程序正在使用它，我们可能无法打开它。该文件可能位于远程存储服务器上，我们可能会失去网络连接。因此，我们必须预期此类代码会出现异常。此示例没有执行任何操作来检测异常，但此 ReadFileLines 方法返回的 IObservable<string> 实际上会报告发生的任何异常。这是因为 Create 方法将捕获回调中出现的任何异常，并使用 OnError 报告该异常。 （如果我们的代码已经在观察者上调用了 OnComplete ，Rx将不会调用 OnError ，因为这会违反规则。相反，它会默默地丢弃异常，所以最好不要尝试在调用 OnCompleted 之后执行任何工作。）

This automatic exception delivery is another example of why the Create factory method is the preferred way to implement custom observable sequences. It is almost always a better option than creating custom types that implement the IObservable<T> interface. This is not just because it saves you some time. It's also that Rx tackles the intricacies that you may not think of such as thread safety of notifications and disposal of subscriptions.
这种自动异常传递是为什么 Create 工厂方法是实现自定义可观察序列的首选方法的另一个例子。它几乎总是比创建实现 IObservable<T> 接口的自定义类型更好的选择。这不仅仅是因为它可以节省你一些时间。 Rx 还解决了你可能没有想到的复杂问题，例如通知的线程安全性和订阅的处理。

The Create method entails lazy evaluation, which is a very important part of Rx. It opens doors to other powerful features such as scheduling and combination of sequences that we will see later. The delegate will only be invoked when a subscription is made. So in the ReadFileLines example, it won't attempt to open the file until you subscribe to the IObservable<string> that is returned. If you subscribe multiple times, it will execute the callback each time. (So if the file has changed, you can retrieve the latest contents by calling Subscribe again.)
Create 方法需要惰性求值，这是 Rx 中非常重要的一部分。它为其他强大的功能打开了大门，例如我们稍后将看到的调度和序列组合。仅当进行订阅时才会调用委托。因此，在 ReadFileLines 示例中，在你订阅返回的 IObservable<string> 之前，它不会尝试打开文件。如果多次订阅，每次都会执行回调。 （因此，如果文件发生更改，你可以通过再次调用 Subscribe 来检索最新的内容。）

As an exercise, try to build the Empty, Return, Never & Throw extension methods yourself using the Create method. If you have Visual Studio or LINQPad available to you right now, code it up as quickly as you can, or if you have Visual Studio Code, you could create a new Polyglot Notebook. (Polyglot Notebooks make Rx available automatically, so you can just write a C# cell with a suitable using directive, and you're up and running.) If you don't (perhaps you are on the train on the way to work), try to conceptualize how you would solve this problem.
作为练习，尝试使用 Create 、 Return 、 Never 和 Throw 扩展方法> 方法。如果你现在有可用的 Visual Studio 或 LINQPad，请尽快对其进行编码，或者如果你有 Visual Studio Code，则可以创建一个新的多语言笔记本。 （多语言笔记本使 Rx 自动可用，因此你只需编写一个带有合适的 using 指令的 C# 单元，然后就可以启动并运行。）如果你不这样做（也许你正在火车上工作方式），尝试概念化你将如何解决这个问题。

You completed that last step before moving onto this paragraph, right? Because you can now compare your versions with these examples of Empty, Return, Never and Throw recreated with Observable.Create:
你在进入本段之前已经完成了最后一步，对吧？因为你现在可以将你的版本与使用 Observable.Create 、 Return 、 Never 和 Throw 示例进行比较>：

```C#
public static IObservable<T> Empty<T>()
{
    return Observable.Create<T>(o =>
    {
        o.OnCompleted();
        return Disposable.Empty;
    });
}

public static IObservable<T> Return<T>(T value)
{
    return Observable.Create<T>(o =>
    {
        o.OnNext(value);
        o.OnCompleted();
        return Disposable.Empty;
    });
}

public static IObservable<T> Never<T>()
{
    return Observable.Create<T>(o =>
    {
        return Disposable.Empty;
    });
}

public static IObservable<T> Throws<T>(Exception exception)
{
    return Observable.Create<T>(o =>
    {
        o.OnError(exception);
        return Disposable.Empty;
    });
}
```

You can see that Observable.Create provides the power to build our own factory methods if we wish.
你可以看到，如果我们愿意， Observable.Create 提供了构建我们自己的工厂方法的能力。

Observable.Defer 可观察.延迟
One very useful aspect of Observable.Create is that it provides a place to put code that should run only when subscription occurs. Often, libraries will make IObservable<T> properties available that won't necessarily be used by all applications, so it can be useful to defer the work involved until you know you will really need it. This deferred initialization is inherent to how Observable.Create works, but what if the nature of our source means that Observable.Create is not a good fit? How can we perform deferred initialization in that case? Rx providers Observable.Defer for this purpose.
Observable.Create 的一个非常有用的方面是，它提供了一个放置仅在订阅发生时才运行的代码的位置。通常，库会提供 IObservable<T> 属性，但不一定会被所有应用程序使用，因此推迟所涉及的工作直到你知道确实需要它可能会很有用。这种延迟初始化是 Observable.Create 工作方式所固有的，但是如果我们的源代码的性质意味着 Observable.Create 不合适怎么办？在这种情况下我们如何执行延迟初始化？用于此目的的 Rx 提供者 Observable.Defer 。

I've already used Defer once. The ObserveFileSystem method returned an IObservable<FileSystemEventArgs> reporting changes in a folder. It was not a good candidate for Observable.Create because it provided all the notifications we wanted as .NET events, so it made sense to use Rx's event adaptation features. But we still wanted to defer the creation of the FileSystemWatcher until the moment of subscription, which is why that example used Observable.Defer.
我已经使用过一次 Defer 。 ObserveFileSystem 方法返回 IObservable<FileSystemEventArgs> 报告文件夹中的更改。它不是 Observable.Create 的良好候选者，因为它提供了我们想要的所有通知作为 .NET 事件，因此使用 Rx 的事件适应功能是有意义的。但我们仍然希望将 FileSystemWatcher 的创建推迟到订阅时刻，这就是该示例使用 Observable.Defer 的原因。

Observable.Defer takes a callback that returns an IObservable<T>, and Defer wraps this with an IObservable<T> that invokes that callback upon subscription. To show the effect, I'm first going to show an example that does not use Defer:
Observable.Defer 接受一个返回 IObservable<T> 的回调，而 Defer 用 IObservable<T> 包装它，在订阅时调用该回调。为了展示效果，我首先展示一个不使用 Defer 的示例：


```C#
static IObservable<int> WithoutDeferal()
{
    Console.WriteLine("Doing some startup work...");
    return Observable.Range(1, 3);
}

Console.WriteLine("Calling factory method");
IObservable<int> s = WithoutDeferal();

Console.WriteLine("First subscription");
s.Subscribe(Console.WriteLine);

Console.WriteLine("Second subscription");
s.Subscribe(Console.WriteLine);
```

This produces the following output:
这会产生以下输出：


```
Calling factory method
Doing some startup work...
First subscription
1
2
3
Second subscription
1
2
3
```

As you can see, the "Doing some startup work... message appears when we call the factory method, and before we've subscribed. So if nothing ever subscribed to the IObservable<int> that method returns, the work would be done anyway, wasting time and energy. Here's the Defer version:
正如你所看到的，当我们调用工厂方法时以及订阅之前，会出现 "Doing some startup work... 消息。因此，如果没有任何内容订阅该方法返回的 IObservable<int> ，那么工作无论如何都会完成，浪费时间和精力。这是 Defer 版本：

```C#
static IObservable<int> WithDeferal()
{
    return Observable.Defer(() =>
    {
        Console.WriteLine("Doing some startup work...");
        return Observable.Range(1, 3);
    });
}
```

If we were to use this with similar code to the first example, we'd see this output:
如果我们使用与第一个示例类似的代码，我们会看到以下输出：


```
Calling factory method
First subscription
Doing some startup work...
1
2
3
Second subscription
Doing some startup work...
1
2
3
```

There are two important differences. First, the "Doing some startup work..." message does not appear until we first subscribe, illustrating that Defer has done what we wanted. However, notice that the message now appears twice: it will do this work each time we subscribe. If you want this deferred initialization but you'd also like once-only execution, you should look at the operators in the Publishing Operators chapter, which provide various ways to enable multiple subscribers to share a single subscription to an underlying source.
有两个重要的区别。首先， "Doing some startup work..." 消息直到我们第一次订阅才出现，说明 Defer 已经完成了我们想要的事情。但是，请注意该消息现在出现两次：每次我们订阅时它都会执行此工作。如果你想要这种延迟初始化，但又希望只执行一次，则应该查看“发布操作符”一章中的操作符，它提供了各种方法来使多个订阅者能够共享对底层源的单个订阅。

Sequence Generators 序列生成器
The creation methods we've looked at so far are straightforward in that they either produce very simple sequences (such as single-element, or empty sequences), or they rely on our code to tell them exactly what to produce. Now we'll look at some methods that can produce longer sequences.
到目前为止，我们所了解的创建方法都很简单，因为它们要么生成非常简单的序列（例如单元素或空序列），要么依赖我们的代码来准确地告诉它们要生成什么。现在我们将研究一些可以产生更长序列的方法。

Observable.Range 可观测范围
Observable.Range(int, int) returns an IObservable<int> that produces a range of integers. The first integer is the initial value and the second is the number of values to yield. This example will write the values '10' through to '24' and then complete.
Observable.Range(int, int) 返回一个生成一系列整数的 IObservable<int> 。第一个整数是初始值，第二个整数是要生成的值的数量。此示例将写入值“10”到“24”，然后完成。

IObservable<int> range = Observable.Range(10, 15);
range.Subscribe(Console.WriteLine, () => Console.WriteLine("Completed"));
Observable.Generate 可观察.生成
Suppose you wanted to emulate the Range factory method using Observable.Create. You might try this:
假设你想使用 Observable.Create 模拟 Range 工厂方法。你可以试试这个：

```C#
// Not the best way to do it!
IObservable<int> Range(int start, int count) =>
    Observable.Create<int>(observer =>
        {
            for (int i = 0; i < count; ++i)
            {
                observer.OnNext(start + i);
            }

            return Disposable.Empty;
        });
```

This will work, but it does not respect request to unsubscribe. That won't cause direct harm, because Rx detects unsubscription, and will simply ignore any further values we produce. However, it's a waste of CPU time (and therefore energy, with consequent battery lifetime and/or environmental impact) to carry on generating numbers after nobody is listening. How bad that is depends on how long a range was requested. But imagine you wanted an infinite sequence? Perhaps it's useful to you to have an IObservable<BigInteger> that produces value from the Fibonacci sequence, or prime numbers. How would you write that with Create? You'd certainly want some means of handling unsubscription in that case. We need our callback to return if we are to be notified of unsubscription (or we could supply an async method, but that doesn't really seem suitable here).
这可行，但不尊重取消订阅的请求。这不会造成直接伤害，因为 Rx 会检测到取消订阅，并且会忽略我们产生的任何进一步的值。然而，在没有人在听的情况下继续生成数字会浪费 CPU 时间（因此也会浪费能源，从而导致电池寿命和/或环境影响）。这有多糟糕取决于请求的范围有多长。但想象一下你想要一个无限序列吗？也许拥有一个从斐波那契数列或素数中生成值的 IObservable<BigInteger> 对你很有用。你会如何用 Create 来写它？在这种情况下，你肯定需要一些处理取消订阅的方法。如果我们要收到取消订阅的通知，我们需要回调返回（或者我们可以提供 async 方法，但这在这里似乎并不合适）。

There's a different approach that can work better here: Observable.Generate. The simple version of Observable.Generate takes the following parameters:
这里有一种效果更好的不同方法： Observable.Generate 。 Observable.Generate 的简单版本采用以下参数：

an initial state 初始状态
a predicate that defines when the sequence should terminate
定义序列何时应终止的谓词
a function to apply to the current state to produce the next state
应用于当前状态以产生下一个状态的函数
a function to transform the state to the desired output
将状态转换为所需输出的函数
public static IObservable<TResult> Generate<TState, TResult>(
    TState initialState, 
    Func<TState, bool> condition, 
    Func<TState, TState> iterate, 
    Func<TState, TResult> resultSelector)
This shows how you could use Observable.Generate to construct a Range method:
这显示了如何使用 Observable.Generate 构造 Range 方法：

```C#
// Example code only
public static IObservable<int> Range(int start, int count)
{
    int max = start + count;
    return Observable.Generate(
        start, 
        value => value < max, 
        value => value + 1, 
        value => value);
}
```

The Generate method calls us back repeatedly until either our condition callback says we're done, or the observer unsubscribes. We can define an infinite sequence simply by never saying we are done:
Generate 方法反复回调我们，直到我们的 condition 回调表明我们已完成，或者观察者取消订阅。我们可以简单地通过永远不说我们已经完成来定义一个无限序列：

```C#
IObservable<BigInteger> Fibonacci()
{
    return Observable.Generate(
        (v1: new BigInteger(1), v2: new BigInteger(1)),
        value => true, // It never ends!
        value => (value.v2, value.v1 + value.v2),
        value => value.v1); ;
}
```

Timed Sequence Generators
定时序列发生器
Most of the methods we've looked at so far have returned sequences that produce all of their values immediately. (The only exception is where we called Observable.Create and produced values when we were ready to.) However, Rx is able to generate sequences on a schedule.
到目前为止，我们看到的大多数方法都返回立即生成所有值的序列。 （唯一的例外是我们调用 Observable.Create 并在准备好时生成值。）但是，Rx 能够按计划生成序列。

As we'll see, operators that schedule their work do so through an abstraction called a scheduler. If you don't specify one, they will pick a default scheduler, but sometimes the timer mechanism is significant. For example, there are timers that integrate with UI frameworks, delivering notifications on the same thread that mouse clicks and other input are delivered on, and we might want Rx's time-based operators to use these. For testing purposes it can be useful to virtualize timings, so we can verify what happens in timing-sensitive code without necessarily waiting for tests to execute in real time.
正如我们将看到的，安排工作的操作员是通过称为调度程序的抽象来完成的。如果你不指定，他们会选择一个默认的调度程序，但有时计时器机制很重要。例如，有与 UI 框架集成的计时器，在传递鼠标单击和其他输入的同一线程上传递通知，我们可能希望 Rx 的基于时间的运算符使用这些。出于测试目的，虚拟化时序可能很有用，因此我们可以验证时序敏感代码中发生的情况，而不必等待测试实时执行。

Schedulers are a complex subject that is out of scope for this chapter, but they are covered in detail in the later chapter on Scheduling and threading.
调度程序是一个复杂的主题，超出了本章的范围，但稍后有关调度和线程的章节将详细介绍它们。

There are three ways of producing timed events.
产生定时事件的方法有3种。

Observable.Interval 可观察的间隔
The first is Observable.Interval(TimeSpan) which will publish incremental values starting from zero, based on a frequency of your choosing.
第一个是 Observable.Interval(TimeSpan) ，它将根据你选择的频率发布从零开始的增量值。

This example publishes values every 250 milliseconds.
此示例每 250 毫秒发布一次值。

```C#
IObservable<long> interval = Observable.Interval(TimeSpan.FromMilliseconds(250));
interval.Subscribe(
    Console.WriteLine, 
    () => Console.WriteLine("completed"));\
```

Output: 输出：

```
0
1
2
3
4
5
```

Once subscribed, you must dispose of your subscription to stop the sequence, because Interval returns an infinite sequence. Rx presumes that you might have considerable patience, because the sequences returned by Interval are of type IObservable<long> (long, not int) meaning you won't hit problems if you produce more than a paltry 2.1475 billion event (i.e. more than int.MaxValue).
订阅后，你必须处理订阅才能停止序列，因为 Interval 返回无限序列。 Rx 假设你可能有相当的耐心，因为 Interval 返回的序列是 IObservable<long> 类型（ long ，而不是 int ）如果你产生的事件超过微不足道的 21.475 亿个事件（即超过 int.MaxValue ），你就不会遇到问题。

Observable.Timer
The second factory method for producing constant time based sequences is Observable.Timer. It has several overloads. The most basic one takes just a TimeSpan as Observable.Interval does. But unlike Observable.Interval, Observable.Timer will publish exactly one value (the number 0) after the period of time has elapsed, and then it will complete.
用于生成基于恒定时间的序列的第二个工厂方法是 Observable.Timer 。它有几个重载。最基本的一个只需要一个 TimeSpan ，就像 Observable.Interval 一样。但与 Observable.Interval 不同的是， Observable.Timer 会在经过一段时间后只发布一个值（数字 0），然后完成。

```C#
var timer = Observable.Timer(TimeSpan.FromSeconds(1));
timer.Subscribe(
    Console.WriteLine, 
    () => Console.WriteLine("completed"));
```

Output: 输出：

```
0
completed
```

Alternatively, you can provide a DateTimeOffset for the dueTime parameter. This will produce the value 0 and complete at the specified time.
或者，你可以为 dueTime 参数提供 DateTimeOffset 。这将产生值 0 并在指定时间完成。

A further set of overloads adds a TimeSpan that indicates the period at which to produce subsequent values. This allows us to produce infinite sequences. It also shows how Observable.Interval is really just a special case of Observable.Timer. Interval could be implemented like this:
另一组重载添加了 TimeSpan ，指示生成后续值的周期。这使我们能够产生无限序列。它还显示了 Observable.Interval 实际上只是 Observable.Timer 的一个特例。 Interval 可以这样实现：

```C#
public static IObservable<long> Interval(TimeSpan period)
{
    return Observable.Timer(period, period);
}
```

While Observable.Interval will always wait the given period before producing the first value, this Observable.Timer overload gives the ability to start the sequence when you choose. With Observable.Timer you can write the following to have an interval sequence that starts immediately.
虽然 Observable.Interval 在生成第一个值之前始终会等待给定的时间，但此 Observable.Timer 重载使你能够在你选择时启动序列。使用 Observable.Timer 你可以编写以下代码以获得立即开始的间隔序列。

```C#
Observable.Timer(TimeSpan.Zero, period);
```

This takes us to our third way and most general way for producing timer related sequences, back to Observable.Generate.
这将我们带入第三种方法，也是生成计时器相关序列的最通用方法，回到 Observable.Generate 。

Timed Observable.Generate
定时 Observable.Generate
There's a more complex overload of Observable.Generate that allows you to provide a function that specifies the due time for the next value.
Observable.Generate 有一个更复杂的重载，它允许你提供一个函数来指定下一个值的到期时间。

```C#
public static IObservable<TResult> Generate<TState, TResult>(
    TState initialState, 
    Func<TState, bool> condition, 
    Func<TState, TState> iterate, 
    Func<TState, TResult> resultSelector, 
    Func<TState, TimeSpan> timeSelector)

```

The extra timeSelector argument lets us tell Generate when to produce the next item. We can use this to write our own implementation of Observable.Timer (and as you've already seen, this in turn enables us to write our own Observable.Interval).
额外的 timeSelector 参数让我们告诉 Generate 何时生成下一个项目。我们可以使用它来编写我们自己的 Observable.Timer 实现（正如你已经看到的，这反过来使我们能够编写我们自己的 Observable.Interval ）。

```C#
public static IObservable<long> Timer(TimeSpan dueTime)
{
    return Observable.Generate(
        0l,
        i => i < 1,
        i => i + 1,
        i => i,
        i => dueTime);
}

public static IObservable<long> Timer(TimeSpan dueTime, TimeSpan period)
{
    return Observable.Generate(
        0l,
        i => true,
        i => i + 1,
        i => i,
        i => i == 0 ? dueTime : period);
}

public static IObservable<long> Interval(TimeSpan period)
{
    return Observable.Generate(
        0l,
        i => true,
        i => i + 1,
        i => i,
        i => period);
}
```

This shows how you can use Observable.Generate to produce infinite sequences. I will leave it up to you the reader, as an exercise using Observable.Generate, to produce values at variable rates.
这展示了如何使用 Observable.Generate 生成无限序列。我将把它留给读者，作为使用 Observable.Generate 的练习，以可变速率生成值。

Observable sequences and state
可观察的序列和状态
As Observable.Generate makes particularly clear, observable sequences may need to maintain state. With that operator it is explicit—we pass in initial state, and we supply a callback to update it on each iteration. Plenty of other operators maintain internal state. The Timer remembers its tick count, and more subtly, has to somehow keep track of when it last raised an event and when the next one is due. And as you'll see in forthcoming chapters, plenty of other operators need to remember information about what they've already seen.
正如 Observable.Generate 特别清楚地表明的那样，可观察序列可能需要维护状态。对于该运算符，它是显式的 - 我们传递初始状态，并提供回调以在每次迭代时更新它。许多其他操作员维护内部状态。 Timer 会记住它的滴答计数，更巧妙的是，它必须以某种方式跟踪它上次引发事件的时间以及下一个事件到期的时间。正如你将在接下来的章节中看到的，许多其他操作员需要记住他们已经看到的信息。

This raises an interesting question: what happens if a process shuts down? Is there a way to preserve that state, and reconstitute it in a new process.
这就提出了一个有趣的问题：如果进程关闭会发生什么？有没有办法保留这种状态，并在新的过程中重建它。

With ordinary Rx.NET, the answer is no: all such state is held entirely in memory and there is no way to get hold of that state, or to ask running subscriptions to serialize their current state. This means that if you are dealing with particularly long-running operations you need to work out how you would restart and you can't rely on System.Reactive to help you. However, there is a related Rx-based set of libraries known collectively as the Reaqtive libraries. These provide implementations of most of the same operators as System.Reactive, but in a form where you can collect the current state, and recreate new subscriptions from previously preserved state. These libraries also include a component called Reaqtor, which is a hosting technology that can manage automatic checkpointing, and post-crash recovery, making it possible to support very long-running Rx logic, by making subscriptions persistent and reliable. Be aware that this is not currently in any productised form, so you will need to do a fair amount of work to use it, but if you need a persistable version of Rx, be aware that it exists.
对于普通的 Rx.NET，答案是否定的：所有此类状态都完全保存在内存中，并且无法获取该状态，或者要求正在运行的订阅序列化其当前状态。这意味着，如果你正在处理特别长时间运行的操作，你需要弄清楚如何重新启动，并且不能依赖 System.Reactive 来帮助你。然而，有一组相关的基于 Rx 的库，统称为 Reaqtive 库。它们提供了与 System.Reactive 相同的大多数运算符的实现，但采用的形式可以收集当前状态，并从以前保留的状态重新创建新的订阅。这些库还包括一个名为 Reaqtor 的组件，这是一种托管技术，可以管理自动检查点和崩溃后恢复，通过使订阅持久且可靠，从而可以支持非常长时间运行的 Rx 逻辑。请注意，目前这还没有任何产品化形式，因此你需要做大量的工作才能使用它，但如果你需要 Rx 的持久版本，请注意它的存在。

Adapting Common Types to IObservable<T>
将常见类型调整为 IObservable<T>
Although we've now seen two very general ways to produce arbitrary sequences—Create and Generate—what if you already have an existing source of information in some other form that you'd like to make available as an IObservable<T>? Rx provides a few adapters for common source types.
尽管我们现在已经看到了两种非常通用的生成任意序列的方法 - Create 和 Generate - 如果你已经拥有你想要的其他形式的现有信息源，该怎么办以 IObservable<T> 形式提供？ Rx 为常见源类型提供了一些适配器。

From delegates 来自代表们
The Observable.Start method allows you to turn a long running Func<T> or Action into a single value observable sequence. By default, the processing will be done asynchronously on a ThreadPool thread. If the overload you use is a Func<T> then the return type will be IObservable<T>. When the function returns its value, that value will be published and then the sequence completed. If you use the overload that takes an Action, then the returned sequence will be of type IObservable<Unit>. The Unit type represents the absence of information, so it's somewhat analogous to void, except you can have an instance of the Unit type. It's particularly useful in Rx because we often care only about when something has happened, and there might not be any information besides timing. In these cases, we often use an IObservable<Unit> so that it's possible to produce definite events even though there's no meaningful data in them. (The name comes from the world of functional programming, where this kind of construct is used a lot.) In this case, Unit is used to publish an acknowledgement that the Action is complete, because an Action does not return any information. The Unit type itself has no value; it just serves as an empty payload for the OnNext notification. Below is an example of using both overloads.
Observable.Start 方法允许你将长时间运行的 Func<T> 或 Action 转换为单个值可观察序列。默认情况下，处理将在 ThreadPool 线程上异步完成。如果你使用的重载是 Func<T> ，那么返回类型将为 IObservable<T> 。当函数返回其值时，该值将被发布，然后序列完成。如果你使用采用 Action 的重载，则返回的序列将为 IObservable<Unit> 类型。 Unit 类型表示缺少信息，因此它有点类似于 void ，只不过你可以拥有 Unit 类型的实例。它在 Rx 中特别有用，因为我们通常只关心某件事何时发生，并且除了时间之外可能没有任何信息。在这些情况下，我们经常使用 IObservable<Unit> ，这样即使其中没​​有有意义的数据，也可以生成明确的事件。 （该名称来自函数式编程领域，这种构造被大量使用。）在本例中， Unit 用于发布 Action 已完成的确认，因为 Action 不返回任何信息。 Unit 类型本身没有值；它只是作为 OnNext 通知的空负载。下面是使用这两个重载的示例。

```C#
static void StartAction()
{
    var start = Observable.Start(() =>
        {
            Console.Write("Working away");
            for (int i = 0; i < 10; i++)
            {
                Thread.Sleep(100);
                Console.Write(".");
            }
        });

    start.Subscribe(
        unit => Console.WriteLine("Unit published"), 
        () => Console.WriteLine("Action completed"));
}

static void StartFunc()
{
    var start = Observable.Start(() =>
    {
        Console.Write("Working away");
        for (int i = 0; i < 10; i++)
        {
            Thread.Sleep(100);
            Console.Write(".");
        }
        return "Published value";
    });

    start.Subscribe(
        Console.WriteLine, 
        () => Console.WriteLine("Action completed"));
}
```

Note the difference between Observable.Start and Observable.Return. The Start method invokes our callback only upon subscription, so it is an example of a 'lazy' operation. Conversely, Return requires us to supply the value up front.
请注意 Observable.Start 和 Observable.Return 之间的区别。 Start 方法仅在订阅时调用我们的回调，因此它是“惰性”操作的示例。相反， Return 要求我们预先提供该值。

The observable returned by Start may seem to have a superficial resemblance to Task or Task<T> (depending on whether you use the Action or Func<T> overload). Each represents work that may take some time before eventually completing, perhaps producing a result. However, there's a significant difference: Start doesn't begin the work until you subscribe to it. Moreover, it will re-execute the callback every time you subscribe to it. So it is more like a factory for a task-like entity.
Start 返回的可观察值可能看起来与 Task 或 Task<T> 表面上相似（取决于你是否使用 Action 或 < b4> 过载）。每一项工作都可能需要一些时间才能最终完成，也许会产生结果。但是，有一个显着的区别： Start 在你订阅之前不会开始工作。而且，每次订阅时它都会重新执行回调。所以它更像是一个类似任务实体的工厂。

From events 来自事件
As we discussed early in the book, .NET has a model for events that is baked into its type system. This predates Rx (not least because Rx wasn't feasible until .NET got generics in .NET 2.0) so it's common for types to support events but not Rx. To be able to integrate with the existing event model, Rx provides methods to take an event and turn it into an observable sequence. I showed this briefly in the file system watcher example earlier, but let's examine this in a bit more detail. There are several different varieties you can use. This show the most succinct form:
正如我们在本书前面讨论的那样，.NET 有一个内置于其类型系统中的事件模型。这早于 Rx（尤其是因为在 .NET 在 .NET 2.0 中获得泛型之前 Rx 不可行），因此类型支持事件但不支持 Rx 是很常见的。为了能够与现有的事件模型集成，Rx 提供了获取事件并将其转换为可观察序列的方法。我之前在文件系统观察器示例中简要展示了这一点，但让我们更详细地研究一下这一点。你可以使用多种不同的品种。这显示了最简洁的形式：

FileSystemWatcher watcher = new (@"c:\incoming");
IObservable<EventPattern<FileSystemEventArgs>> changeEvents = Observable
    .FromEventPattern<FileSystemEventArgs>(watcher, nameof(watcher.Changed));
If you have an object that provides an event, you can use this overload of FromEventPattern, passing in the object and the name of the event that you'd like to use with Rx. Although this is the simplest way to adapt events into Rx's world, it has a few problems.
如果你有一个提供事件的对象，则可以使用 FromEventPattern 的此重载，传入该对象以及你想要与 Rx 一起使用的事件的名称。尽管这是将事件适应 Rx 世界的最简单方法，但它存在一些问题。

Firstly, why do I need to pass the event name as a string? Identifying members with strings is an error-prone technique. The compiler won't notice if there's a mismatch between the first and second argument (e.g., if I passed the arguments (somethingElse, nameof(watcher.Changed)) by mistake). Couldn't I just pass watcher.Changed itself? Unfortunately not—this is an example of the issue I mentioned in the first chapter: .NET events are not first class citizens. We can't use them in the way we can use other objects or values. For example, we can't pass an event as an argument to a method. In fact the only thing you can do with a .NET event is attach and remove event handlers. If I want to get some other method to attach handlers to the event of my choosing (e.g., here I want Rx to handle the events), then the only way to do that is to specify the event's name so that the method (FromEventPattern) can then use reflection to attach its own handlers.
首先，为什么我需要将事件名称作为字符串传递？使用字符串标识成员是一种容易出错的技术。编译器不会注意到第一个参数和第二个参数之间是否不匹配（例如，如果我错误地传递了参数 (somethingElse, nameof(watcher.Changed)) ）。我不能直接传递 watcher.Changed 本身吗？不幸的是不是——这是我在第一章中提到的问题的一个例子：.NET 事件不是一等公民。我们不能像使用其他对象或值那样使用它们。例如，我们不能将事件作为参数传递给方法。事实上，你可以对 .NET 事件执行的唯一操作是附加和删除事件处理程序。如果我想获得一些其他方法来将处理程序附加到我选择的事件（例如，这里我希望 Rx 处理事件），那么唯一的方法就是指定事件的名称，以便方法 ( FromEventPattern ）然后可以使用反射来附加它自己的处理程序。

This is a problem for some deployment scenarios. It is increasingly common in .NET to do extra work at build time to optimize runtime behaviour, and reliance on reflection can compromise these techniques. For example, instead of relying on Just In Time (JIT) compilation of code, we might use Ahead of Time (AOT) mechanisms. .NET's Ready to Run (R2R) system enables you to include pre-compiled code targeting specific CPU types alongside the normal IL, avoiding having to wait for .NET to compile the IL into runnable code. This can have a significant effect on startup times. In client side applications, it can fix problems where applications are sluggish when they first start up. It can also be important in server-side applications, especially in environments where code may be moved from one compute node to another fairly frequently, making it important to minimize cold start costs. There are also scenarios where JIT compilation is not even an option, in which case AOT compilation isn't merely an optimization: it's the only means by which code can run at all.
对于某些部署场景来说这是一个问题。在 .NET 中，在构建时做额外的工作来优化运行时行为越来越常见，而对反射的依赖可能会损害这些技术。例如，我们可以使用提前（AOT）机制，而不是依赖即时（JIT）代码编译。 .NET 的就绪运行 (R2R) 系统使你能够将针对特定 CPU 类型的预编译代码与普通 IL 一起包含在内，从而避免等待 .NET 将 IL 编译为可运行代码。这会对启动时间产生重大影响。在客户端应用程序中，它可以解决应用程序首次启动时缓慢的问题。它在服务器端应用程序中也很重要，特别是在代码可能相当频繁地从一个计算节点移动到另一个计算节点的环境中，因此最小化冷启动成本非常重要。还有一些场景甚至无法选择 JIT 编译，在这种情况下，AOT 编译不仅仅是一种优化：它是代码运行的唯一方式。

The problem with reflection is that it makes it difficult for the build tools to work out what code will execute at runtime. When they inspect this call to FromEventPattern they will just see arguments of type object and string. It's not self-evident that this is going to result in reflection-driven calls to the add and remove methods for FileSystemWatcher.Changed at runtime. There are attributes that can be used to provide hints, but there are limits to how well these can work. Sometimes the build tools will be unable to determine what code would need to be AOT compiled to enable this method to execute without relying on runtime JIT.
反射的问题在于，它使构建工具很难确定在运行时将执行哪些代码。当他们检查对 FromEventPattern 的调用时，他们只会看到 object 和 string 类型的参数。不言而喻，这将导致在运行时对 FileSystemWatcher.Changed 的 add 和 remove 方法进行反射驱动的调用。有些属性可用于提供提示，但这些属性的效果有限。有时，构建工具将无法确定需要 AOT 编译哪些代码才能使该方法在不依赖运行时 JIT 的情况下执行。

There's another, related problem. The .NET build tools support a feature called 'trimming', in which they remove unused code. The System.Reactive.dll file is about 1.3MB in size, but it would be a very unusual application that used every member of every type in that component. Basic use of Rx might need only a few tens of kilobytes. The idea with trimming is to work out which bits are actually in use, and produce a copy of the DLL that contains only that code. This can dramatically reduce the volume of code that needs to be deployed for an executable to run. This can be especially important in client-side Blazor applications, where .NET components end up being downloaded by the browser. Having to download an entire 1.3MB component might make you think twice about using it. But if trimming means that basic usage requires only a few tens of KB, and that the size would increase only if you were making more extensive use of the component, that can make it reasonable to use a component that would, without trimming, have imposed too large a penalty to justify its inclusion. But as with AOT compilation, trimming can only work if the tools can determine which code is in use. If they can't do that, it's not just a case of falling back to a slower path, waiting while the relevant code gets JIT compiler. If code has been trimmed, it will be unavailable at runtime, and your application might crash with a MissingMethodException.
还有另一个相关的问题。 .NET 构建工具支持一种称为“修剪”的功能，可以删除未使用的代码。 System.Reactive.dll 文件大小约为 1.3MB，但它是一个非常不寻常的应用程序，它使用了该组件中每种类型的每个成员。 Rx 的基本使用可能只需要几十千字节。修剪的想法是找出实际使用的位，并生成仅包含该代码的 DLL 副本。这可以显着减少可执行文件运行所需部署的代码量。这在客户端 Blazor 应用程序中尤其重要，其中 .NET 组件最终由浏览器下载。必须下载整个 1.3MB 组件可能会让你在使用它时三思而后行。但是，如果修剪意味着基本使用只需要几十 KB，并且只有更广泛地使用该组件，大小才会增加，那么使用一个在不进行修剪的情况下会强加的组件是合理的。处罚太大，不足以证明其纳入的合理性。但与 AOT 编译一样，只有当工具可以确定正在使用哪些代码时，修剪才能起作用。如果他们做不到这一点，那么这不仅仅是退回到较慢路径、等待相关代码获得 JIT 编译器的情况。如果代码已被修剪，它将在运行时不可用，并且你的应用程序可能会崩溃并显示 MissingMethodException 。

So reflection-based APIs can be problematic if you're using any of these techniques. Fortunately, there's an alternative. We can use an overload that takes a couple of delegates, and Rx will invoke these when it wants to add or remove handlers for the event:
因此，如果你使用这些技术中的任何一种，基于反射的 API 可能会出现问题。幸运的是，还有一个替代方案。我们可以使用一个带有几个委托的重载，当 Rx 想要添加或删除事件的处理程序时，它会调用这些委托：

```C#
IObservable<EventPattern<FileSystemEventArgs>> changeEvents = Observable
    .FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
        h => watcher.Changed += h,
        h => watcher.Changed -= h);
```

This is code that AOT and trimming tools can understand easily. We've written methods that explicitly add and remove handlers for the FileSystemWatcher.Changed event, so AOT tools can pre-compile those two methods, and trimming tools know that they cannot remove the add and remove handlers for those events.
这是 AOT 和修剪工具可以轻松理解的代码。我们编写了显式添加和删除 FileSystemWatcher.Changed 事件的处理程序的方法，因此 AOT 工具可以预编译这两个方法，并且修剪工具知道它们无法删除这些事件的添加和删除处理程序。

The downside is that this is a pretty cumbersome bit of code to write. If you've not already bought into the idea of using Rx, this might well be enough to make you think "I'll just stick with ordinary .NET events, thanks. But the cumbersome nature is a symptom of what is wrong with .NET events. We wouldn't have had to write anything so ugly if events had been first class citizens in the first place.
缺点是编写这样的代码相当麻烦。如果你还没有接受使用 Rx 的想法，这可能足以让你想“我将继续使用普通的 .NET 事件，谢谢。但是繁琐的本质是 .NET 问题的症状。 NET 事件。如果事件一开始就是一等公民，我们就不必编写如此丑陋的东西。

Not only has that second-class status meant we couldn't just pass the event itself as an argument, it has also meant that we've had to state type arguments explicitly. The relationship between an event's delegate type (FileSystemEventHandler in this example) and its event argument type (FileSystemEventArgs here) is, in general, not something that C#'s type inference can determine automatically, which is why we've had to specify both types explicitly. (Events that use the generic EventHandler<T> type are more amenable to type inference, and can use a slightly less verbose version of FromEventPattern. Unfortunately, relatively few events actually use that. Some events provide no information besides the fact that something just happened, and use the base EventHandler type, and for those kinds of events, you can in fact omit the type arguments completely, making the code slightly less ugly. You still need to provide the add and remove callbacks though.)
二等状态不仅意味着我们不能只将事件本身作为参数传递，还意味着我们必须显式地声明类型参数。一般来说，事件的委托类型（本例中为 FileSystemEventHandler ）与其事件参数类型（此处为 FileSystemEventArgs ）之间的关系并不是 C# 的类型推断可以自动确定的。这就是为什么我们必须明确指定这两种类型。 （使用通用 EventHandler<T> 类型的事件更适合类型推断，并且可以使用稍微简洁的 FromEventPattern 版本。不幸的是，实际使用它的事件相对较少。某些事件提供除了刚刚发生的事情之外没有任何信息，并使用基本 EventHandler 类型，对于这些类型的事件，你实际上可以完全省略类型参数，使代码稍微不那么难看。你仍然需要提供添加和删除回调。）

Notice that the return type of FromEventPattern in this example is:
请注意，此示例中 FromEventPattern 的返回类型为：

```C#
IObservable<EventPattern<FileSystemEventArgs>>.
```

The EventPattern<T> type encapsulates the information that the event passes to handlers. Most .NET events follow a common pattern in which handler methods take two arguments: an object sender, which just tells you which object raised the event (useful if you attach one event handler to multiple objects) and then a second argument of some type derived from EventArgs that provides information about the event. EventPattern<T> just packages these two arguments into a single object that offers Sender and EventArgs properties. In cases where you don't in fact want to attach one handler to multiple sources, you only really need that EventArgs property, which is why the earlier FileSystemWatcher examples went on to extract just that, to get a simpler result of type IObservable<FileSystemEventArgs>. It did this with the Select operator, which we'll get to in more detail later:
EventPattern<T> 类型封装了事件传递给处理程序的信息。大多数 .NET 事件都遵循一种常见模式，其中处理程序方法采用两个参数：一个 object sender ，它仅告诉你哪个对象引发了事件（如果你将一个事件处理程序附加到多个对象，则很有用），然后是第二个参数从 EventArgs 派生的某种类型的参数，提供有关事件的信息。 EventPattern<T> 只是将这两个参数打包到一个提供 Sender 和 EventArgs 属性的对象中。如果你实际上不想将一个处理程序附加到多个源，则你实际上只需要 EventArgs 属性，这就是为什么前面的 FileSystemWatcher 示例继续提取，以获得 IObservable<FileSystemEventArgs> 类型的更简单的结果。它使用 Select 运算符来完成此操作，我们稍后将详细介绍：

```C#
IObservable<FileSystemEventArgs> changes = changeEvents.Select(ep => ep.EventArgs);
```

It is very common to want to expose property changed events as observable sequences. The .NET runtime libraries define a .NET-event-based interface for advertising property changes, INotifyPropertyChanged, and some user interface frameworks have more specialized systems for this, such as WPF's DependencyProperty. If you are contemplating writing your own wrappers to do this sort of thing, I would strongly suggest looking at the Reactive UI libraries first. It has a set of features for wrapping properties as IObservable<T>.
希望将属性更改事件公开为可观察序列是很常见的。 .NET 运行时库定义了一个基于 .NET 事件的接口，用于通告属性更改 INotifyPropertyChanged ，并且某些用户界面框架为此拥有更专门的系统，例如 WPF 的 DependencyProperty 。如果你正在考虑编写自己的包装器来执行此类操作，我强烈建议你首先查看 Reactive UI 库。它具有一组将属性包装为 IObservable<T> 的功能。

From Task 来自任务
The Task and Task<T> types are very widely used in .NET. Mainstream .NET languages have built-in support for working with them (e.g., C#'s async and await keywords). There's some conceptual overlap between tasks and IObservable<T>: both represent some sort of work that might take a while to complete. There is a sense in which an IObservable<T> is a generalization of a Task<T>: both represent potentially long-running work, but an IObservable<T> can produce multiple results whereas Task<T> can produce just one.
Task 和 Task<T> 类型在 .NET 中使用非常广泛。主流 .NET 语言具有对其使用的内置支持（例如，C# 的 async 和 await 关键字）。任务和 IObservable<T> 之间存在一些概念上的重叠：两者都代表某种可能需要一段时间才能完成的工作。从某种意义上说， IObservable<T> 是 Task<T> 的泛化：两者都代表潜在的长时间运行的工作，但 IObservable<T> 可以产生多个结果，而 < b8> 只能产生一个。

Since IObservable<T> is the more general abstraction, we should be able to represent a Task<T> as an IObservable<T>. Rx defines various extension methods for Task and Task<T> to do this. These methods are all called ToObservable(), and it offers various overloads offering control of the details where required, and simplicity for the most common scenarios.
由于 IObservable<T> 是更通用的抽象，因此我们应该能够将 Task<T> 表示为 IObservable<T> 。 Rx 为 Task 和 Task<T> 定义了各种扩展方法来执行此操作。这些方法都称为 ToObservable() ，它提供了各种重载，可以在需要时控制细节，并为最常见的场景提供简单性。

Although they are conceptually similar, Task<T> does a few things differently in the details. For example, you can retrieve its Status property, which might report that it is in a cancelled or faulted state. IObservable<T> doesn't provide a way to ask a source for its state; it just tells you things. So ToObservable makes some decisions about how to present status in a way that makes makes sense in an Rx world:
尽管它们在概念上相似，但 Task<T> 在细节上做了一些不同的事情。例如，你可以检索其 Status 属性，该属性可能会报告其处于已取消或故障状态。 IObservable<T> 没有提供向源询问其状态的方法；它只是告诉你一些事情。因此， ToObservable 做出了一些关于如何以在 Rx 世界中有意义的方式呈现状态的决定：

if the task is Cancelled, IObservable<T> invokes a subscriber's OnError passing a TaskCanceledException
如果任务被取消， IObservable<T> 调用订阅者的 OnError 并传递 TaskCanceledException
if the task is Faulted IObservable<T> invokes a subscriber's OnError passing the task's inner exception
如果任务发生故障 IObservable<T> 调用订阅者的 OnError 传递任务的内部异常
if the task is not yet in a final state (neither Cancelled, Faulted, or RanToCompletion), the IObservable<T> will not produce any notifications until such time as the task does enter one of these final states
如果任务尚未处于最终状态（既不是 Cancelled、Faulted 也不是 RanToCompletion），则 IObservable<T> 将不会生成任何通知，直到任务进入这些最终状态之一
It does not matter whether the task is already in a final state at the moment that you call ToObservable. If it has finished, ToObservable will just return a sequence representing that state. (In fact, it uses either the Return or Throw creation methods you saw earlier.) If the task has not yet finished, ToObservable will attach a continuation to the task to detect the outcome once it does complete.
在你调用 ToObservable 时任务是否已处于最终状态并不重要。如果完成， ToObservable 将仅返回表示该状态的序列。 （事实上​​，它使用你之前看到的 Return 或 Throw 创建方法。）如果任务尚未完成， ToObservable 将附加一个延续到任务完成后检测结果。

Tasks come in two forms: Task<T>, which produces a result, and Task, which does not. But in Rx, there is only IObservable<T>—there isn't a no-result form. We've already seen this problem once before, when the Observable.Start method needed to be able to adapt a delegate as an IObservable<T> even when the delegate was an Action that produced no result. The solution was to return an IObservable<Unit>, and that's also exactly what you get when you call ToObservable on a plain Task.
任务有两种形式： Task<T> ，它产生结果； Task ，它不产生结果。但在 Rx 中，只有 IObservable<T> — 没有无结果形式。我们之前已经遇到过这个问题，当时 Observable.Start 方法需要能够将委托调整为 IObservable<T> ，即使委托是 Action 这没有产生任何结果。解决方案是返回一个 IObservable<Unit> ，这也正是你在普通 Task 上调用 ToObservable 时得到的结果。

The extension method is simple to use:
扩展方法使用起来很简单：

```C#
Task<string> t = Task.Run(() =>
{
    Console.WriteLine("Task running...");
    return "Test";
});
IObservable<string> source = t.ToObservable();
source.Subscribe(
    Console.WriteLine,
    () => Console.WriteLine("completed"));
source.Subscribe(
    Console.WriteLine,
    () => Console.WriteLine("completed"));
```

Here's the output. 这是输出。

```
Task running...
Test
completed
Test
completed
```

Notice that even with two subscribers, the task runs only once. That shouldn't be surprising since we only created a single task. If the task has not yet finished, then all subscribers will receive the result when it does. If the task has finished, the IObservable<T> effectively becomes a single-value cold observable.
请注意，即使有两个订阅者，该任务也仅运行一次。这应该不足为奇，因为我们只创建了一个任务。如果任务尚未完成，那么所有订阅者都会在任务完成时收到结果。如果任务完成， IObservable<T> 实际上成为单值冷可观察值。

One Task per subscription
每个订阅一个任务
There's a different way to get an IObservable<T> for a source. I can replace the first statement in the preceding example with this:
有一种不同的方法来获取源的 IObservable<T> 。我可以将前面示例中的第一条语句替换为：

```C#
IObservable<string> source = Observable.FromAsync(() => Task.Run(() =>
{
    Console.WriteLine("Task running...");
    return "Test";
}));
```

Subscribing twice to this produces slightly different output:
订阅两次会产生略有不同的输出：

```
Task running...
Task running...
Test
Test
completed
completed
```

Notice that this executes the task twice, once for each call to Subscribe. FromAsync can do this because instead of passing a Task<T> we pass a callback that returns a Task<T>. It calls that when we call Subscribe, so each subscriber essentially gets their own task.
请注意，这会执行任务两次，每次调用 Subscribe 一次。 FromAsync 可以做到这一点，因为我们传递一个返回 Task<T> 的回调，而不是传递 Task<T> 。当我们调用 Subscribe 时，它会调用它，因此每个订阅者本质上都会获得自己的任务。

If I want to use async and await to define my task, then I don't need to bother with the Task.Run because an async lambda creates a Func<Task<T>>, which is exactly the type FromAsync wants:
如果我想使用 async 和 await 来定义我的任务，那么我不需要担心 Task.Run 因为 async lambda 创建一个 Func<Task<T>> ，这正是 FromAsync 想要的类型：

```
IObservable<string> source = Observable.FromAsync(async () =>
{
    Console.WriteLine("Task running...");
    await Task.Delay(50);
    return "Test";
});
```

This produces exactly the same output as before. There is a subtle difference with this though. When I used Task.Run the lambda ran on a task pool thread from the start. But when I write it this way, the lambda will begin to run on whatever thread calls Subscribe. It's only when it hits the first await that it returns (and the call to Subscribe will then return), with the remainder of the method running on the thread pool.
这会产生与之前完全相同的输出。但这有一个微妙的区别。当我使用 Task.Run 时，lambda 从一开始就在任务池线程上运行。但是当我这样写时，lambda 将开始在调用 Subscribe 的任何线程上运行。仅当它到达第一个 await 时才会返回（然后对 Subscribe 的调用将返回），方法的其余部分在线程池上运行。

From IEnumerable<T> 来自 IEnumerable<T>
Rx defines another extension method called ToObservable, this time for IEnumerable<T>. In earlier chapters I described how IObservable<T> was designed to represent the same basic abstraction as IEnumerable<T>, with the only difference being the mechanism we use to obtain the elements in the sequence: with IEnumerable<T>, we write code that pulls values out of the collection (e.g., a foreach loop), whereas IObservable<T> pushes values to us by invoking OnNext on our IObserver<T>.
Rx 定义了另一个名为 ToObservable 的扩展方法，这次是针对 IEnumerable<T> 的。在前面的章节中，我描述了 IObservable<T> 是如何设计来表示与 IEnumerable<T> 相同的基本抽象，唯一的区别是我们用来获取序列中元素的机制：使用 IEnumerable<T> ，我们编写从集合中提取值的代码（例如 foreach 循环），而 IObservable<T> 通过调用 OnNext 将值推送给我们> 在我们的 IObserver<T> 上。

We could write code that bridges from pull to push:
我们可以编写从拉到推的桥梁：

```C#
// Example code only - do not use!
public static IObservable<T> ToObservableOversimplified<T>(this IEnumerable<T> source)
{
    return Observable.Create<T>(o =>
    {
        foreach (var item in source)
        {
            o.OnNext(item);
        }

        o.OnComplete();

        // Incorrectly ignoring unsubscription.
        return Disposable.Empty;
    });
}
```

This crude implementation conveys the basic idea, but it is naive. It does not attempt to handle unsubscription, and it's not easy to fix that when using Observable.Create for this particular scenario. And as we will see later in the book, Rx sources that might try to deliver large numbers of events in quick succession should integrate with Rx's concurrency model. The implementation that Rx supplies does of course cater for all of these tricky details. That makes it rather more complex, but that's Rx's problem; you can think of it as being logically equivalent to the code shown above, but without the shortcomings.
这种粗略的实现传达了基本思想，但它很幼稚。它不会尝试处理取消订阅，并且在针对此特定场景使用 Observable.Create 时修复该问题并不容易。正如我们将在本书后面看到的，可能尝试快速连续传递大量事件的 Rx 源应该与 Rx 的并发模型集成。 Rx 提供的实现当然可以满足所有这些棘手的细节。这使得它变得更加复杂，但这就是 Rx 的问题；你可以认为它在逻辑上与上面所示的代码等效，但没有缺点。

In fact this is a recurring theme throughout Rx.NET. Many of the built-in operators are useful not because they do something particularly complicated, but because they deal with many subtle and tricky issues for you. You should always try to find something built into Rx.NET that does what you need before considering rolling your own solution.
事实上，这是整个 Rx.NET 中反复出现的主题。许多内置运算符很有用，不是因为它们做了特别复杂的事情，而是因为它们为你处理了许多微妙而棘手的问题。在考虑推出自己的解决方案之前，你应该始终尝试找到 Rx.NET 中内置的可以满足你需求的东西。

When transitioning from IEnumerable<T> to IObservable<T>, you should carefully consider what you are really trying to achieve. Consider that the blocking synchronous (pull) nature of IEnumerable<T> does always not mix well with the asynchronous (push) nature of IObservable<T>. As soon as something subscribes to an IObservable<T> created in this way, it is effectively asking to iterate over the IEnumerable<T>, immediately producing all of the values. The call to Subscribe might not return until it has reached the end of the IEnumerable<T>, making it similar to the very simple example shown at the start of this chapter. (I say "might" because as we'll see when we get to schedulers, the exact behaviour depends on the context.) ToObservable can't work magic—something somewhere has to execute what amounts to a foreach loop.
从 IEnumerable<T> 转换到 IObservable<T> 时，你应该仔细考虑你真正想要实现的目标。请考虑 IEnumerable<T> 的阻塞同步（拉）性质始终无法与 IObservable<T> 的异步（推）性质很好地混合。一旦订阅了以这种方式创建的 IObservable<T> ，它就会有效地要求迭代 IEnumerable<T> ，立即生成所有值。对 Subscribe 的调用可能不会返回，直到到达 IEnumerable<T> 的末尾，这使得它类似于本章开头所示的非常简单的示例。 （我说“可能”是因为当我们到达调度程序时我们会看到，确切的行为取决于上下文。） ToObservable 无法发挥魔力 - 某些地方必须执行相当于 < b9> 循环。

So although this can be a convenient way to bring sequences of data into an Rx world, you should carefully test and measure the performance impact.
因此，尽管这可能是一种将数据序列带入 Rx 世界的便捷方法，但你应该仔细测试和衡量性能影响。

From APM 来自 APM
Rx provides support for the ancient .NET Asynchronous Programming Model (APM). Back in .NET 1.0, this was the only pattern for representing asynchronous operations. It was superseded in 2010 when .NET 4.0 introduced the Task-based Asynchronous Pattern (TAP). The old APM offers no benefits over the TAP. Moreover, C#'s async and await keywords (and equivalents in other .NET languages) only support the TAP, meaning that the APM is best avoided. However, the TAP was fairly new back in 2011 when Rx 1.0 was released, so it offered adapters for presenting an APM implementation as an IObservable<T>.
Rx 为古老的 .NET 异步编程模型 (APM) 提供支持。回到 .NET 1.0，这是表示异步操作的唯一模式。它于 2010 年被 .NET 4.0 引入基于任务的异步模式 (TAP) 所取代。与 TAP 相比，旧的 APM 没有任何优势。此外，C# 的 async 和 await 关键字（以及其他 .NET 语言中的等效关键字）仅支持 TAP，这意味着最好避免使用 APM。然而，早在 2011 年 Rx 1.0 发布时，TAP 就相当新了，因此它提供了用于将 APM 实现呈现为 IObservable<T> 的适配器。

Nobody should be using the APM today, but for completeness (and just in case you have to use an ancient library that only offers the APM) I will provide a very brief explanation of Rx's support for it.
今天没有人应该使用 APM，但为了完整性（以防万一你必须使用仅提供 APM 的古老库），我将提供 Rx 对它的支持的非常简短的解释。

The result of the call to Observable.FromAsyncPattern does not return an observable sequence. It returns a delegate that returns an observable sequence. (So it is essentially a factory factory) The signature for this delegate will match the generic arguments of the call to FromAsyncPattern, except that the return type will be wrapped in an observable sequence. The following example wraps the Stream class's BeginRead/EndRead methods (which are an implementation of the APM).
调用 Observable.FromAsyncPattern 的结果不返回可观察的序列。它返回一个返回可观察序列的委托。 （因此它本质上是一个工厂工厂）此委托的签名将与调用 FromAsyncPattern 的通用参数匹配，但返回类型将包装在可观察序列中。以下示例包装 Stream 类的 BeginRead / EndRead 方法（它们是 APM 的实现）。

Note: this is purely to illustrate how to wrap the APM. You would never do this in practice because Stream has supported the TAP for years.
注意：这纯粹是为了说明如何包装APM。实际上你永远不会这样做，因为 Stream 多年来一直支持 TAP。

```C#
Stream stream = GetStreamFromSomewhere();
var fileLength = (int) stream.Length;

Func<byte[], int, int, IObservable<int>> read = 
            Observable.FromAsyncPattern<byte[], int, int, int>(
              stream.BeginRead, 
              stream.EndRead);
var buffer = new byte[fileLength];
IObservable<int> bytesReadStream = read(buffer, 0, fileLength);
bytesReadStream.Subscribe(byteCount =>
{
    Console.WriteLine(
        "Number of bytes read={0}, buffer should be populated with data now.", 
        byteCount);
});
```

Subjects 科目
So far, this chapter has explored various factory methods that return IObservable<T> implementations. There is another way though: System.Reactive defines various types that implement IObservable<T> that we can instantiate directly. But how do we determine what values these types produce? We're able to do that because they also implement IObserver<T>, enabling us to push values into them, and those very same values we push in will be the ones seen by observers.
到目前为止，本章已经探讨了返回 IObservable<T> 实现的各种工厂方法。还有另一种方法： System.Reactive 定义了实现 IObservable<T> 的各种类型，我们可以直接实例化它们。但是我们如何确定这些类型产生什么值？我们之所以能够做到这一点，是因为它们还实现了 IObserver<T> ，使我们能够将值推入其中，而我们推入的那些完全相同的值将被观察者看到。

Types that implement both IObservable<T> and IObserver<T> are called subjects in Rx. There's an an ISubject<T> to represent this. (This is in the System.Reactive NuGet package, unlike IObservable<T> and IObserver<T>, which are both built into the .NET runtime libraries.) ISubject<T> looks like this:
同时实现 IObservable<T> 和 IObserver<T> 的类型在 Rx 中称为主题。有一个 ISubject<T> 来表示这一点。 （它位于 System.Reactive NuGet 包中，与 IObservable<T> 和 IObserver<T> 不同，它们都内置于 .NET 运行时库中。） ISubject<T> 看起来像这样：

```C#
public interface ISubject<T> : ISubject<T, T>
{
}

```
So it turns out there's also a two-argument ISubject<TSource, TResult> to accommodate the fact that something that is both an observer and an observable might transform the data that flows through it in some way, meaning that the input and output types are not necessarily the same. Here's the two-type-argument definition:
因此，事实证明还有一个双参数 ISubject<TSource, TResult> 来适应这样一个事实：既是观察者又是可观察者的东西可能会以某种方式转换流经它的数据，这意味着输入和输出类型不一定相同。这是两种类型参数的定义：

```C#
public interface ISubject<in TSource, out TResult> : IObserver<TSource>, IObservable<TResult>
{
}
```

As you can see the ISubject interfaces don't define any members of their own. They just inherit from IObserver<T> and IObservable<T>—these interfaces are nothing more than a direct expression of the fact that a subject is both an observer and an observable.
正如你所看到的， ISubject 接口没有定义自己的任何成员。它们只是继承自 IObserver<T> 和 IObservable<T> ——这些接口只不过是主体既是观察者又是可观察者这一事实的直接表达。

But what is this for? You can think of IObserver<T> and the IObservable<T> as the 'consumer' and 'publisher' interfaces respectively. A subject, then is both a consumer and a publisher. Data flows both into and out of a subject.
但这是为了什么？你可以将 IObserver<T> 和 IObservable<T> 分别视为“消费者”和“发布者”接口。那么，主体既是消费者又是发布者。数据流入和流出主题。

Rx offers a few subject implementations that can occasionally be useful in code that wants to make an IObservable<T> available. Although Observable.Create is usually the preferred way to do this, there's one important case where a subject might make more sense: if you have some code that discovers events of interest (e.g., by using the client API for some messaging technology) and wants to make them available through an IObservable<T>, subjects can sometimes provide a more convenient way to to this than with Observable.Create or a custom implementation.
Rx 提供了一些主题实现，这些实现有时在想要使 IObservable<T> 可用的代码中很有用。虽然 Observable.Create 通常是执行此操作的首选方法，但在一个重要的情况下，主题可能更有意义：如果你有一些代码可以发现感兴趣的事件（例如，通过使用客户端 API 进行某些消息传递）技术）并希望通过 IObservable<T> 提供它们，主题有时可以提供比 Observable.Create 或自定义实现更方便的方法。

Rx offers a few subject types. We'll start with the most straightforward one to understand.
Rx 提供了几种主题类型。我们将从最容易理解的一个开始。

Subject<T>
The Subject<T> type immediately forwards any calls made to its IObserver<T> methods on to all of the observers currently subscribed to it. This example shows its basic operation:
Subject<T> 类型立即将对它的 IObserver<T> 方法的任何调用转发给当前订阅它的所有观察者。这个例子展示了它的基本操作：

```C#
Subject<int> s = new();
s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));

s.OnNext(1);
s.OnNext(2);
s.OnNext(3);
```

I've created a Subject<int>. I've subscribed to it twice, and then called its OnNext method repeatedly. This produces the following output, illustrating that the Subject<int> forwards each OnNext call onto both subscribers:
我创建了一个 Subject<int> 。我已经订阅了它两次，然后重复调用它的 OnNext 方法。这会产生以下输出，说明 Subject<int> 将每个 OnNext 调用转发给两个订阅者：

```
Sub1: 1
Sub2: 1
Sub1: 2
Sub2: 2
Sub1: 3
Sub2: 3
```

We could use this as a way to bridge between some API from which we receive data into the world of Rx. You could imagine writing something of this kind:
我们可以使用它作为一些 API 之间的桥梁，我们从这些 API 接收数据到 Rx 的世界。你可以想象写这样的东西：

```C# {linenos=table,hl_lines=[],linenostart=1,filename="MessageQueueToRx.cs"}
public class MessageQueueToRx : IDisposable
{
    private readonly Subject<string> messages = new();

    public IObservable<string> Messages => messages;

    public void Run()
    {
        while (true)
        {
            // Receive a message from some hypothetical message queuing service
            string message = MqLibrary.ReceiveMessage();
            messages.OnNext(message);
        }
    }

    public void Dispose()
    {
        message.Dispose();
    }
}
```

It wouldn't be too hard to modify this to use Observable.Create instead. But where this approach can become easier is if you need to provide multiple different IObservable<T> sources. Imagine we distinguish between different message types based on their content, and publish them through different observables. That's hard to arrange with Observable.Create if we still want a single loop pulling messages off the queue.
修改它以使用 Observable.Create 代替并不会太难。但如果你需要提供多个不同的 IObservable<T> 源，则此方法会变得更容易。想象一下，我们根据内容区分不同的消息类型，并通过不同的可观察量发布它们。如果我们仍然想要一个循环将消息从队列中拉出，那么就很难用 Observable.Create 来安排。

Subject<T> also distributes calls to either OnCompleted or OnError to all subscribers. Of course, the rules of Rx require that once you have called either of these methods on an IObserver<T> (and any ISubject<T> is an IObserver<T>, so this rule applies to Subject<T>) you must not call OnNext, OnError, or OnComplete on that observer ever again. In fact, Subject<T> will tolerate calls that break this rule—it just ignores them, so even if your code doesn't quite stick to these rules internally, the IObservable<T> you present to the outside world will behave correctly, because Rx enforces this.
Subject<T> 还将对 OnCompleted 或 OnError 的呼叫分配给所有订阅者。当然，Rx 的规则要求一旦你在 IObserver<T> 上调用了这些方法中的任何一个（并且任何 ISubject<T> 都是 IObserver<T> ，因此此规则适用到 Subject<T> ），你不得再对该观察者调用 OnNext 、 OnError 或 OnComplete 。事实上， Subject<T> 会容忍违反此规则的调用 - 它只是忽略它们，因此即使你的代码在内部不太遵守这些规则，你呈现给的 IObservable<T> 外部世界将表现正确，因为 Rx 强制执行此操作。

Subject<T> implements IDisposable. Disposing a Subject<T> puts it into a state where it will throw an exception if you call any of its methods. The documentation also describes it as unsubscribing all observers, but since a disposed Subject<T> isn't capable of producing any further notifications in any case, this doesn't really mean much. (Note that it does not call OnCompleted on its observers when you Dispose it.) The one practical effect is that its internal field that keeps track of observers is reset to a special sentinel value indicating that it has been disposed, meaning that the one externally observable effect of "unsubscribing" the observers is that if, for some reason, your code held onto a reference to a Subject<T> after disposing it, that would no longer keep all the subscribers reachable for GC purposes. If a Subject<T> remains reachable indefinitely after it is no longer in use, that in itself is effectively a memory leak, but disposal would at least limit the effects: only the Subject<T> itself would remain reachable, and not all of its subscribers.
Subject<T> 实现 IDisposable 。处置 Subject<T> 会将其置于一种状态，如果你调用其任何方法，它将引发异常。该文档还将其描述为取消订阅所有观察者，但由于已处理的 Subject<T> 在任何情况下都无法生成任何进一步的通知，因此这并没有多大意义。 （请注意，当你 Dispose 它时，它不会对其观察者调用 OnCompleted 。）一个实际效果是，其跟踪观察者的内部字段被重置为特殊的哨兵值表明它已被处置，这意味着“取消订阅”观察者的一个外部可观察的效果是，如果由于某种原因，你的代码在处置它之后保留了对 Subject<T> 的引用，则不会不再为了 GC 目的而保持所有订阅者可达。如果 Subject<T> 在不再使用后仍然无限期地可访问，那么它本身实际上就是内存泄漏，但处置至少会限制影响：只有 Subject<T> 本身会保留可以访问，但并非所有订阅者都可以访问。

Subject<T> is the most straightforward subject, but there are other, more specialized ones.
Subject<T> 是最简单的主题，但还有其他更专业的主题。

ReplaySubject<T>
Subject<T> does not remember anything: it immediately distributes incoming values to subscribers. If new subscribers come along, they will only see events that occur after they subscribe. ReplaySubject<T>, on the other hand, can remember every value it has ever seen. If a new subject comes along, it will receive the complete history of events so far.
Subject<T> 不记得任何事情：它立即将传入的值分发给订阅者。如果有新订阅者出现，他们只会看到订阅后发生的事件。另一方面， ReplaySubject<T> 可以记住它见过的每个值。如果出现新的主题，它将收到迄今为止事件的完整历史记录。

This is a variation on the first example in the preceding Subject<T> section. It creates a ReplaySubject<int> instead of a Subject<int>. And instead of immediately subscribing twice, it creates an initial subscription, and then a second one only after a couple of values have been emitted.
这是前面 Subject<T> 部分中第一个示例的变体。它创建一个 ReplaySubject<int> 而不是 Subject<int> 。它不是立即订阅两次，而是创建一个初始订阅，然后仅在发出几个值后才创建第二个订阅。

```C#
ReplaySubject<int> s = new();
s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));

s.OnNext(1);
s.OnNext(2);

s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));

s.OnNext(3);
```

This produces the following output:
这会产生以下输出：

```
Sub1: 1
Sub1: 2
Sub2: 1
Sub2: 2
Sub1: 3
Sub2: 3
```

As you'd expect, we initially see output only from Sub1. But when we make the second call to subscribe, we can see that Sub2 also received the first two values. And then when we report the third value, both see it. If this example had used Subject<int> instead, we would have seen just this output:
正如你所期望的，我们最初只看到来自 Sub1 的输出。但是当我们第二次调用 subscribe 时，我们可以看到 Sub2 也收到了前两个值。然后，当我们报告第三个值时，双方都会看到它。如果这个例子使用 Subject<int> 代替，我们就会看到这样的输出：

```
Sub1: 1
Sub1: 2
Sub1: 3
Sub2: 3
```

There's an obvious potential problem here: if ReplaySubject<T> remembers every value published to it, we mustn't use it with endless event sources, because it will eventually cause us to run out of memory.
这里有一个明显的潜在问题：如果 ReplaySubject<T> 记住发布给它的每个值，我们就不能将它与无穷无尽的事件源一起使用，因为它最终会导致我们耗尽内存。

ReplaySubject<T> offers constructors that accept simple cache expiry settings that can limit memory consumption. One option is to specify the maximum number of item to remember. This next example creates a ReplaySubject<T> with a buffer size of 2:
ReplaySubject<T> 提供接受简单缓存到期设置的构造函数，这些设置可以限制内存消耗。一种选择是指定要记住的最大项目数。下一个示例创建缓冲区大小为 2 的 ReplaySubject<T> ：

```C#
ReplaySubject<int> s = new(2);
s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));

s.OnNext(1);
s.OnNext(2);
s.OnNext(3);

s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));

s.OnNext(4);
```

Since the second subscription only comes along after we've already produced 3 values, it no longer sees all of them. It only receives the last two values published prior to subscription (but the first subscription continues to see everything of course):
由于第二个订阅仅在我们生成 3 个值之后才会出现，因此它不再看到所有这些值。它只接收订阅之前发布的最后两个值（但第一个订阅当然会继续看到所有内容）：

```
Sub1: 1
Sub1: 2
Sub1: 3
Sub2: 2
Sub2: 3
Sub1: 4
Sub2: 4
```

Alternatively, you can specify a time-based limit by passing a TimeSpan to the ReplaySubject<T> constructor.
或者，你可以通过将 TimeSpan 传递给 ReplaySubject<T> 构造函数来指定基于时间的限制。

BehaviorSubject<T>
Like ReplaySubject<T>, BehaviorSubject<T> also has a memory, but it remembers exactly one value. However, it's not quite the same as a ReplaySubject<T> with a buffer size of 1. Whereas a ReplaySubject<T> starts off in a state where it has nothing in its memory, BehaviorSubject<T> always remembers exactly one item. How can that work before we've made our first call to OnNext? BehaviorSubject<T> enforces this by requiring us to supply the initial value when we construct it.
与 ReplaySubject<T> 一样， BehaviorSubject<T> 也有记忆，但它只记住一个值。但是，它与缓冲区大小为 1 的 ReplaySubject<T> 并不完全相同。 ReplaySubject<T> 开始时内存中没有任何内容，而 BehaviorSubject<T> 之前，它如何工作？ BehaviorSubject<T> 通过要求我们在构造它时提供初始值来强制执行此操作。

So you can think of BehaviorSubject<T> as a subject that always has a value available. If you subscribe to a BehaviorSubject<T> it will instantly produce a single value. (It may then go on to produce more values, but it always produces one right away.) As it happens, it also makes that value available through a property called Value, so you don't need to subscribe an IObserver<T> to it just to retrieve the value.
因此，你可以将 BehaviorSubject<T> 视为始终具有可用值的主题。如果你订阅 BehaviorSubject<T> ，它将立即生成一个值。 （然后它可能会继续生成更多值，但它总是立即生成一个值。）碰巧的是，它还通过名为 Value 的属性提供该值，因此你无需订阅向其添加 IObserver<T> 只是为了检索值。

A BehaviorSubject<T> could be thought of an as observable property. Like a normal property, it can immediately supply a value whenever you ask it. The difference is that it can then go on to notify you every time its value changes. If you're using the ReactiveUI framework (an Rx-based framework for building user interfaces), BehaviourSubject<T> can make sense as the implementation type for a property in a view model (the type that mediates between your underlying domain model and your user interface). It has property-like behaviour, enabling you to retrieve a value at any time, but it also provides change notifications, which ReactiveUI can handle in order to keep the UI up to date.
BehaviorSubject<T> 可以被认为是可观察的属性。与普通属性一样，只要你询问，它就可以立即提供值。不同之处在于，每次其值发生变化时，它都会继续通知你。如果你使用 ReactiveUI 框架（用于构建用户界面的基于 Rx 的框架），则 BehaviourSubject<T> 可以作为视图模型中属性的实现类型（在底层域之间进行调解的类型）。模型和你的用户界面）。它具有类似属性的行为，使你能够随时检索值，但它还提供更改通知，ReactiveUI 可以处理这些通知，以使 UI 保持最新。

This analogy falls down slightly when it comes to completion. If you call OnCompleted, it immediately calls OnCompleted on all of its observers, and if any new observers subscribe, they will also immediately be completed—it does not first supply the last value. (So this is another way in which it is different from a ReplaySubject<T> with a buffer size of 1.)
这个类比在完成时略有下降。如果你调用 OnCompleted ，它会立即在其所有观察者上调用 OnCompleted ，并且如果有任何新观察者订阅，它们也将立即完成 - 它不会首先提供最后一个值。 （所以这是与缓冲区大小为 1 的 ReplaySubject<T> 不同的另一种方式。）

Similarly, if you call OnError, all current observers will receive an OnError call, and any subsequent subscribers will also receive nothing but an OnError call.
同样，如果你调用 OnError ，所有当前观察者都将收到 OnError 调用，并且任何后续订阅者也只会收到 OnError 调用。

AsyncSubject<T>
AsyncSubject<T> provides all observers with the final value it receives. Since it can't know which is the final value until OnCompleted is called, it will not invoke any methods on any of its subscribers until either its OnCompleted or OnError method is called. (If OnError is called, it just forwards that to all current and future subscribers.) You will often use this subject indirectly, because it is the basis of Rx's integration with the await keyword. (When you await an observable sequence, the await returns the final value emitted by the source.)
AsyncSubject<T> 向所有观察者提供它收到的最终值。由于在调用 OnCompleted 之前它无法知道哪个是最终值，因此在 OnCompleted 或 OnError 方法被调用。 （如果 OnError 被调用，它只是将其转发给所有当前和未来的订阅者。）你经常会间接使用这个主题，因为它是Rx与 await 关键字集成的基础。 （当你 await 可观察序列时， await 返回源发出的最终值。）

If no calls were made to OnNext before OnCompleted then there was no final value, so it will just complete any observers without providing a value.
如果在 OnCompleted 之前没有调用 OnNext 则没有最终值，因此它将完成任何观察者而不提供值。

In this example no values will be published as the sequence never completes. No values will be written to the console.
在此示例中，由于序列永远不会完成，因此不会发布任何值。不会将任何值写入控制台。

```C#
AsyncSubject<string> subject = new();
subject.OnNext("a");
subject.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
subject.OnNext("b");
subject.OnNext("c");
```

In this example we invoke the OnCompleted method so there will be a final value ('c') for the subject to produce:
在此示例中，我们调用 OnCompleted 方法，以便主题生成最终值（'c'）：

```C#
AsyncSubject<string> subject = new();

subject.OnNext("a");
subject.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
subject.OnNext("b");
subject.OnNext("c");
subject.OnCompleted();
subject.Subscribe(x => Console.WriteLine($"Sub2: {x}"));
```

This produces the following output:
这会产生以下输出：

```
Sub1: c
Sub2: c
```

If you have some potentially slow work that needs to be done when your application starts up, and which needs to be done just once, you might choose an AsyncSubject<T> to make the results of that work available. Code requiring those results can subscribe to the subject. If the work is not yet complete, they will receive the results as soon as they are available. And if the work has already completed, they will receive it immediately.
如果你有一些可能很慢的工作需要在应用程序启动时完成，并且只需完成一次，你可以选择 AsyncSubject<T> 来提供该工作的结果。需要这些结果的代码可以订阅该主题。如果工作尚未完成，他们将在结果出来后立即收到。如果工作已经完成，他们会立即收到。

Subject factory 对象工厂
Finally it is worth making you aware that you can also create a subject via a factory method. Considering that a subject combines the IObservable<T> and IObserver<T> interfaces, it seems sensible that there should be a factory that allows you to combine them yourself. The Subject.Create(IObserver<TSource>, IObservable<TResult>) factory method provides just this.
最后值得让你意识到的是，你还可以通过工厂方法创建主题。考虑到一个主题结合了 IObservable<T> 和 IObserver<T> 接口，似乎应该有一个工厂允许你自己组合它们。 Subject.Create(IObserver<TSource>, IObservable<TResult>) 工厂方法就提供了这一点。

```C#
// Creates a subject from the specified observer used to publish messages to the
// subject and observable used to subscribe to messages sent from the subject
public static ISubject<TSource, TResult> Create<TSource, TResult>(
    IObserver<TSource> observer, 
    IObservable<TResult> observable)
{...}
```

Note that unlike all of the other subjects just discussed, this creates a subject where there is no inherent relationship between the input and the output. This just takes whatever IObserver<TSource> and IObserver<TResult> implementations you supply and wraps them up in a single object. All calls made to the subject's IObserver<TSource> methods will be passed directly to the observer you supplied. If you want values to emerge to subscribers to the corresponding IObservable<TResult>, it's up to you to make that happen. This really combines the two objects you supply with the absolute minimum of glue.
请注意，与刚刚讨论的所有其他主题不同，这创建了一个输入和输出之间没有内在关系的主题。这只需要你提供的任何 IObserver<TSource> 和 IObserver<TResult> 实现并将它们包装在一个对象中。对主题的 IObserver<TSource> 方法进行的所有调用都将直接传递给你提供的观察者。如果你希望向相应 IObservable<TResult> 的订阅者显示值，则由你来实现。这确实用最少的胶水将你提供的两个物体结合在一起。

Subjects provide a convenient way to poke around Rx, and are occasionally useful in production scenarios, but they are not recommended for most cases. An explanation is in the Usage Guidelines appendix. Instead of using subjects, favour the factory methods shown earlier in this chapter..
主题提供了一种探索 Rx 的便捷方法，并且在生产场景中偶尔有用，但在大多数情况下不推荐使用。使用指南附录中有解释。不要使用主题，而应使用本章前面所示的工厂方法。

Summary 概括
We have looked at the various eager and lazy ways to create a sequence. We have seen how to produce timer based sequences using the various factory methods. And we've also explored ways to transition from other synchronous and asynchronous representations.
我们已经研究了创建序列的各种急切和惰性方法。我们已经了解了如何使用各种工厂方法生成基于计时器的序列。我们还探索了从其他同步和异步表示转换的方法。

As a quick recap:
快速回顾一下：

Factory Methods 工厂方法

Observable.Return
Observable.Empty 可观察.空
Observable.Never 可观察。从不
Observable.Throw 可观察.抛出
Observable.Create 可观察.创建
Observable.Defer 可观察.延迟
Generative methods 生成方法

Observable.Range 可观测范围
Observable.Generate 可观察.生成
Observable.Interval 可观察的间隔
Observable.Timer
Adaptation 适应

Observable.Start
Observable.FromEventPattern
Task.ToObservable 任务.ToObservable
Task<T>.ToObservable 任务.ToObservable
IEnumerable<T>.ToObservable
Observable.FromAsyncPattern
Creating an observable sequence is our first step to practical application of Rx: create the sequence and then expose it for consumption. Now that we have a firm grasp on how to create an observable sequence, we can look in more detail at the operators that allow us to describe processing to be applied, to build up more complex observable sequences.
创建可观察序列是我们实际应用 Rx 的第一步：创建序列，然后将其公开以供使用。现在我们已经牢牢掌握了如何创建可观察序列，我们可以更详细地了解允许我们描述要应用的处理的运算符，以构建更复杂的可观察序列。