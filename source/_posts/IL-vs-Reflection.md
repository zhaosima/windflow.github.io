---
title: IL vs Reflection
date: 2018-04-17 10:19:29
tags:
---
## IL vs Reflection

最近在阅读某CQRS开源项目的源代码时，作者通过IL来生成绑定到interface具体子类的方法的调用，堪称巧妙。当然通过反射也可以实现，这么做的原因大概是为了效率。那么我来做个简单的测试，看效率差别有多大.

假设我们有个打招呼的程序

```Java
    class Aloha
    {
        public string Hello(string msg)
        {
            return string.Format("Hello, {0}", msg);
        }
    }
```

那么通过反射来打招呼是这样的

```Java
        private MethodInfo hello = typeof(Aloha).GetMethod("Hello");

        public string SayHello(Aloha hi, string m)
        {
            return hello.Invoke(hi, new[] { m }) as string;
        }
```

接着通过IL来回应是这样的

```Java
        public Func<string, string> SayHelloToo(Aloha hi)
        {
            var dm = new DynamicMethod("SayHelloToo", typeof(string), new[] {typeof(Aloha), typeof(string) }, typeof(Aloha));
            var gen = dm.GetILGenerator();

            gen.Emit(OpCodes.Ldarg_0);
            gen.Emit(OpCodes.Ldarg_1);
            gen.Emit(OpCodes.Call, hello);
            gen.Emit(OpCodes.Ret);

            var m = dm.CreateDelegate(typeof(Func<Aloha, string, string>)) as Func<Aloha, string, string>;

            return s => m(hi, s);
        }
```

接下来就让我们看看怎么调用

```Java
        var hi = new Aloha();
        var sayhelloFunc = SayHelloToo(hi);
        var loopCount = 10000000;

        Tune("For Reflect", m => SayHello(hi, m), loopCount);
        Tune("For IL     ", m => sayhelloFunc(m), loopCount);
        Tune("For Direct ", m => hi.Hello(m), loopCount);
```

调用结果

```m
    [For Reflect] loop 1000000 cost 00:00:00.4748191
    [For IL     ] loop 1000000 cost 00:00:00.1292819
    [For Direct ] loop 1000000 cost 00:00:00.1290007
    
    [For Reflect] loop 1000000 cost 00:00:00.4798441
    [For IL     ] loop 1000000 cost 00:00:00.1282383
    [For Direct ] loop 1000000 cost 00:00:00.1297988

    [For Reflect] loop 1000000 cost 00:00:00.5172965
    [For IL     ] loop 1000000 cost 00:00:00.1312012
    [For Direct ] loop 1000000 cost 00:00:00.1271685
```

可以看到效率差了4倍。 原因在于hi.Hello会被编译成相似SayHelloToo的IL代码，因此它们的执行效率相近。

注意，实际运用中不要每次都为相同场景生成IL代码，这样做反而非常影响效率，正确做法是缓存methodinfo或者type。
