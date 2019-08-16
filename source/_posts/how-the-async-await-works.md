---
title: how the async/await works
date: 2018-12-17 16:45:37
tags:
---

# How the async/await works

为了一探 async&await 的究竟，让我们先写一个最简单的 WinForm。 有一个按钮，点一下加载图片。只不过这次加载是异步的，不会阻塞 UI 线程。


```Java
    private async void button1_Click(object sender, EventArgs e)
    {
        this.textBox1.Text = $"#{GetThreadInfo()}# 获取图像中.....";
        var sh = await LoadImage();
        this.textBox1.Text = $"#{sh}# 已获得图像";
    }
    
    private async Task<string> LoadImage()
    {
        await Task.Delay(5000);
        return "welcome.gif";
    }
```

使用 ILSpy 或者 DotPeek 加载编译后的 exe, 找到 Form1 class, 然后打开 IL 查看器。首先来看看 button1_Click 的编译结果。

```Java
    .method private hidebysig instance void 
        button1_Click(
          object sender, 
          class [mscorlib]System.EventArgs e
        ) cil managed 
      {
        .custom instance void [mscorlib]    System.Runtime.CompilerServices.AsyncStateMachineAttribute::.ctor(class [mscorlib]    System.Type) 
          = (
            01 00 2a 41 73 79 6e 63 43 61 6c 6c 57 69 6e 46 // ..*AsyncCallWinF
            6f 72 6d 2e 46 6f 72 6d 31 2b 3c 62 75 74 74 6f // orm.Form1+<butto
            6e 31 5f 43 6c 69 63 6b 3e 64 5f 5f 31 00 00    // n1_Click>d__1..
          )
          // type(class AsyncCallWinForm.Form1/'<button1_Click>d__1')
        .custom instance void [mscorlib]    System.Diagnostics.DebuggerStepThroughAttribute::.ctor() 
          = (01 00 00 00 )
        .maxstack 2
        .locals init (
          [0] class AsyncCallWinForm.Form1/'<button1_Click>d__1' V_0,
          [1] valuetype [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder     V_1
        )
    
        IL_0000: newobj       instance void     AsyncCallWinForm.Form1/'<button1_Click>d__1'::.ctor()
        IL_0005: stloc.0      // V_0
        IL_0006: ldloc.0      // V_0
        IL_0007: ldarg.0      // this
        IL_0008: stfld        class AsyncCallWinForm.Form1     AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>4__this'
        IL_000d: ldloc.0      // V_0
        IL_000e: ldarg.1      // sender
        IL_000f: stfld        object AsyncCallWinForm.Form1/'<button1_Click>d__1'::sender
        IL_0014: ldloc.0      // V_0
        IL_0015: ldarg.2      // e
        IL_0016: stfld        class [mscorlib]System.EventArgs     AsyncCallWinForm.Form1/'<button1_Click>d__1'::e
        IL_001b: ldloc.0      // V_0
        IL_001c: call         valuetype [mscorlib]    System.Runtime.CompilerServices.AsyncVoidMethodBuilder [mscorlib]    System.Runtime.CompilerServices.AsyncVoidMethodBuilder::Create()
        IL_0021: stfld        valuetype [mscorlib]    System.Runtime.CompilerServices.AsyncVoidMethodBuilder     AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>t__builder'
        IL_0026: ldloc.0      // V_0
        IL_0027: ldc.i4.m1    // -1
        IL_0028: stfld        int32     AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>1__state'
        IL_002d: ldloc.0      // V_0
        IL_002e: ldfld        valuetype [mscorlib]    System.Runtime.CompilerServices.AsyncVoidMethodBuilder     AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>t__builder'
        IL_0033: stloc.1      // V_1
        IL_0034: ldloca.s     V_1
        IL_0036: ldloca.s     V_0
        IL_0038: call         instance void [mscorlib]    System.Runtime.CompilerServices.AsyncVoidMethodBuilder::Start<class     AsyncCallWinForm.Form1/'<button1_Click>d__1'>(!!0/*class     AsyncCallWinForm.Form1/'<button1_Click>d__1'*/&)
        IL_003d: ret          
    
      } // end of method Form1::button1_Click        
```

