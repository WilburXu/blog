---
//typora-root-url: ./
typora-copy-images-to: ./images/
---

# lua远程调试 Remote Debug

## 日常的debug

当把一个本地项目部署到**远程测试服务器**后有可能出现意想不到错误，为了排查问题可能会变成：

这样：

![](./images/remote-01.png)



然后这样：

![](./images/remote-02.jpg)



最后就：

![](./images/image-20210420102007629.png)

最可怕的是，由于堆栈的关系，很难在一次debug日志中拿到想要的信息，往往是一层层往下打日志，才能拿到想要的debug信息。



##  remote debug

本地服务器开放端口，将远程服务器的断点信息打到本地服务器。



<img src="./images/image-20210420145522999.png" alt="image-20210420145522999" style="zoom:50%;" />



### 那具体如何实现呢？

jetbrains的“EmmyLua”插件  + mobdebug库

#### 本地jetbrains增加EmmyLua插件安装

![image-20210420150127368](./images/image-20210420150127368.png)

#### 远端服务器增加mobdebug包放到项目debug目录下，并增加配置信息

https://github.com/pkulchenko/MobDebug/blob/master/src/mobdebug.lua

```lua
local mobdebug = require("debug.mobdebug");
mobdebug.rbasedir("/usr/local/openresty/nginx/lua/")  -- remote
mobdebug.lbasedir("/Users/wilburxu/lua/test/")  -- local
mobdebug.start("host.docker.internal", 28172);
```

![image-20210420150815357](./images/image-20210420150815357.png)

![image-20210420150927332](./images/image-20210420150927332.png)

ps：断点信息发回的是远端服务器的line，所以本地服务器要保证和远端服务器的line一致。





#### 本地添加调试configuration

![image-20210420150345594](./images/image-20210420150345594.png)





### 发送请求

![image-20210420145915108](.//images//image-20210420145915108.png)

就可以得到我们想要的堆栈信息了。

