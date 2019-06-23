# Golang Gin加载流程剖析

框架一直是敏捷开发中的利器，能让开发者很快的上手并做出应用。但是发现有不少开发者在使用框架的时候，对框架的`加载流程`并不是特别了解，那么对框架的`使用`、`扩展` 甚至`改造`都可能会遇到不少的阻碍，那么今天就带大家一起来看下Gin框架的`加载流程`

## Gin安装

首次安装，使用 `go get`命令获取即可。

```
go get github.com/gin-gonic/gin
```

### 快速运行

在 main 包中，引入 gin 包并初始化。

```go
package main

import (
	"net/http"
	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
  
	router.GET("/", func(c *gin.Context) {
		c.String(http.StatusOK, "Hello World")
	})
  
	router.Run(":8013")
}
```

#### 访问测试

[http://127.0.0.1:8013/](http://127.0.0.1:8013/)

#### 输出结果

```go
Hello World
```

## 加载流程

从开始的`import`到`Hello Wrold`中间究竟经历了什么？