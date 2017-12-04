---
title: agreement better than config
date: 2017-09-13 10:54:24
tags:
---
## 如何做到约定优于配置

不同于.net framework的asp.net项目，asp.net core的启动入口直接就是Program.cs，就好像在写一个Console。启动代码非常的简洁，只有这么几句。

        public static void Main(string[] args)
        {
            BuildWebHost(args).Run();
        }
    
        public static IWebHost BuildWebHost(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .UseStartup<Startup>()
                .Build();

粗看上去没什么花样，但是请注意这里的StartUp。 查看它的定义，你会惊奇的发现它没有继承任何接口，但是提供了两个Config方法用于初始化Service和App。

那么这两个方法是怎么被调用的呢？ 带着这个疑问，我去翻看了 Microsoft.AspNetCore.Hosting 的[源码](https://github.com/aspnet/Hosting)。对于代码的细节就不做详细描述了，其内部思想如下图
![](../img/aibtc.png) 


上面贴出的启动代码体现了一种思想，约定好于配置。只要一个类实现了某些方法，它就可以被作为某种接口被调用。是不是有点像Golang的duck type。

最后，我也写了一段模仿实现的代码，在[这里](https://github.com/simazhao/show-me-the-code/tree/master/dotnet/core/AgreementBetterThanConfig)。