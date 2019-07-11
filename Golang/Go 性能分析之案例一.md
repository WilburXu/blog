### Go 性能分析之案例一

# 思考

相信大家在实际的项目开发中会遇到这么一个事，有的程序员写的代码不仅bug少，而且性能高；而有的程序员写的代码能否流畅的跑起来，都是一个很大问题。
而我们今天要讨论的就是一个关于性能优化的案例分析。

# 案例分析
我们先来构造一些基础数据（长度为10亿的切片，并赋上值）：

```
var testData = GenerateData()

// generate billion slice data
func GenerateData() []int {
	data := make([]int, 1000000000)
	for key, _ := range data {
		data[key] = key % 128
	}

	return data
}

// get length
func GetDataLen() int {
	return len(testData)
}
```

## 案例一

```
// case one
func CaseSumOne(result *int) {
	data := GenerateData()
	for i := 0; i < GetDataLen(); i++ {
		*result += data[i]
	}
}
```

```
// case two
func CaseSumTwo(result *int) {
	data := GenerateData()
	dataLen := GetDataLen()
	for i := 0; i < dataLen; i++ {
		*result += data[i]
	}
}
```

### 执行结果

```
$ go test -bench=.
goos: windows
goarch: amd64
BenchmarkCaseSumOne-8                  1        7439749000 ns/op
BenchmarkCaseSumTwo-8                  1        2529266700 ns/op
PASS
ok      _/C_/go-code/perform/case-one   14.059s
```
### 问题分析
 - CaseSumTwo执行效率是CaseSumOne的2.94倍，快了近三倍，这是为什么呢？
 - 我想这个其实很容易猜到，这里有一个连续的函数调用“GetDataLen()”,
 我们来看下两个函数的汇编，做个简单的对比：
### 生成汇编
```
go tool compile -S main.go
```

#### 函数CaseSumOne

```
"".CaseSumOne STEXT size=83 args=0x4 locals=0xc
        0x0000 00000 (point.go:22)      TEXT    "".CaseSumOne(SB), $12-4
        ...
        // point.go:24 -> for i := 0; i < GetDataLen(); i++ 
        0x0021 00033 (point.go:24)      PCDATA  $2, $2
        0x0021 00033 (point.go:24)      PCDATA  $0, $1
        0x0021 00033 (point.go:24)      MOVL    "".result+16(SP), DX 
        0x0025 00037 (point.go:24)      XORL    BX, BX
        0x0027 00039 (point.go:24)      JMP     47
        0x0029 00041 (point.go:25)      MOVL    (CX)(BX*4), BP    // CX循环计数器
        0x002c 00044 (point.go:25)      ADDL    BP, (DX)
        0x002e 00046 (point.go:24)      INCL    BX // i++
        0x002f 00047 (point.go:24)      MOVL    "".testData+4(SB), BP // 栈指针寄存器
        0x0035 00053 (point.go:24)      CMPL    BX, BP
        0x0037 00055 (point.go:24)      JGE     65
        ...
        0x0045 00069 (point.go:25)      CALL    runtime.panicindex(SB)
        0x004c 00076 (point.go:22)      CALL    runtime.morestack_noctxt(SB)
        ...
```

#### 函数CaseSumTwo
```
"".CaseSumTwo STEXT size=83 args=0x4 locals=0xc
        0x0000 00000 (point.go:30)      TEXT    "".CaseSumTwo(SB), $12-4
        ...
        // point.go:32 -> dataLen := GetDataLen()
	    // point.go:33 -> for i := 0; i < dataLen; i++ {
        0x0021 00033 (point.go:32)      MOVL    "".testData+4(SB), DX
        0x0027 00039 (point.go:33)      PCDATA  $2, $2
        0x0027 00039 (point.go:33)      PCDATA  $0, $1
        0x0027 00039 (point.go:33)      MOVL    "".result+16(SP), BX
        0x002b 00043 (point.go:33)      XORL    BP, BP
        0x002d 00045 (point.go:33)      JMP     53
        0x002f 00047 (point.go:34)      MOVL    (AX)(BP*4), SI
        0x0032 00050 (point.go:34)      ADDL    SI, (BX)
        0x0034 00052 (point.go:33)      INCL    BP
        0x0035 00053 (point.go:33)      CMPL    BP, DX
        0x0037 00055 (point.go:33)      JGE     65
        ...
        0x0045 00069 (point.go:34)      CALL    runtime.panicindex(SB)
        0x004c 00076 (point.go:30)      CALL    runtime.morestack_noctxt(SB)
        ...
```