根据 IL 翻译成的代码如下：

```Java
    AsyncCallWinForm.Form1/'<button1_Click>d__1 V_0;
    AsyncVoidMethodBuilder V_1;
    
    new AsyncCallWinForm.Form1/'<button1_Click>d__1(); // value_on_stack_0
    V_0 = value_on_stack_0
    V_0.refer_to_form = this;
    V_0.sender = sender;
    V_0.e = e;
    V_0.t_builder = AsyncVoidMethodBuilder::Create();
    V_0.state = -1;
    V_1= V_0.t_builder;
    Methodof(AsyncVoidMethodBuilder.Start).Invoke(V_1/*this pointer of     AsyncVoidMethodBuilder*/, V_0);
```

转成方便人类阅读的代码如下：

```Java
    var buidler = AsyncVoidMethodBuilder::Create(); // 由于是 async void 所以生成的是 AsyncVoidMethodBuilder。 当然一般不建议用 async void。
    var stateMachine = new button1_Click_stateMachine();
    stateMachine.Init(); // 初始化代码，忽略
    stateMachine.State = -1;
    buidler.Start(stateMachine);
```

那么 builder.Start 做了什么呢？ 继续看。  找到 AsyncVoidMethodBuilder(.net core 版本) 的[源码](https://github.com/dotnet/coreclr/blob/master/src/System.Private.CoreLib/src/System/Runtime/CompilerServices/AsyncMethodBuilder.cs)

```Java
    [DebuggerStepThrough]
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Start<TStateMachine>(ref TStateMachine stateMachine) where StateMachine : IAsyncStateMachine =>
        AsyncMethodBuilderCore.Start(ref stateMachine);

    public static void Start<TStateMachine>(ref TStateMachine stateMachine) where TStateMachine : IAsyncStateMachine
    {
        // 一些省略的保存ExecutionContext的代码

        try
        {
            stateMachine.MoveNext();
        }
        finally
        {
            // 一些省略的复原ExecutionContext的代码
        }
    }
```

接下来就要看看 StateMachine.MoveNext 都做了什么. 这次回到 Form1 的反编译结果。

```Java
    .class nested private sealed auto ansi beforefieldinit 
    '<button1_Click>d__1'
      extends [mscorlib]System.Object
      implements [mscorlib]System.Runtime.CompilerServices.IAsyncStateMachine
    {
    .custom instance void [mscorlib]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor() 
      = (01 00 00 00 )

    .field public int32 '<>1__state'

    .field public valuetype [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder '<>t__builder'

    .field public object sender

    .field public class [mscorlib]System.EventArgs e

    .field public class AsyncCallWinForm.Form1 '<>4__this'

    .field private string '<sh>5__1'

    .field private string '<>s__2'

    .field private valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string> '<>u__1'

    .method public hidebysig specialname rtspecialname instance void 
      .ctor() cil managed 
    {
      .maxstack 8

      IL_0000: ldarg.0      // this
      IL_0001: call         instance void [mscorlib]System.Object::.ctor()
      IL_0006: nop          
      IL_0007: ret          

    } // end of method '<button1_Click>d__1'::.ctor

    .method private final hidebysig virtual newslot instance void 
      MoveNext() cil managed 
    {
      .override method instance void [mscorlib]System.Runtime.CompilerServices.IAsyncStateMachine::MoveNext() 
      .maxstack 4
      .locals init (
        [0] int32 V_0,
        [1] valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string> V_1,
        [2] class AsyncCallWinForm.Form1/'<button1_Click>d__1' V_2,
        [3] class [mscorlib]System.Exception V_3
      )

      IL_0000: ldarg.0      // this
      IL_0001: ldfld        int32 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>1__state'
      IL_0006: stloc.0      // V_0
      .try
      {

        IL_0007: ldloc.0      // V_0
        IL_0008: brfalse.s    IL_000c
        IL_000a: br.s         IL_000e
        IL_000c: br.s         IL_0075

        // [21 9 - 21 10]
        IL_000e: nop          

        // [22 13 - 22 68]
        IL_000f: ldarg.0      // this
        IL_0010: ldfld        class AsyncCallWinForm.Form1 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>4__this'
        IL_0015: ldfld        class [System.Windows.Forms]System.Windows.Forms.TextBox AsyncCallWinForm.Form1::textBox1
        IL_001a: ldstr        "#"
        IL_001f: call         string AsyncCallWinForm.Form1::GetThreadInfo()
        IL_0024: ldstr        "# 获取图像中....."
        IL_0029: call         string [mscorlib]System.String::Concat(string, string, string)
        IL_002e: callvirt     instance void [System.Windows.Forms]System.Windows.Forms.Control::set_Text(string)
        IL_0033: nop          

        // [24 13 - 24 40]
        IL_0034: ldarg.0      // this
        IL_0035: ldfld        class AsyncCallWinForm.Form1 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>4__this'
        IL_003a: call         instance class [mscorlib]System.Threading.Tasks.Task`1<string> AsyncCallWinForm.Form1::LoadImage()
        IL_003f: callvirt     instance valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<!0/*string*/> class [mscorlib]System.Threading.Tasks.Task`1<string>::GetAwaiter()
        IL_0044: stloc.1      // V_1

        IL_0045: ldloca.s     V_1
        IL_0047: call         instance bool valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string>::get_IsCompleted()
        IL_004c: brtrue.s     IL_0091
        IL_004e: ldarg.0      // this
        IL_004f: ldc.i4.0     
        IL_0050: dup          
        IL_0051: stloc.0      // V_0
        IL_0052: stfld        int32 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>1__state'
        IL_0057: ldarg.0      // this
        IL_0058: ldloc.1      // V_1
        IL_0059: stfld        valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string> AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>u__1'
        IL_005e: ldarg.0      // this
        IL_005f: stloc.2      // V_2
        IL_0060: ldarg.0      // this
        IL_0061: ldflda       valuetype [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>t__builder'
        IL_0066: ldloca.s     V_1
        IL_0068: ldloca.s     V_2
        IL_006a: call         instance void [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder::AwaitUnsafeOnCompleted<valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string>, class AsyncCallWinForm.Form1/'<button1_Click>d__1'>(!!0/*valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string>*/&, !!1/*class AsyncCallWinForm.Form1/'<button1_Click>d__1'*/&)
        IL_006f: nop          
        IL_0070: leave        IL_0105
        IL_0075: ldarg.0      // this
        IL_0076: ldfld        valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string> AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>u__1'
        IL_007b: stloc.1      // V_1
        IL_007c: ldarg.0      // this
        IL_007d: ldflda       valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string> AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>u__1'
        IL_0082: initobj      valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string>
        IL_0088: ldarg.0      // this
        IL_0089: ldc.i4.m1    
        IL_008a: dup          
        IL_008b: stloc.0      // V_0
        IL_008c: stfld        int32 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>1__state'
        IL_0091: ldarg.0      // this
        IL_0092: ldloca.s     V_1
        IL_0094: call         instance !0/*string*/ valuetype [mscorlib]System.Runtime.CompilerServices.TaskAwaiter`1<string>::GetResult()
        IL_0099: stfld        string AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>s__2'
        IL_009e: ldarg.0      // this
        IL_009f: ldarg.0      // this
        IL_00a0: ldfld        string AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>s__2'
        IL_00a5: stfld        string AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<sh>5__1'
        IL_00aa: ldarg.0      // this
        IL_00ab: ldnull       
        IL_00ac: stfld        string AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>s__2'

        // [26 13 - 26 50]
        IL_00b1: ldarg.0      // this
        IL_00b2: ldfld        class AsyncCallWinForm.Form1 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>4__this'
        IL_00b7: ldfld        class [System.Windows.Forms]System.Windows.Forms.TextBox AsyncCallWinForm.Form1::textBox1
        IL_00bc: ldstr        "#"
        IL_00c1: ldarg.0      // this
        IL_00c2: ldfld        string AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<sh>5__1'
        IL_00c7: ldstr        "# 已获得图像"
        IL_00cc: call         string [mscorlib]System.String::Concat(string, string, string)
        IL_00d1: callvirt     instance void [System.Windows.Forms]System.Windows.Forms.Control::set_Text(string)
        IL_00d6: nop          
        IL_00d7: leave.s      IL_00f1
      } // end of .try
      catch [mscorlib]System.Exception
      {

        IL_00d9: stloc.3      // V_3
        IL_00da: ldarg.0      // this
        IL_00db: ldc.i4.s     -2 // 0xfe
        IL_00dd: stfld        int32 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>1__state'
        IL_00e2: ldarg.0      // this
        IL_00e3: ldflda       valuetype [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>t__builder'
        IL_00e8: ldloc.3      // V_3
        IL_00e9: call         instance void [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder::SetException(class [mscorlib]System.Exception)
        IL_00ee: nop          
        IL_00ef: leave.s      IL_0105
      } // end of catch

      // [28 9 - 28 10]
      IL_00f1: ldarg.0      // this
      IL_00f2: ldc.i4.s     -2 // 0xfe
      IL_00f4: stfld        int32 AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>1__state'

      IL_00f9: ldarg.0      // this
      IL_00fa: ldflda       valuetype [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder AsyncCallWinForm.Form1/'<button1_Click>d__1'::'<>t__builder'
      IL_00ff: call         instance void [mscorlib]System.Runtime.CompilerServices.AsyncVoidMethodBuilder::SetResult()
      IL_0104: nop          
      IL_0105: ret          

    } // end of method '<button1_Click>d__1'::MoveNext
```

让我们再来开动下粗糙的大脑，试图想明白 IL 指令都做了什么。

```Java
    int32 V_0；
    System.Runtime.CompilerServices.TaskAwaiter<string> V_1;
    AsyncCallWinForm.Form1/'<button1_Click>d__1' V_2;
    mscorlib]System.Exception V_3;

    try
    {
        V_0 = this.state
        if (state == 0) goto label1;
        goto label2;
        label1: goto label3;

        label2: no operation;
        this.refer_to_form.textBox1.set_Text(String::Concat("#", Form1::GetThreadInfo(), "# 获取图像中.....")); // 一大段代码简化成这么一句

        V_1 = this.refer_to_from.LoadImage().GetAwaiter();
        V_1.get_IsCompleted();
        if completed is true, goto label4;

        this.state = 0;
        this.u_1 = V_1;

        this.t_builder.AwaitUnsafeOnCompleted(this.u_1, this);
        jump to label5;
        
        label3:
        this.u_1 = V_1;

        label4:
        this.s__2 = V_1.GetResult();
        this.'<sh>5__1' = this.s__2;
        this.refer_to_form.textBox1.set_Text(String::Concat("#", this.'<sh>5__1', "# 已获得图像")); // 一大段代码简化成这么一句
    }
    catch(System.Exception ex)
    {
        this.state = -2;
        this.t_buidler.SetException(ex);
        jump to label5;
    }

    this.state = -2;
    this.t_buidler.SetResult(); 

    label5: return;
```

转成方便人类阅读的代码如下：

```Java
    #class button1_Click_stateMachine#
    public void MoveNext() 
    {
        try
        {
            switch(this.state)
            {
                case -1:
                    this.refer_to_form.textBox1.Text = String::Concat("#", Form1::GetThreadInfo(), "# 获取图像中....."));
                    this.awaiter = this.refer_to_from.LoadImage().GetAwaiter(); // typeof TaskAwaiter<string>
                    if (this.IsCompleted)
                        goto label1;
                    this.state = 0;
                    this.builder.AwaitUnsafeOnCompleted(this.awaiter, this);
                    return;
                case 0:
                    goto label1;
                    break;
                default:
                    break;
            }
            label1:
            this.state = -2;
            this.result = this.awaiter.GetResult();
            this.refer_to_form.textBox1.Text = $"#{this.result}# 已获得图像";
            this.t_buidler.SetResult(); 
        }
        catch(Exception ex)
        {
            this.state = -2;
            this.buidler.SetException(ex);
        }
    }
