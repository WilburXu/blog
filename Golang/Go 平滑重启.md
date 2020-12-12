# Go 平滑重启（优雅重启）

## 问题背景

生产环境重要且复杂，许多的操作需要在任何场景都要保证正常运行。

如果我们对线上服务进行更新的步骤如下：

1. `kill -9`服务
2. 再启动服务

那么将不可避免的出现以下两个问题：

- 未处理完的请求，被迫中断，数据一致性被破坏
- 新服务启动期间，请求无法进来，导致一段时间的服务不可用现象

一般有三种方案处理以上问题：

1. 生产环境会通过四层(lb)->七层(gateway)->服务，那么可以通过流量调度的方式实现平滑重启
2. k8s管理
3. 程序自身完成平滑重启（本章介绍）



## 什么事平滑重启

进程在不关闭其所监听端口的情况下进行重启，并且重启的整个过程保证所有请求都能被**正确**处理。

主要步骤：

1. 原进程（父进程）先`fork`一个子进程，同时让`fork`出来的子进程继承父进程所监听的`socket`。
2. 子进程完成初始化后，开始接收`socket`的请求。
3. 父进程停止接收新的请求，并将当下的请求处理完，等待连接空闲后，平滑退出。



## 信号（Signal）

服务的平滑重启，主要依赖进程接收的信号（实现进程间通信），这里简单的介绍`Golang`中信号的处理：

### 发送信号

- *kill*： **命令**允许用户发送一个特定的信号给进程
- *raise*： **库函数**可以发送特定的信号给当前进程

在Linux下运行`man kill`可以查看此命令的介绍和用法。

> **kill** -- terminate or signal a process
> The kill utility sends a signal to the processes specified by the pid operands.
> Only the super-user may send signals to other users' processes.



### 常用信号类型

信号的默认行为：

- term：信号终止进程
- core：产生核心转储文件并退出
- ignore：忽略信号
- stop：信号停止进程
- cont：信号恢复一个已停止的进程

| 信号    | 值       | 默认动作                                 | 说明                                                         |
| :------ | :------- | :--------------------------------------- | :----------------------------------------------------------- |
| SIGHUP  | 1        | Term                                     | HUP (hang up)：终端控制进程结束(终端连接断开)                |
| SIGINT  | 2        | Term                                     | INT (interrupt)：用户发送INTR字符(Ctrl+C)触发（强制进程结束） |
| SIGQUIT | 3        | Core                                     | QUIT (quit)：用户发送QUIT字符(Ctrl+/)触发（进程结束）        |
| SIGKILL | 9        | Term                                     | KILL (non-catchable, non-ignorable kill)：无条件结束程序(不能被捕获、阻塞或忽略) |
| SIGUSR1 | 30,10,16 | Term                                     | 用户自定义信号1                                              |
| SIGUSR2 | 31,12,17 | Term                                     | 用户自定义信号2                                              |
| SIGKILL | 15       | KILL (non-catchable, non-ignorable kill) | TERM (software termination signal)：程序终止信号             |

### 信号接收测试

```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	sigs := make(chan os.Signal)
	signal.Notify(sigs, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGUSR1, syscall.SIGUSR2)

	// 监听所有信号
	log.Println("listen sig")
	signal.Notify(sigs)

	// 打印进程id
	log.Println("PID:", os.Getppid())


	s := <-sigs
	log.Println("退出信号", s)
}
```

```shell
go run main.go
## --> listen sig
## --> PID: 4604

kill -s HUP 4604
# --> Hangup: 1
```



## 实现案例

demo：

```go
func main() {
   sigs := make(chan os.Signal)
   signal.Notify(sigs, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGUSR1, syscall.SIGUSR2)

   // 监听所有信号
   log.Println("listen sig")
   signal.Notify(sigs)


   // 打印进程id
   log.Println("PID:", os.Getppid())
   go func() {
      for s := range sigs {
         switch s {
         case syscall.SIGHUP:
            log.Println("startNewProcess...")
            startNewProcess()
            log.Println("shutdownParentProcess...")
            shutdownParentProcess()
         case syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT:
            log.PrintLn("Program Exit...", s)
         case syscall.SIGUSR1:
            log.Println("usr1 signal", s)
         case syscall.SIGUSR2:
            log.Println("usr2 signal", s)
         default:
            log.Println("other signal", s)
         }
      }
   }()

   <-sigs
}
```



### 推荐组件

[Facebookarchive/grace](https://github.com/facebookarchive/grace)



## shutdown优雅退出

go 1.8.x后，golang在http里加入了shutdown方法，用来控制优雅退出。

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	s := http.NewServeMux()
	s.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(3 * time.Second)
		log.Println(w, "Hello world!")
	})
	server := &http.Server{
		Addr:    ":8090",
		Handler: s,
	}
	go server.ListenAndServe()

	listenSignal(context.Background(), server)
}

func listenSignal(ctx context.Context, httpSrv *http.Server) {
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)

	select {
	case <-sigs:
		log.Println("notify sigs")
		httpSrv.Shutdown(ctx)
		log.Println("http shutdown")
	}
}
```



## 小结

在平常的生产环境中，优雅的重启是一个不可缺少的环节，无论是在go进程层间，或者上层的服务流量调度层面，都有许多的方案，选择最适合团队，保证项目稳定才是最重要的。



## 参考文章

https://github.com/facebookarchive/grace

https://mp.weixin.qq.com/s/UVZKFmv8p4ghm8ICdz85wQ

