---
typora-copy-images-to: ./images
typora-root-url: ../nginx
---
# Nginx Mirror 流量复制的心路历程

## 背景

上篇文章写到 [Lua OpenResty的容器化](https://segmentfault.com/a/1190000039867278)，项目完成了上云，但是有面临了几个新的问题：

1. 旧业务跑了这么多年，如何保证上云后的稳定性呢？
2. 如何确保接口都能正常访问呢？
4. 如何确保两边api请求的幂等呢？

为了保证服务的质量，我们决定借助 `nginx mirror（流量复制）`来统计服务的幂等情况，作为判断服务是否达到上线标准的一个重要指标。



## nginx mirror是什么？

**[ngx_http_mirror_module](http://nginx.org/en/docs/http/ngx_http_mirror_module.html)** 模块（1.13.4）通过创建后台镜像子请求来实现原始请求的镜像。镜像子请求的响应将被忽略。

利用 mirror 模块，业务可以将线上实时访问流量拷贝至其他环境，基于这些流量可以做版本发布前的预先验证，进行流量放大后的压测。

<img src="./images/image-20210421161757605.png" alt="image-20210421161757605" style="zoom:50%;" />

官方案例：

> ```nginx
> upstream mirror_backend {
>     server mirrorHost:mirrorPort;
> }
> 
> location / {
>     mirror /mirror;
>     proxy_pass http://backend;
> }
> 
> location = /mirror {
>     internal;
>     proxy_pass http://mirror_backend$request_uri;
> }
> ```



## 如何判断mirror后的结果是否幂等？

mirror后，可以通过分析两边的access.log，来判断幂等情况：

```shell
170.21.40.147 [20/Apr/2021:14:02:02 +0000] "POST /test/query?client_name=test&client_version=2148205812&client_sequence=1&r=1618327396996&isvip=0&release_version=1.1.0
```

但是要判断同一个请求在两边的结果是否幂等，单目前的信息还是不够的，比如：

1. 如何判断两边是同一个请求？
2. 如何判断请求返回的结果是一致的？
3. 如何对所有请求的幂等性进行统计？



### 如何判断两边是同一个请求？

要判断两边请求是否同一个，我们需要一个Unique Tracing ID，并且贯穿两边的access.log。

> Nginx在 `1.11.0` 版本中就提供了内置变量 `$request_id` ，其原理就是生成32位的随机字符串，虽不能比拟UUID的概率，但32位的随机字符串的重复概率也是微不足道了，所以一般可视为UUID来使用即可。

so，我们可以在生产机器的nginx的access.log增加$request_id：

```nginx
log_format  access_log_format
           ''$remote_addr $request_time $request $request_id';
```



并将`$request_id`通过header和请求一同转发到mirror的服务器上：

```nginx
location = /mirror {
    internal;
  	proxy_set_header X-Request-Id $request_id; # $request_id
    proxy_pass http://test_backend$request_uri;
}
```



最后在mirror服务器的nginx access日志格式增加：

`$http_request_id`，就可以得到：

```shell
170.21.40.147 [20/Apr/2021:14:02:02 +0000] "POST /test/query?client_name=test&client_version=2148205812&client_sequence=1&r=1618327396996&isvip=0&release_version=1.1.0 nqrbyg9l50jwu3ezmvtfc6hx487i2kpo
```

两个通过`nqrbyg9l50jwu3ezmvtfc6hx487i2kpo`就可以将请求关联上了。



### 如何判断请求返回的结果是一致的？

`$request_id`已经有了，那么如何判断结果是否一致呢？这里可以将两边服务器的返回结果（`resp_body`）进行md5，然后进行比较。

在两边的nginx定义变量：

```nginx
set $resp_body "";
body_filter_by_lua '
    local resp_body = string.sub(ngx.arg[1], 1, 1000)
    ngx.ctx.buffered = (ngx.ctx.buffered or "") .. resp_body
    if ngx.arg[2] then
       ngx.var.resp_body = ngx.md5(ngx.ctx.buffered)
    end
';
```
然后在`log_format  access_log_format`增加`$resp_body`，即：

```nginx
log_format  access_log_format
           ''$remote_addr $request_time $request $request_id $resp_body';
```

得出日志：

```shell
170.21.40.147 [20/Apr/2021:14:02:02 +0000] "POST /test/query?client_name=test&client_version=2148205812&client_sequence=1&r=1618327396996&isvip=0&release_version=1.1.0 nqrbyg9l50jwu3ezmvtfc6hx487i2kpo 0vy27c35aqrsxgzflk4inwtjd1omu96p
```

ps：`nqrbyg9l50jwu3ezmvtfc6hx487i2kpo`为`request_id`，`0vy27c35aqrsxgzflk4inwtjd1omu96p`为`resp_body`md5后的结果。



### 如何对所有请求的幂等性进行统计？

至此，我们已经得到了我们想要的日志格式了，通过`request_id`和`resp_body`可以对两个请求的信息幂等判断。

但是又有一个新的问题，一台机器采集到的access_log就接近`3000W`，由于`$request_id`是字符随机串，所以需要对两边access log进行排序，才能做统计，但是要怎么做呢？

1. 由于日志太大，所以根据`request_id`第一个字符进行mod 10，将文件切割成10份。
2. 对10份log根据`request_id`进行排序。
3. 合并文件
4. 最后通过比较的方式，对两个文件进行统计。