```

这里就变得很有意思了，button1_Click 的代码被编译器完全改变了模样，一方面被 AsyncTaskBuilder::Start() 消去了 async，另一方面函数的逻辑被分成了两半，转化成了 StateMachine 的不同状态逻辑代码，消去了await。 不得不佩服 .net 设计人员如此奇妙的设计。

StateMachine 根据 state 的值进入不同的状态逻辑。

1. 当 LoadImage 没有立即完成时， 将状态置成0，并且通知 awaiter 等待完成。 函数就直接返回了。 这就是为什么不会阻塞界面，因为 UI 线程的执行任务到这里就暂时告一段落，可以去响应用户的其他操作。
2. 什么时候会再次进入状态机，并进入 state == 0 的逻辑分支呢？

继续查看 AsyncTaskMethodBuilder::AwaitUnsafeOnCompleted 这里实际调用的是AsyncTaskMethodBuilder&lt;VoidTaskResult&gt;::AwaitUnsafeOnCompleted

```Java
    public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>(
            ref TAwaiter awaiter, ref TStateMachine stateMachine)
            where TAwaiter : ICriticalNotifyCompletion
            where TStateMachine : IAsyncStateMachine
    {
        IAsyncStateMachineBox box = GetStateMachineBox(ref stateMachine);

        // The null tests here ensure that the jit can optimize away the interface
        // tests when TAwaiter is a ref type.

        if ((null != (object)default(TAwaiter)) && (awaiter is ITaskAwaiter))
        {
            ref TaskAwaiter ta = ref Unsafe.As<TAwaiter, TaskAwaiter>(ref awaiter); // relies on TaskAwaiter/TaskAwaiter<T> having the same layout
            TaskAwaiter.UnsafeOnCompletedInternal(ta.m_task, box, continueOnCapturedContext: true);
        }
        else if ((null != (object)default(TAwaiter)) && (awaiter is IConfiguredTaskAwaiter))
        {
            ref ConfiguredTaskAwaitable.ConfiguredTaskAwaiter ta = ref Unsafe.As<TAwaiter, ConfiguredTaskAwaitable.ConfiguredTaskAwaiter>(ref awaiter);
            TaskAwaiter.UnsafeOnCompletedInternal(ta.m_task, box, ta.m_continueOnCapturedContext);
        }
        else if ((null != (object)default(TAwaiter)) && (awaiter is IStateMachineBoxAwareAwaiter))
        {
            try
            {
                ((IStateMachineBoxAwareAwaiter)awaiter).AwaitUnsafeOnCompleted(box);
            }
            catch (Exception e)
            {
                // Whereas with Task the code that hooks up and invokes the continuation is all local to corelib,
                // with ValueTaskAwaiter we may be calling out to an arbitrary implementation of IValueTaskSource
                // wrapped in the ValueTask, and as such we protect against errant exceptions that may emerge.
                // We don't want such exceptions propagating back into the async method, which can't handle
                // exceptions well at that location in the state machine, especially if the exception may occur
                // after the ValueTaskAwaiter already successfully hooked up the callback, in which case it's possible
                // two different flows of execution could end up happening in the same async method call.
                AsyncMethodBuilderCore.ThrowAsync(e, targetContext: null);
            }
        }
        else
        {
            // The awaiter isn't specially known. Fall back to doing a normal await.
            try
            {
                awaiter.UnsafeOnCompleted(box.MoveNextAction);
            }
            catch (Exception e)
            {
                AsyncMethodBuilderCore.ThrowAsync(e, targetContext: null);
            }
        }
    }
