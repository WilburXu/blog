# Go Exec 僵尸与孤儿进程

最近，使用 `golang` 去管理本地应用的生命周期，期间有几个有趣的点，今天就一起看下。



## 场景一

我们来看看下面两个脚本会产生什么问题：

### 创建两个 shell 脚本

- **start.sh**

```sh
#!/bin/sh
sh sub.sh
```

- **sub.sh**

```sh
#!/bin/sh
n=0
while [ $n -le 100 ]
do
  echo $n
  let n++
  sleep 1
done
```

### 执行脚本

#### 输出结果

```sh
$ ./start.sh 
0
1
2
...
```

#### 进程关系

**查看进程信息**

```sh
ps -j

USER   PID    PPID   PGID   SESS  JOBC  STAT   TT     TIME     COMMAND
root   31758  31346  31758  0     1     S+     s000   0:00.00  /bin/sh ./start.sh
root   31759  31758  31758  0     1     S+     s000   0:00.01  sh sub.sh
```

- `sub.sh` 的 父进程（PPID）为 `start.sh` 的进程id（PID）

- `sub.sh` 和 `start.sh` 两个进程的 `PGID` 是同一个，（ 属一个进程组）。



### 删除 `start.sh` 的进程

```sh
kill -9 31758

# 再查看进程组
ps -j

## 返回
USER     PID       PPID  PGID     SESS  JOBC   STAT    TT       TIME     COMMAND
root     31759     1     31758    0      0     S       s000     0:00.03  sh sub.sh
```

- `start.sh` 进程不在了
- `sub.sh` 进程还在执行
- `sub.sh` 进程的 `PID` 变成了 **1**

### 问题1：

**那`sub.sh` 这个进程现在属于什么？**



## 场景二

假设`sub.sh` 是实际的应用，  `start.sh` 是应用的启动脚本。

那么，`golang` 是如何管理他们的呢？ 我们继续看看下面 关于`golang`的场景。



在上面**两个脚本**的基础上，我们用`golang` 的 `os/exec`库去调用 `start.sh`脚本

```go
package main

import (
	"context"
	"log"
	"os"
	"os/exec"
	"time"
)

func main()  {
	cmd := exec.CommandContext(context.Background(), "./start.sh")

  // 将 start.sh 和 sub.sh 移到当前目录下
	cmd.Dir = "/Go/src/go-code/cmd/"
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Start(); err != nil {
		log.Printf("cmd.Start error %+v \n", err)
	}

	for {
		select {
		default:
			log.Println(cmd.Process.Pid)
			time.Sleep(2 * time.Second)
		}
	}
}
```

### 执行程序

```sh
go run ./main.go
```



### 查看进程

```sh
ps -j

USER   PID    PPID   PGID     SESS  JOBC  STAT   TT      TIME     COMMAND
root   45458  45457  45457    0     0     Ss+    s004    0:00.03  ...___1go_build_go_code_cmd
root   45462  45458  45457    0     0     S+     s004    0:00.01  /bin/sh ./start.sh
root   45463  45462  45457    0     0     S+     s004    0:00.03  sh sub.sh
```

发现 `go` 、 `start.sh` 、`sub.sh` 三个进程为同一个进程组（同一个 PGID）

父子关系为： `main.go` -> `start.sh` -> `sub.sh` 



### 删除 `start.sh` 的进程

实际场景，有可能启动程序挂了，导致我们无法监听到执行程序的情况，删除`start.sh`进程，模拟下场景 ：

```sh 
kill -9 45462
```

#### 再查看进程

```sh
ps -j

USER   PID    PPID   PGID     SESS  JOBC  STAT   TT      TIME     COMMAND
root   45458  45457  45457    0     0     Ss+    s004    0:00.03  ...___1go_build_go_code_cmd
root   45462  1      45457    0     0     S+     s004    0:00.01  (bash)
root   45463  45462  45457    0     0     S+     s004    0:00.03  sh sub.sh
```

- 发现没， `start.sh` 的 `PPID`  为1
- 即使 `start.sh` 的 `PPID`变成了1 ，`log.Println(cmd.Process.Pid)` 还持续的输出 .

### 问题2:

那如果 `PPID`为1 ，`golang`程序不就无法管理了吗？ 即使 sub.sh 退出也不知道了，那要如何处理？



## 问题分析

- 两个场景中， 都有一个共同的点，就是 `PPID` 为1，这妥妥的成为没人要的娃了——`孤儿进程`

- 场景二中，如果 `cmd`的没有进程没有被回收，`go`程序也无法管理，那么`start.sh`就成为了占着茅坑不拉屎的子进程——`僵尸进程`

那究竟什么是`孤儿进程` 和 `僵尸进程`？

### 孤儿进程

在类 `UNIX` 操作系统中，**孤儿进程（Orphan Process）**指：是在其父进程执行完成或被终止后仍继续运行的一类进程。

