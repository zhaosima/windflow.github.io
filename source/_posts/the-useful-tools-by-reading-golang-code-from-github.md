---
title: the useful things learned from reading github golang code 
date: 2019-08-16 09:26:16
tags:
---

# 从 Github Golang 开源库中学到的知识

## cmdline prompt
### cobra
cobra [github.com/spf13/cobra](https://github.com/spf13/cobra) 支持多级菜单，命令建议，建立文档，使用说明等。


## DI
### wire

wire [github.com/google/wire](https://github.com/google/wire) 省去了繁琐的调用构造函数的生成最终对象的过程，把你从无意义的劳动中解脱出来。

## Go routine
### async
async [github.com/studiosol/async](https://github.com/studiosol/async) 非常轻量的并发库
学到的点：
1. context 的作用:
context.Context 提供了一种幽雅地中断，收尾程序的机制。
2. channel selector 机制:
select 在某一channel case满足时退出，如果同时有channel case到达，随机选择一个。 
3. channel closed:
writeable channel closed 之后，仍然可以被接收。可通过 if x,ok <- somechannel; ok 来判断是否关闭。




















---
## 具体例子
### corba
``` golang
func Execute() {
	var echoTimes int

	var cmdPrint = &cobra.Command{
		Use:   "print [string to print]",
		Short: "Print anything to the screen",
		Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
		Args: cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("Print: " + strings.Join(args, " "))
		},
	}

	var cmdEcho = &cobra.Command{
		Use:   "echo [string to echo]",
		Short: "Echo anything to the screen",
		Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
		Args: cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("Print: " + strings.Join(args, " "))
		},
	}

	var cmdTimes = &cobra.Command{
		Use:   "times [string to echo]",
		Short: "Echo anything to the screen more times",
		Long: `echo things multiple times back to the user by providing
a count and a string.`,
		Args: cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
			for i := 0; i < echoTimes; i++ {
				fmt.Println("Echo: " + strings.Join(args, " "))
			}
		},
	}

	cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")

	var subCmd = &cobra.Command{
		Use:   "sub [no options!]",
		Short: "My subcommand",
		PreRun: func(cmd *cobra.Command, args []string) {
			fmt.Printf("Inside subCmd PreRun with args: %v\n", args)
		},
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Printf("Inside subCmd Run with args: %v\n", args)
		},
		PostRun: func(cmd *cobra.Command, args []string) {
			fmt.Printf("Inside subCmd PostRun with args: %v\n", args)
		},
		PersistentPostRun: func(cmd *cobra.Command, args []string) {
			fmt.Printf("Inside subCmd PersistentPostRun with args: %v\n", args)
		},
	}

	var rootCmd = &cobra.Command{Use: "app"}
	rootCmd.AddCommand(cmdPrint, cmdEcho, subCmd)
	cmdEcho.AddCommand(cmdTimes)

	err := doc.GenMarkdownTree(rootCmd, "./")
	if err != nil {
		fmt.Println(err.Error())
	}

	rootCmd.Execute()

}
```

### wire
1. 按照[官方例子](https://github.com/google/wire/blob/master/_tutorial/README.md)建立 mian.go wire.go
2. 下载wire: 
``` bash
go get github.com/google/wire/cmd/wire
```
3. 编译出 wire executable
4. wire main.go wire.go, 不出意外将得到wire_gen.go
5. go build 得到编译的最终结果


### context
``` golang
func dowait() {
	var count = 7
	var wg sync.WaitGroup // use default value is ok
	wg.Add(count)

	ctx, cancel := context.WithCancel(context.Background())

	for i := 0; i < count; i++ {
		var k = i
		go func() {
			defer wg.Done()
			d := k * int(time.Second)
			select {
			case <-ctx.Done(): // 给一个优雅地结束任务的机会
				fmt.Println("context done", k)
				return
			case <-time.After(time.Duration(d)):
				fmt.Println("work done", k)
			}
		}()
	}

	go func() {
		wg.Wait()
		cancel()
		fmt.Println("wait over")
	}()

	go func() {
		<-time.After(3 * time.Second)
		cancel()
	}()
}
```