```
    
首先创建了一个 StateMachine 的包裹器，然后根据当前情况得知进入了 TaskAwaiter.UnsafeOnCompletedInternal(ta.m_task, box, continueOnCapturedContext: true);

```Java
    internal static void TaskAwaiter::UnsafeOnCompletedInternal(Task task, IAsyncStateMachineBox stateMachineBox, bool continueOnCapturedContext)
    {
        Debug.Assert(stateMachineBox != null);

        // If TaskWait* ETW events are enabled, trace a beginning event for this await
        // and set up an ending event to be traced when the asynchronous await completes.
        if (TplEtwProvider.Log.IsEnabled() || Task.s_asyncDebuggingEnabled)
        {
            task.SetContinuationForAwait(OutputWaitEtwEvents(task, stateMachineBox.MoveNextAction), continueOnCapturedContext, flowExecutionContext: false);
        }
        else
        {
            task.UnsafeSetContinuationForAwait(stateMachineBox, continueOnCapturedContext);
        }
    }

    internal void Task::UnsafeSetContinuationForAwait(IAsyncStateMachineBox stateMachineBox, bool continueOnCapturedContext)
    {
        Debug.Assert(stateMachineBox != null);

        // If the caller wants to continue on the current context/scheduler and there is one,
        // fall back to using the state machine's delegate.
        if (continueOnCapturedContext)
        {
            SynchronizationContext syncCtx = SynchronizationContext.Current;
            if (syncCtx != null && syncCtx.GetType() != typeof(SynchronizationContext))
            {
                var tc = new SynchronizationContextAwaitTaskContinuation(syncCtx, stateMachineBox.MoveNextAction, flowExecutionContext: false);
                if (!AddTaskContinuation(tc, addBeforeOthers: false))
                {
                    tc.Run(this, canInlineContinuationTask: false);
                }
                return;
            }
            else
            {
                TaskScheduler scheduler = TaskScheduler.InternalCurrent;
                if (scheduler != null && scheduler != TaskScheduler.Default)
                {
                    var tc = new TaskSchedulerAwaitTaskContinuation(scheduler, stateMachineBox.MoveNextAction, flowExecutionContext: false);
                    if (!AddTaskContinuation(tc, addBeforeOthers: false))
                    {
                        tc.Run(this, canInlineContinuationTask: false);
                    }
                    return;
                }
            }
        }

        // Otherwise, add the state machine box directly as the ITaskCompletionAction continuation.
        // If we're unable to because the task has already completed, queue the delegate.
        if (!AddTaskContinuation(stateMachineBox, addBeforeOthers: false))
        {
            AwaitTaskContinuation.UnsafeScheduleAction(stateMachineBox.MoveNextAction, this);
        }
    }

    private bool AddTaskContinuation(object tc, bool addBeforeOthers)
    {
        Debug.Assert(tc != null);

        // Make sure that, if someone calls ContinueWith() right after waiting for the predecessor to complete,
        // we don't queue up a continuation.
        if (IsCompleted) return false;

        // Try to just jam tc into m_continuationObject
        if ((m_continuationObject != null) || (Interlocked.CompareExchange(ref m_continuationObject, tc, null) != null))
        {
            // If we get here, it means that we failed to CAS tc into m_continuationObject.
            // Therefore, we must go the more complicated route.
            return AddTaskContinuationComplex(tc, addBeforeOthers);
        }
        else return true;
    }
