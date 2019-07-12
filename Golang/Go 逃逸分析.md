# Go 逃逸分析

## 堆和栈

要理解什么是逃逸分析会涉及堆和栈的一些基本知识，如果忘记的同学我们可以简单的回顾一下：

- 堆（Heap）：一般来讲是人为手动进行管理，手动申请、分配、释放。堆适合不可预知大小的内存分配，这也意味着为此付出的代价是分配速度较慢，而且会形成内存碎片。
- 栈（Stack）：由编译器进行管理，自动申请、分配、释放。一般不会太大，因此栈的分配和回收速度非常快；我们常见的函数参数（不同平台允许存放的数量不同），局部变量等都会存放在栈上。

栈分配内存只需要两个CPU指令：“PUSH”和“RELEASE”，分配和释放；而堆分配内存首先需要去找到一块大小合适的内存块，之后要通过垃圾回收才能释放。

通俗比喻的说，`栈`就如我们去饭馆吃饭，只需要点菜（发出申请）--》吃吃吃（使用内存）--》吃饱就跑剩下的交给饭馆（操作系统自动回收），而`堆`就如在家里做饭，大到家，小到买什么菜，每一个环节都需要自己来实现，但是自由度会大很多。



## 什么是逃逸分析

在编译程序优化理论中，逃逸分析是一种确定指针动态范围的方法，简单来说就是分析在程序的哪些地方可以访问到该指针。

再往简单的说，Go是通过在编译器里做逃逸分析（escape analysis）来决定一个对象放栈上还是放堆上，不逃逸的对象放栈上，可能逃逸的放堆上；即我发现`变量`在退出函数后没有用了，那么就把丢到栈上，毕竟栈上的内存分配和回收比堆上快很多；反之，函数内的普通变量经过`逃逸分析`后，发现在函数退出后`变量`还有在其他地方上引用，那就将`变量`分配在堆上。做到按需分配（哪里的人民需要我，我就往哪去~~，一个党员的呐喊）。



## 为何需要逃逸分析

ok，了解完`堆`和`栈`各自的优缺点后，我们就可以更好的知道`逃逸分析`存在的目的了：

1. 减少`gc`压力，栈上的变量，随着函数退出后系统直接回收，不需要`gc`标记后再清除。
2. 减少内存碎片的产生。
3. 减轻分配堆内存的开销，提高程序的运行速度。



## 如何确定是否逃逸

在`Go`中通过逃逸分析日志来确定变量是否逃逸，开启逃逸分析日志：

```shell
go run -gcflags '-m -l' main.go
```

- `-m` 会打印出逃逸分析的优化策略，实际上最多总共可以用 4 个 `-m`，但是信息量较大，一般用 1 个就可以了。
- `-l` 会禁用函数内联，在这里禁用掉`内联`能更好的观察逃逸情况，减少干扰。



## 逃逸案例

### 案例一：取地址发生逃逸

```go
package main

type UserData struct {
	Name  string
}

func main() {
	var info UserData
	info.Name = "WilburXu"
	_ = GetUserInfo(info)
}

func GetUserInfo(userInfo UserData) *UserData {
	return &userInfo
}
```

执行 `go run -gcflags '-m -l' main.go` 后返回以下结果：

```shell
# command-line-arguments
.\main.go:14:9: &userInfo escapes to heap
.\main.go:13:18: moved to heap: userInfo
```

> GetUserInfo函数里面的变量 `userInfo` 逃到堆上了（分配到堆内存空间上了）。
>
> GetUserInfo 函数的返回值为 *UserData 指针类型，然后 将值变量`userInfo` 的地址返回，此时编译器会判断该值可能会在函数外使用，就将其分配到了堆上，所以变量`userInfo`就逃逸了。

#### 优化方案

```go
func main() {
	var info UserData
	info.Name = "WilburXu"
	_ = GetUserInfo(&info)
}

func GetUserInfo(userInfo *UserData) *UserData {
	return userInfo
}
```

```shell
# command-line-arguments
.\main.go:13:18: leaking param: userInfo to result ~r1 level=0
.\main.go:10:18: main &info does not escape
```

对一个变量取地址，可能会被分配到堆上。但是编译器进行逃逸分析后，如果发现到在函数返回后，此变量不会被引用，那么还是会被分配到栈上。套个取址符，就想骗补助？Too young！



### 案例二 





## 参考文章

[Golang escape analysis](http://www.agardner.me/golang/garbage/collection/gc/escape/analysis/2015/10/18/go-escape-analysis.html)



## 总结

不要盲目使用变量的指针作为函数参数，虽然它会减少复制操作。但其实当参数为变量自身的时候，复制是在栈上完成的操作，开销远比变量逃逸后动态地在堆上分配内存少的多。