### 比较结论
不难发现主要的区别是在CaseSumOne中多了这么一行：
```
0x002f 00047 (point.go:24)      MOVL    "".testData+4(SB), BP
```
其实虽然只有一行，但是对于函数“GetDataLen”里需要调用的指令对CPU的消耗：

```
"".GetDataLen STEXT size=36 args=0x4 locals=0x0
        0x0000 00000 (point.go:17)      TEXT    "".GetDataLen(SB), $0-4 // 
        0x0000 00000 (point.go:17)      MOVL    TLS, CX
        0x0007 00007 (point.go:17)      MOVL    (CX)(TLS*2), CX
        0x000d 00013 (point.go:17)      CMPL    SP, 8(CX)
        0x0010 00016 (point.go:17)      JLS     29
        0x0012 00018 (point.go:17)      FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0012 00018 (point.go:17)      FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0012 00018 (point.go:17)      FUNCDATA        $3, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0012 00018 (point.go:18)      PCDATA  $2, $0
        0x0012 00018 (point.go:18)      PCDATA  $0, $0
        0x0012 00018 (point.go:18)      MOVL    "".testData+4(SB), AX // 寄存器寻址 AX = lenVAL
        0x0018 00024 (point.go:18)      MOVL    AX, "".~r0+4(SP)	// SP = AX = lenVal
        0x001c 00028 (point.go:18)      RET
        0x001d 00029 (point.go:18)      NOP
        0x001d 00029 (point.go:17)      PCDATA  $0, $-1
        0x001d 00029 (point.go:17)      PCDATA  $2, $-1
        0x001d 00029 (point.go:17)      CALL    runtime.morestack_noctxt(SB)	// 压栈
        ...
```
虽然，看似小小一行代码的区别，但是在指令级的角度上，进行了创建栈空间、压栈、寻址、赋值等一系列操作，况且这里进行了循环调用。
## 案例二

```
// case two
func CaseSumTwo(result *int) {
	data := GenerateData()
	dataLen := GetDataLen()
	for i := 0; i < dataLen; i++ {
		*result += data[i]
	}
}
```

```
// case three
func CaseSumThree(result *int) {
	data := GenerateData()
	dataLen := GetDataLen()
	tmp := *result
	for i:= 0; i < dataLen; i++ {
		tmp += data[i]
	}
	*result = tmp
}
```



### 执行结果

```
$ go test -bench=.
goos: windows
goarch: amd64
BenchmarkCaseSumTwo-8                  1        2529266700 ns/op
BenchmarkCaseSumThree-8                1        1657554600 ns/op
PASS
ok      _/C_/go-code/perform/case-one   8.2773
```

### 问题分析
 - 虽然对连续函数调用进行了优化，但是CaseSumThree对执行效率还是高于CaseSumTwo1.52倍，还有哪些情况会影响执行性能呢？
我们再来对比下“CaseSumTwo”和“CaseSumThree”对汇编源码：

### 生成汇编
```
go tool compile -S main.go
```

#### 函数CaseSumTwo

```
"".CaseSumTwo STEXT size=83 args=0x4 locals=0xc
        0x0000 00000 (point.go:30)      TEXT    "".CaseSumTwo(SB), $12-4
        ...
        // point.go:31 -> data := GenerateData()
        // point.go:34 -> *result += data[i] 
        0x001a 00026 (point.go:31)      MOVL    (SP), AX
        0x0027 00039 (point.go:33)      MOVL    "".result+16(SP), BX
        0x002f 00047 (point.go:34)      MOVL    (AX)(BP*4), SI // 栈寄存器移动四个字节， -> SI源变址寄存器
        0x0032 00050 (point.go:34)      ADDL    SI, (BX)  // SI
        0x0034 00052 (point.go:33)      INCL    BP
        0x0035 00053 (point.go:33)      CMPL    BP, DX
        0x0037 00055 (point.go:33)      JGE     65
        0x0039 00057 (point.go:34)      TESTB   AX, (BX)
        0x003b 00059 (point.go:34)      CMPL    BP, CX
        0x003d 00061 (point.go:34)      JCS     47
        0x003f 00063 (point.go:34)      JMP     69
        0x0041 00065 (<unknown line number>)    PCDATA  $2, $-2
        0x0041 00065 (<unknown line number>)    PCDATA  $0, $-2
        0x0041 00065 (<unknown line number>)    ADDL    $12, SP
        0x0044 00068 (<unknown line number>)    RET
        0x0045 00069 (point.go:34)      PCDATA  $2, $0
        0x0045 00069 (point.go:34)      PCDATA  $0, $1
        0x0045 00069 (point.go:34)      CALL    runtime.panicindex(SB)
        0x004a 00074 (point.go:34)      UNDEF
        0x004c 00076 (point.go:34)      NOP
```