```
    
此处continueOnCapturedContext为true，并且由于是 UI 线程发起，SynchronizationContext.Current 也不为空。 所以进入了第一个分支。 在将 StateMachineBox 包裹成 SynchronizationContextAwaitTaskContinuation 这个完成种子，然后加入到 LoadImage Task 的完成队列。 如果这时任务已经完成，那么直接运行完成种子。 直接运行的情况后面再说，我们先看看加入之后的情况。

当 LoadImage Task 没有完成时，界面将一直显示“获取图像中....."。 直到5秒过去， task 进入了 Finish 状态。 关于 Task Finish 不想再贴一大堆代码，总之经过层层的处理，在它行将退场之前将会拆开其他任务送给他的礼物（如果有的话）：一堆完成种子。 最终它会走到这么一步：

```Java
    switch (currentContinuation)
    {
        case IAsyncStateMachineBox stateMachineBox:
            AwaitTaskContinuation.RunOrScheduleAction(stateMachineBox, canInlineContinuations);
            break;

        case Action action:
            AwaitTaskContinuation.RunOrScheduleAction(action, canInlineContinuations);
            break;

        case TaskContinuation tc:
            tc.Run(this, canInlineContinuations);
            break;

        default:
            Debug.Assert(currentContinuation is ITaskCompletionAction);
            RunOrQueueCompletionAction((ITaskCompletionAction)currentContinuation, canInlineContinuations);
            break;
    }
