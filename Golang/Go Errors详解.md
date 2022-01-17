# Go Errors 详解

Golang 中的错误处理和 PHP、JAVA 有很大不同，没有 `try...catch` 语句来处理错误。因此，Golang 中的错误处理是一个比较有争议的点，如何更好的 **理解** 和 **处理** 错误信息是值得去深入研究的。

## Go 内置 errors

Go `error` 是一个接口类型，它包含一个 `Error()` 方法，返回值为 `string`。任何实现这个接口的类型都可以作为一个错误使用，Error 这个方法提供了对错误的描述：

```go
// http://golang.org/pkg/builtin/#error
// error 接口的定义
type error interface {
    Error() string
}

// http://golang.org/pkg/errors/error.go
// errors 构建 error 对象
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

`error` 是一个接口类型，它包含一个 `Error()` 方法，返回值为 `string`。只要实现这个 `interface` 都可以作为一个错误使用，`Error` 这个方法提供了对错误的描述。



## error 创建

`error` 的创建方式有两种方法：

### 1. errors.New()

```go
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}
```

Q：为何 `errors.New()` 要返回指针？

A：避免 New 的内容相当，造成的歧义，看看下面的例子就可以理解为什么了：

```go
func main() {
    errOne := errors.New("test")
    errTwo := errors.New("test")

    if errOne == errTwo {
      log.Printf("Equal \n")
    } else {
      log.Printf("notEqual \n")
    }
}
```

输出：

```sh
notEqual
```

如果使用 `errorString` 的值去比较，当项目逐渐盘大、复杂，对于 `New()` 内容也就难以保证唯一，到那时对于问题的排查，也将是灾难性的。

有些时候我们需要更加具体的信息。即需要具体的 “上下文” 信息，表明具体的错误值。

这就用到了 `fmt.Errorf` 函数



### 2. fmt.Errorf()

```go
fmtErr := fmt.Errorf("fmt.Errorf() err, http status is %d", 404)
fmt.Printf("fmtErr errType：%T，err: %v\n", fmtErr, fmtErr)
```

输出：

```go
fmtErr errType is *errors.errorString，err is fmt.Errorf() err, http status is 404
```

为什么 `fmtErr` 返回的错误类型也是 ：`*errors.errorString`，我们不是用 `fmt.Errorf()` 创建的吗？

一起来看下源码：

```go
// Errorf formats according to a format specifier and returns the string as a
// value that satisfies error.
//
// If the format specifier includes a %w verb with an error operand,
// the returned error will implement an Unwrap method returning the operand. It is
// invalid to include more than one %w verb or to supply it with an operand
// that does not implement the error interface. The %w verb is otherwise
// a synonym for %v.
func Errorf(format string, a ...interface{}) error {
    p := newPrinter()
    p.wrapErrs = true
    p.doPrintf(format, a)
    s := string(p.buf)
    var err error
    if p.wrappedErr == nil {
      err = errors.New(s)
    } else {
      err = &wrapError{s, p.wrappedErr}
    }
    p.free()
    return err
}
```

通过源码，可以发现，`p.wrappedErr` 为 `nil` 的时候，会调用 `errors.New()` 来创建错误。

那问题来了，这个 `p.wrappedErr` 是什么？

我们来看个例子：

```go
wErrOne := errors.New("this is one ")
wErrTwo := fmt.Errorf("this is two %w", wErrOne)
fmt.Printf("wErrOne type is %T err is %v \n", wErrOne, wErrOne)
fmt.Printf("wErrTwo type is %T err is %v \n", wErrTwo, wErrTwo)
```

输出：

```go
wErrOne type is *errors.errorString err is this is one  
wErrTwo type is *fmt.wrapError err is this is two this is one  
```

发现没有？使用 %w 返回的 `error` 对象，输出的类型是 `*fmt.wrapError`

`%w` 是 go 1.13 新增加的错误处理特性 。



## Go 错误处理实践

如何获得更详细错误信息，比如`stack trace`，帮助定位错误原因?

**有人说**，层层打 log，但这会造成日志打得到处都是，难以维护。

**又有人说**，使用 `recover` 捕获 `panic`，但是这样会导致 `panic` 的滥用。

`panic` 只用于真正异常的情况，如

- 在程序启动的时候，如果有强依赖的服务出现故障时 `panic` 退出
- 在程序启动的时候，如果发现有配置明显不符合要求， 可以 `panic` 退出（防御编程）
- 在程序入口处，例如 gin 中间件需要使用 `recovery` 预防 `panic` 程序退出



### pkg/errors 库

这里，我们通过一个很小的包 [github.com/pkg/errors](https://github.com/pkg/errors) 来试图解决上面的问题。

看一个案例：

```go
package main

import (
    "github.com/pkg/errors"
    "log"
    "os"
)

func main() {
    err := mid()
    if err != nil {
      	// 返回 err 的根本原因
    		log.Printf("cause is %+v \n", errors.Cause(err))
      
      	// 返回 err 调用的堆栈信息
    		log.Printf("strace tt %+v \n", err)
    }
}

func mid() (err error) {
		return test()
}


func test() (err error) {
		_, err = os.Open("test/test.txt")
    if err != nil {
      	return errors.Wrap(err, "open error")
    }

    return nil
}

```

输出：

```go
2022/01/17 00:26:17 cause is open test.test: no such file or directory 
2022/01/17 00:26:17 strace tt open test.test: no such file or directory
open error
main.test
        /path/err/wrap_t/main.go:41
main.mid
        /path/err/wrap_t/main.go:35
main.main
        /path/err/wrap_t/main.go:13
runtime.main
        /usr/local/Cellar/go/1.17.2/libexec/src/runtime/proc.go:255
runtime.goexit
        /usr/local/Cellar/go/1.17.2/libexec/src/runtime/asm_amd64.s:1581 
```



#### pkg/errors

上层调用者使用errors.Cause(err)方法就能拿到这次错误造成的罪魁祸首。

```go
// Wrap returns an error annotating err with a stack trace
// at the point Wrap is called, and the supplied message.
// If err is nil, Wrap returns nil.
func Wrap(err error, message string) error {
   if err == nil {
      return nil
   }
   err = &withMessage{
      cause: err,
      msg:   message,
   }
   return &withStack{
      err,
      callers(),
   }
}
```


```go
// Is 指出当前的错误链是否存在目标错误。
func Is(err, target error) bool

// As 检查当前错误链上是否存在目标类型。若存在则ok为true，e为类型转换后的结果。若不存在则ok为false，e为空值
func As(type E)(err error) (e E, ok bool)
```



## 参考

https://go.googlesource.com/proposal/+/master/design/29934-error-values.md

https://github.com/golang/go/labels/LanguageChange

https://github.com/pkg/errors

https://go.googlesource.com/proposal/+/master/design/go2draft-error-values-overview.md

https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling.md
