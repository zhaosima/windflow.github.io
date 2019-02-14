---
title: why use ConfigureAwait
date: 2018-12-14 17:15:53
tags:
---
# Why use ConfigureAwait(false)

思考当我们写下这样的代码时，发生了什么？

``` c#
    public async Task BuyJuice()
    {
        await WaitInQueue();

        Pay();

        await GetJuice();
    }
```

## What happened in async/await

内部原理参考这篇 [文章](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-1-compilation)

简单地来说，编译器会将 async 方法转化为 AsyncStateMachine。 而 await 会转化为 StateMachine.MoveNext() 方法内部的 GetAwaiter();。接着判断 awaiter 是否 Completed。 如果没有，则阻塞等待。 等待结束后，变更状态。如果还有后续任务，则继续调用MoveNext(); StateMachine 根据上一步的状态来决定当前的操作。

需要注意的是，不管方法是否有 async 标识，只要返回了 Task，都是 awaitable 。

## What's more

当创建一个 Task 的时候，会捕捉当前线程的 ExecuteionContext。当 Task Start 时，会将自己 push 到线程池的任务队列中。 当线程池调度到某个线程，线程会从池队列中捞取一个 Task。 然后通过 ExecuteionContext 的静态方法 RunXXX 来执行 Task。

## Know ExecuteionContext

>The ExecutionContext class provides a single container for all information relevant to a logical thread of execution. This includes security context, call context, and synchronization context.
>The ExecutionContext class provides the functionality for user code to capture and transfer this context across user-defined asynchronous points. The common language runtime ensures that the ExecutionContext is consistently transferred across runtime-defined asynchronous points within the managed process."

flow ExecutionContext 是指从创建 Task 时从一个线程中捕捉 ExecutionContext, 并在执行时"*复制*"到运行的线程中。主要是用来做数据共享。

## What is SynchronizationContext

SynchronizationContext 类是一个基类，可提供不带同步的自由线程上下文。 此类实现的同步模型的目的是使公共语言运行库内部的异步/同步操作能够针对不同的异步模型采取正确的行为。此模型还简化了托管应用程序为在不同的同步环境下正常工作而必须遵循的一些要求。同步模型的提供程序可以扩展此类并为这些方法提供自己的实现。【MSDN】

SynchronizationContext 提供了一种高级别的抽象。

这篇[文章](https://blogs.msdn.microsoft.com/pfxteam/2012/06/15/executioncontext-vs-synchronizationcontext/)讲述了 ExecutionContext 与 SynchronizationContext 之间的差异。

1. 每个 async 方法都有自己的 SynchronizationContext
2. 不是每个线程都附加了 SynchronizationContext
3. 获取SynchronizationContext 通过 SynchronizationContext.Current。 通常 UI Thread , asp.net Thread 会设置该 Current。

## Why need SynchronizationContext

简单的来说用于线程间通讯。

这篇[文章](https://www.codeproject.com/Articles/31971/Understanding-SynchronizationContext-Part-I)深入浅出的讲解了 SynchronizationContext 的用处以及 Post , Send 方法的使用。

## When to abandon SynchronizationContext

当我们不需要通过 SynchronizationContext 通讯的时候，或者说我们无需调用发起Task的那个线程的代码的时候。

当采用 ConfigureAwait(continueOnCapturedContext:false) 时， 得到的是 ConfiguredTaskAwaiter。 这个awaiter 在continueOnCapturedContext等于false的时候，不会创建用于捕捉 SynchronizationContext 的 TaskContinuation(SynchronizationContextAwaitTaskContinuation)。