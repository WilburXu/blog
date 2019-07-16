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

对一个变量取地址，可能会被分配到堆上。但是编译器进行逃逸分析后，如果发现到在函数返回后，此变量不会被引用，那么还是会被分配到栈上。套个取址符，就想骗补助？

编译器傲娇的说：Too young，Too Cool...！



### 案例二 ：未确定类型

```go
package main

type User struct {
	name interface{}
}

func main() {
	name := "WilburXu"
	MyPrintln(name)
}

func MyPrintln(one interface{}) (n int, err error) {
	var userInfo = new(User)
	userInfo.name = one // 泛型赋值 逃逸咯
	return
}
```

执行 `go run -gcflags '-m -l' main.go` 后返回以下结果：

```shell
# command-line-arguments
./main.go:12:16: leaking param: one
./main.go:13:20: MyPrintln new(User) does not escape
./main.go:9:11: name escapes to heap
```

这里可能有同学会好奇，`MyPrintln`函数内并没有被引用的便利，为什么变了`name`会被分配到了`堆`上呢？

上一个案例我们知道了，普通的手法想去"骗取补助"，聪明灵利的编译器是不会“上当受骗的噢”；但是对于`interface`类型，很遗憾，编译器在编译的时候很难知道在函数的调用或者结构体的赋值过程会是怎么类型，因此只能分配到`堆`上。

### 优化方案

将结构体`User`的成员`name`的类型、函数`MyPringLn`参数`one`的类型改为 `string`，将得出：

```shell
# command-line-arguments
./main.go:12:16: leaking param: one
./main.go:13:20: MyPrintln new(User) does not escape
```

### 拓展分析

对于案例二的分析，我们还可以通过反编译命令`go tool compile -S main.go`查看，会发现如果为`interface`类型，main主函数在编译后会`额外`多出以下指令：

```shell
# main.go:9 -> MyPrintln(name)
	0x001d 00029 (main.go:9)	PCDATA	$2, $1
	0x001d 00029 (main.go:9)	PCDATA	$0, $1
	0x001d 00029 (main.go:9)	LEAQ	go.string."WilburXu"(SB), AX
	0x0024 00036 (main.go:9)	PCDATA	$2, $0
	0x0024 00036 (main.go:9)	MOVQ	AX, ""..autotmp_5+32(SP)
	0x0029 00041 (main.go:9)	MOVQ	$8, ""..autotmp_5+40(SP)
	0x0032 00050 (main.go:9)	PCDATA	$2, $1
	0x0032 00050 (main.go:9)	LEAQ	type.string(SB), AX
	0x0039 00057 (main.go:9)	PCDATA	$2, $0
	0x0039 00057 (main.go:9)	MOVQ	AX, (SP)
	0x003d 00061 (main.go:9)	PCDATA	$2, $1
	0x003d 00061 (main.go:9)	LEAQ	""..autotmp_5+32(SP), AX
	0x0042 00066 (main.go:9)	PCDATA	$2, $0
	0x0042 00066 (main.go:9)	MOVQ	AX, 8(SP)
	0x0047 00071 (main.go:9)	CALL	runtime.convT2Estring(SB)
```

对于`Go汇编语法`不熟悉的可以参考 [Golang汇编快速指南](https://studygolang.com/articles/2917)



### 案例三：“点点点”参数逃逸

```go
package main

func noescape(y ...interface{}){
}

func main() {
	x := 0 // BAD: x escapes
	noescape(&x)
}
```

执行 `go run -gcflags '-m -l -l' main.go`   （是两个`-l`噢）后返回以下结果：

```shell
# command-line-arguments
.\main.go:3:6: can inline noescape
.\main.go:6:6: can inline main
.\main.go:8:10: inlining call to noescape
.\main.go:3:15: noescape y does not escape
.\main.go:8:11: &x escapes to heap
.\main.go:8:11: &x escapes to heap
.\main.go:7:2: moved to heap: x
.\main.go:8:10: main []interface {} literal does not escape
```

先解释一下为什么要使用两个`-l` ，由于当前的场景比较特殊，变量`x`没有被引用，且生命周期也在`main`里，x没有逃逸，分配在栈上，但是我们通过分析，其实发现，如果不在`main`中，那么变量`x`是会逃逸的。这里不得不说，编译器真的太聪明了。

#### 优化方案

`y ...interface{}` 改为 `y interface{}`



### 案例四：间接赋值（Assignment to indirection escapes）

对某个引用类对象中的引用类成员进行赋值。Go 语言中的引用类数据类型有 `func`, `interface`, `slice`, `map`, `chan`, `*Type(指针)`。

```go
package main

type User struct {
	name interface{}
	age *int
}

func main() {
	var (
		userOne User
		userTwo = new(User)
	)
	userOne.name = "WilburXuOne"	// 不逃逸
	userTwo.name = "WilburXuTwo"	// 逃逸

	userOne.age = new(int)	// 不逃逸
	userTwo.age = new(int)	// 逃逸
}
```

执行 `go run -gcflags '-m -l' main.go` 后返回以下结果：

```shell
# command-line-arguments
.\main.go:14:17: "WilburXuTwo" escapes to heap
.\main.go:17:19: new(int) escapes to heap
.\main.go:11:16: main new(User) does not escape
.\main.go:13:17: main "WilburXuOne" does not escape
.\main.go:16:19: main new(int) does not escape
```

为什么这里`值`类型不会逃逸而`引用类型`会逃逸呢？这是因为在 `userTwo = new(User)` 对象的创建时，编译器先是分析`userTwo` 对象可能分配在`堆`上，同时成员变量 `name` 和 `age` 也为`引用类型`，为了保证不出现`栈`回收后，导致对象`userTwo`的成员值也被回收，所以`name`和`age`需要逃逸。

但是，如果`name`和`age`为值类型，那么编译器虽然初步分析`userTwo`会分配在`堆`上，但由于`main`主函数结束后，变量都会被回收，也就是说对象没有被其他引用，那么就都会分配在`栈`上，所以`name`和`age`没有发生逃逸。

#### 优化建议

尽量不要将`引用对象`赋值给`引用对象`。



## 总结

不要盲目使用变量的指针作为函数参数，虽然它会减少复制操作。但其实当参数为变量自身的时候，复制是在栈上完成的操作，开销远比变量逃逸后动态地在堆上分配内存少的多。

Go的编译器就如一个聪明的`孩子`一般，大多时候在逃逸分析问题上的处理都令人眼前一亮，但有时`闹性子`的时候处理也是非常粗糙的分析或完全放弃，毕竟这是孩子天性不是吗？ 所以也需要我们在编写代码的时候多多观察，多多留意了。



## 参考文章

http://www.agardner.me/golang/garbage/collection/gc/escape/analysis/2015/10/18/go-escape-analysis.html

https://segmentfault.com/a/1190000019234268

https://docs.google.com/document/d/1CxgUBPlx9iJzkz9JWkb6tIpTe5q32QDmz8l0BouG0Cw/preview#heading=h.3i6ywlgy4wrw

http://npat-efault.github.io/programming/2016/10/10/escape-analysis-and-interfaces.html