```

此时，前面提到的 SynchronizationContextAwaitTaskContinuation 就要 run 起来了。

```Java
    internal sealed override void Run(Task task, bool canInlineContinuationTask)
    {
        // If we're allowed to inline, run the action on this thread.
        if (canInlineContinuationTask &&
            m_syncContext == SynchronizationContext.Current)
        {
            RunCallback(GetInvokeActionCallback(), m_action, ref Task.t_currentTask);
        }
        // Otherwise, Post the action back to the SynchronizationContext.
        else
        {
            TplEtwProvider etwLog = TplEtwProvider.Log;
            if (etwLog.IsEnabled())
            {
                m_continuationId = Task.NewId();
                etwLog.AwaitTaskContinuationScheduled((task.ExecutingTaskScheduler ?? TaskScheduler.Default).Id, task.Id, m_continuationId);
            }
            RunCallback(GetPostActionCallback(), this, ref Task.t_currentTask);
        }
        // Any exceptions will be handled by RunCallback.
    }

    private static ContextCallback GetPostActionCallback()
    {
        ContextCallback callback = s_postActionCallback;
        if (callback == null) { s_postActionCallback = callback = PostAction; } // lazily initialize SecurityCritical delegate
        return callback;
    }

    private static void PostAction(object state)
    {
        var c = (SynchronizationContextAwaitTaskContinuation)state;

        TplEtwProvider etwLog = TplEtwProvider.Log;
        if (etwLog.TasksSetActivityIds && c.m_continuationId != 0)
        {
            c.m_syncContext.Post(s_postCallback, GetActionLogDelegate(c.m_continuationId, c.m_action));
        }
        else
        {
            c.m_syncContext.Post(s_postCallback, c.m_action); // s_postCallback is manually cached, as the compiler won't in a SecurityCritical method
        }
    }

    protected void RunCallback(ContextCallback callback, object state, ref Task currentTask)
    {
        Debug.Assert(callback != null);
        Debug.Assert(currentTask == Task.t_currentTask);

        // Pretend there's no current task, so that no task is seen as a parent
        // and TaskScheduler.Current does not reflect false information
        var prevCurrentTask = currentTask;
        try
        {
            if (prevCurrentTask != null) currentTask = null;

            ExecutionContext context = m_capturedContext;
            if (context == null)
            {
                // If there's no captured context, just run the callback directly.
                callback(state);
            }
            else
            {
                // Otherwise, use the captured context to do so.
                ExecutionContext.RunInternal(context, callback, state);
            }
        }
        catch (Exception exc) // we explicitly do not request handling of dangerous exceptions like AVs
        {
            ThrowAsyncIfNecessary(exc);
        }
        finally
        {
            // Restore the current task information
            if (prevCurrentTask != null) currentTask = prevCurrentTask;
        }
    }
```

最后 Task 会通过 SynchronizationContext 将 callback post 给 UI 线程。  此处的 callback 就是 stateMachineBox.MoveNextAction。 我们就回到了 state == 0 那一幕。 MoveNext 进入下一步，得到返回值后顺利输出。

至此，我们看到为了实现 async/await 这对语法糖，编译器以及 TPL 框架都做出了哪些努力。 通过这一番探索，我们能够更加清楚的了解到一些原则的必要性。

1. 异步要一直异步到底
2. 异步操作不要调用同步阻塞，这是经常造成死锁的根源。
3. 用 Task.Delay 来替代 Thread.Sleep 等等
4. 什么时候 ConfigureAwait 的参数 continueOnCapturedContext 必须要为 true

PS: 关于 Task 是如何运行的，SynchronizationContext 以及 ExecutionContext 又是怎么回事，则要另起几章才能讲得清楚。