#### 函数CaseSumThree
```
"".CaseSumThree STEXT size=97 args=0x4 locals=0x10
        0x0000 00000 (point.go:39)      TEXT    "".CaseSumThree(SB), $16-4
        ...
        // point.go:40 -> data := GenerateData()
        // point.go:42 -> tmp := *result
        // point.go:44 -> tmp += data[i]
        // point.go:46 -> *result = tmp
        0x001a 00026 (point.go:40)      MOVL    (SP), AX
        0x0021 00033 (point.go:42)      PCDATA  $2, $2
        0x0021 00033 (point.go:42)      PCDATA  $0, $1
        0x0021 00033 (point.go:42)      MOVL    "".result+20(SP), DX
        0x0025 00037 (point.go:42)      MOVL    (DX), BX // ->BX数据指针寄存器
        0x0027 00039 (point.go:41)      MOVL    "".testData+4(SB), BP
        0x002d 00045 (point.go:41)      XORL    SI, SI
        0x002f 00047 (point.go:43)      JMP     67
        0x0031 00049 (point.go:43)      LEAL    1(SI), DI
        0x0034 00052 (point.go:43)      MOVL    DI, "".i+12(SP) // 移动DI到栈指针12字节的位置
        0x0038 00056 (point.go:44)      MOVL    (AX)(SI*4), DI // 源变址寄存器移动四个字节（32位），-> 目的变址寄存器
        0x003b 00059 (point.go:44)      ADDL    DI, BX // DI+BX
        0x003d 00061 (point.go:43)      MOVL    "".i+12(SP), DI 
        0x0041 00065 (point.go:43)      MOVL    DI, SI
        0x0043 00067 (point.go:43)      CMPL    SI, BP
        0x0045 00069 (point.go:43)      JGE     77
        0x0047 00071 (point.go:44)      CMPL    SI, CX
        0x0049 00073 (point.go:44)      JCS     49
        0x004b 00075 (point.go:44)      JMP     83
        0x004d 00077 (point.go:46)      PCDATA  $2, $0
        0x004d 00077 (point.go:46)      MOVL    BX, (DX)
        0x004f 00079 (point.go:47)      ADDL    $16, SP
        0x0052 00082 (point.go:47)      RET
        0x0053 00083 (point.go:44)      CALL    runtime.panicindex(SB)
        ...
```

### 比较结论
CaseSumTwo函数，在进行ADDL之前，因为“*result”为指针变量，所以不能直接与data[i]运算。因此需要创建一个栈空间，并指向data的地址并，然后通过移动栈指针后得到下一个值的地址，并赋与SI。
CaseSumThree函数，在进行ADDL执行前，创建了一个值变量，那么在执行ADDL的时候，只需要移动SI获取下一个data的值就可以直接进行算数运算，中间少了地址的引用的栈的操作。
#### 堆和栈
其实说白了，就是CaseSumTwo中 `*result`内存是分配在堆上的，而 CaseSumThree中 `tmp`是分配在栈上的，而堆和栈堆性能区别这里做一个简单堆比较：
1. 有寄存器直接对栈进行访问（esp，ebp），而对堆访问，只能是间接寻址。 
也就是说，可以直接从地址取数据放至目标地址；使用堆时，第一步将分配的地址放到寄存器，然后取出这个地址的值，然后放到目标地址。 
2. 栈中数据cpu命中率更高，满足局部性原理。 
3. 栈是编译时系统自动分配空间，而堆是动态分配（运行时分配空间），所以栈的速度快。 
4. 栈是先进后出的队列结构，比堆结构相对简单，分配速度大于堆。


# 总结
本章主要讲了三个点：
 1. 消除循环的低效率
 2. 减少过程调用
 3. 消除不必要的内存引用

引用《深入计算机系统原理》一书中对性能优化所提到的三个方面：
 1. 高级设计，为遇到的问题选择适当的算法和数据结构。要特别警觉，避免使用那些会渐进地产生糟糕性能的算法或编码技术。
 2. 基本编码原则，从指令的角度考虑，开发中应如何编码，才能减少执行的指令。
 3. 低级优化，针对现代处理器，如何让cpu的流水线尽量饱合。
所以，一个优秀的程序员在写每一行代码，定义每一个变量，也许背后思考的就会更多。