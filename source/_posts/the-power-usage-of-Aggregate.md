---
title: the power usage of Aggregate
date: 2017-09-14 10:40:10
tags:
---
## the power usage of Aggregate


因为阅读了某位作者文章的缘故，我也去翻了下.net Mvc启动管道的源码。在其中发现了这么一段代码：

```C#
     endContinuation = filters.Reverse<IActionFilter>().Aggregate<IActionFilter, Func<Func<ActionExecutedContext>>>((Func<Func<ActionExecutedContext>>) (() =>
        {
          innerAsyncResult = this.BeginInvokeActionMethod(controllerContext, actionDescriptor, parameters, asyncCallback, asyncState);
          return (Func<ActionExecutedContext>) (() => new ActionExecutedContext(controllerContext, actionDescriptor, false, (Exception) null)
          {
            Result = this.EndInvokeActionMethod(innerAsyncResult)
          });
        }), (Func<Func<Func<ActionExecutedContext>>, IActionFilter, Func<Func<ActionExecutedContext>>>) ((next, filter) => (Func<Func<ActionExecutedContext>>) (() => AsyncControllerActionInvoker.InvokeActionMethodFilterAsynchronously(filter, preContext, next))))();
```

一看之下，几乎要被它长长的气势所吓倒。其本质是Linq聚合函数，只不过一串Func实在太过耀眼。为了深入理解，我写了段相似代码来模拟运行。

```C#
        public static void TestFilters()
        {
            var filters = new List<IMyFilter> {new MyFilter.FilterA(), new MyFilter.FilterB(), new MyFilter.FilterC()};
            var invoker = new Invoker();
            var context = new ExecutingContext();
            var endContinuation = filters.Aggregate<IMyFilter, Func<Func<ExecutedContext>>>((Func<Func<ExecutedContext>>)(() =>
            {
                var innerResult = invoker.BeginInvoke(context);
                return (Func<ExecutedContext>)(() => new ExecutedContext()
                {
                    Result = invoker.EndInvoke(innerResult)
                });
            }), (Func<Func<Func<ExecutedContext>>, IMyFilter, Func<Func<ExecutedContext>>>)((next, filter) => 
            (Func<Func<ExecutedContext>>)(() => InvokeAction(filter, context, next))))();

            endContinuation();
        }

        internal static Func<ExecutedContext> InvokeAction(IMyFilter filter, ExecutingContext preContext, Func<Func<ExecutedContext>> nextInChain)
        {
            filter.OnExecuting(preContext);
            if (preContext.Result != null)
            {
                ExecutedContext shortCircuitedPostContext = new ExecutedContext()
                {
                    Result = preContext.Result
                };
                return () => shortCircuitedPostContext;
            }
            try
            {
                Func<ExecutedContext> continuation = nextInChain();
                return (Func<ExecutedContext>)(() =>
                {
                    bool flag = true;
                    ExecutedContext filterContext;
                    try
                    {
                        filterContext = continuation();
                        flag = false;
                    }
                    catch (Exception ex)
                    {
                        filterContext = new ExecutedContext();
                        filter.OnExecuted(filterContext);
                        if (!filterContext.ExceptionHandled)
                            throw;
                    }
                    if (!flag)
                        filter.OnExecuted(filterContext);
                    return filterContext;
                });
            }
            catch (Exception ex)
            {
                ExecutedContext postContext = new ExecutedContext();
                filter.OnExecuted(postContext);
                if (postContext.ExceptionHandled)
                    return (Func<ExecutedContext>)(() => postContext);
                throw;
            }
        }
```

这里的流程如下：

>集合：FilterA, FilterB, FilterC

>seed：seedFunc

>第一趟：FilterA, seedFunc => nextFunc1

>第二趟：FilterB, nextFunc1 => nextFunc2

>第三趟：FilterC, nextFunc2 => nextFunc3

循环结束，调用nextFunc3: 

>FilterC.OnExecuting, call nextFunc2

>>FilterB.OnExecuting, call nextFunc1

>>>FilterA.OnExecuting, call seedFunc

>>>>invoker.BeginInvoke

>>>>invoker.EndInvoke

>>>FilterA.OnExecuted

>>FilterB.OnExecuting

>FilterC.OnExecuting

从此，我们也可以窥见Mvc Action Filter的原理是什么。

一般来说，我们对于Linq Aggregate 的运用都来自于很简单的例子，或者很常见的场景，比如字符串连接：

```C#
    new List<String>() {"aa", "bb"}.Aggregate((t, e) => t + "-" + e)
```

这么写谁都会，但涉及到复杂一点的东西，就不会想到用聚合来做。说到底，还是对某种概念的理解流于表面。例如，我们可以这样实现一个累加生成器：

```C#
        Func<int, int> Sumer(IEnumerable<int> seeds)
        {
            return seeds.Aggregate<int, Func<int, int>>(x => x, (acc, number) => y => acc(number) + y);
        }
        Sumer(new [] {1,2,3})(4); //10
```