为避免孤儿进程退出时无法释放所占用的资源而僵死，任何孤儿进程产生时都会立即为系统进程 `init` 或 `systemd` 自动接收为子进程，这一过程也被称为`收养`。在此需注意，虽然事实上该进程已有`init`作为其父进程，但由于创建该进程的进程已不存在，所以仍应称之为`孤儿进程`。孤儿进程**会浪费服务器的资源，甚至有耗尽资源的潜在危险**。

#### 解决&预防

1. 终止机制：强制杀死孤儿进程（最常用的手段）；

2. 再生机制：服务器在指定时间内查找调用的客户端，若找不到则直接杀死孤儿进程；

3. 超时机制：给每个进程指定一个确定的运行时间，若超时仍未完成则强制终止之。若有需要，亦可让进程在指定时间耗尽之前申请延时。

4. 进程组：因为父进程终止或崩溃都会导致对应子进程成为孤儿进程，所以也无法预料一个子进程执行期间是否会被“遗弃”。有鉴于此，多数类UNIX系统都引入了进程组以防止产生孤儿进程。



### 僵尸进程

在类 `UNIX` 操作系统中，**僵尸进程（zombie process）**指：完成执行（通过exit系统调用，或运行时发生致命错误或收到终止信号所致），但在操作系统的进程表中仍然存在其进程控制块，处于"终止状态"的进程。
正常情况下，进程直接被其父进程 `wait` 并由系统回收。而僵尸进程与正常进程不同，`kill` 命令对僵尸进程无效，并且无法回收，从而导致**资源泄漏**。

#### 解决&预防

收割僵尸进程的方法是通过 `kill` 命令手工向其父进程发送SIGCHLD信号。如果其父进程仍然拒绝收割僵尸进程，则终止父进程，使得 `init` 进程收养僵尸进程。`init` 进程周期执行 `wait` 系统调用收割其收养的所有僵尸进程。

### 查看进程详情

````sh
# 列出进程
ps -l
````

- USER：进程的所属用户
- PID：进程的进程ID号 
- RSS：进程占用的固定的内存量 (Kbytes)
- **S：查看进程状态**
- CMD：进程对应的实际程序



#### 进程状态（S）
- R：运行 Runnable (on run queue) 正在运行或在运行队列中等待
- S：睡眠 Sleeping 休眠中，受阻，在等待某个条件的形成或接受到信号
- I：空闲 Idle 
- **Z：僵死 Zombie（a defunct process) 进程已终止，但进程描述符存在， 直到父进程调用wait4()系统调用后释放**
- D：不可中断 Uninterruptible sleep (ususally IO) 收到信号不唤醒和不可运行， 进程必须等待直到有中断发生
- T：终止 Terminate 进程收到SIGSTOP、SIGSTP、 SIGTIN、SIGTOU信号后停止运行运行
- P：等待交换页
- W：无驻留页 has no resident pages 没有足够的记忆体分页可分配
- X：死掉的进程 



## Go解决方案

采用 杀掉进程组（kill process group，而不是只 kill 父进程，在 Linux 里面使用的是 `kill -- -PID`） 与 进程wait方案，结果如下：

```go
package main

import (
	"context"
	"log"
	"os"
	"os/exec"
	"syscall"
	"time"
)

func main() {

	ctx := context.Background()
	cmd := exec.CommandContext(ctx, "./start.sh")
  
  // 设置进程组
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Setpgid: true,
	}

	cmd.Dir = "/Users/Wilbur/Project/Go/src/go-code/cmd/"
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Start(); err != nil {
		log.Printf("cmd.Start error %+v \n", err)
	}

  // 监听进程wait
	errCmdCh := make(chan error, 1)
	go func() {
		errCmdCh <- cmd.Wait()
	}()

	for {
		select {
		case <-ctx.Done():
			log.Println("ctx.done")
			pid := cmd.Process.Pid
			if err := syscall.Kill(-1*pid, syscall.SIGKILL); err != nil {
				return
			}
		case err := <-errCmdCh:
			log.Printf("errCmdCh error %+v \n", err)
			return
		default:
			log.Println(cmd.Process.Pid)
			time.Sleep(2 * time.Second)
		}
	}
}
```



剖析 `cmd.Wait()` 源码

在 `os/exec_unix`下： 

```go
var (
		status syscall.WaitStatus
		rusage syscall.Rusage
		pid1   int
		e      error
	)

for {
		pid1, e = syscall.Wait4(p.Pid, &status, 0, &rusage)
		if e != syscall.EINTR {
			break
		}
}
```

进行了 `syscall.Wait4`对系统监听，正如"僵死 Zombie（a defunct process) 进程已终止，但进程描述符存在， 直到父进程调用wait4()系统调用后释放"，所说一致。



## 总结

严格地来说，僵尸进程并不是问题的根源，罪魁祸首是产生出大量僵尸进程的那个父进程。

因此，当我们寻求如何消灭系统中大量的僵尸进程时，更应该是在实际的开发过程中，思考如何避免僵尸进程的产生。



## 参考：

https://pkg.go.dev/syscall

https://cs.opensource.google/go/go/+/refs/tags/go1.17.7:src/syscall/syscall_linux.go;l=279

https://pkg.go.dev/os/exec
