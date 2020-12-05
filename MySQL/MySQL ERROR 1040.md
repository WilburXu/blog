# MySQL ERROR 1040: Too many connections

如题，本章主要讲下当服务器出现` ERROR 1040: Too many connections`错误时的一些处理心得。

## max_connections查看

```mysql
## 查看最大连接数
SHOW VARIABLES LIKE "max_connections";
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 512   |
+-----------------+-------+

## 查看已使用最大连接数
SHOW VARIABLES LIKE 'Max_used_connections';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Max_used_connections | 499   |
+----------------------+-------+
```



## 处理方案

这个问题一般有两种处理方案，解决方案非常容易，我们只需要增加`max_connections`连接数即可。

### 增加当前会话的mysql最大连接数

```mysql
SET GLOBAL max_connections = 1000;
```

上面mysql连接值临时增加到1000，但仅适用于当前会话。一旦我们重新启动mysql服务或重新启动系统，该值将重置为默认值。

### 永久增加mysql最大连接数

为了永久增加mysql连接数，我们需要编辑mysql配置文件，即`/etc/my.cnf`。

`sudo vim /etc/my.cnf`

```shell
## 修改
max_connections = 1000
```



保存文件重启MySQL即可生效。



## 扩多少合适？

`Max_connextions`并不是越大越好的，那么如何配置？

### 方式一

对于提高MySQL的并发，很大程度取决于内存，官方提供了一个关于`innodb`的内存[计算方式](https://dev.mysql.com/doc/refman/8.0/en/innodb-init-startup-configuration.html)：

```mysql
innodb_buffer_pool_size
+ key_buffer_size
+ max_connections * (sort_buffer_size + read_buffer_size + binlog_cache_size)
+ max_connections * 2MB
```



### 方式二

安装比例扩容：

```mysql
max_used_connections / max_connections * 100% = [85, 90]%
```

`最大使用连接数/最大连接数`达到了80%～90%区间，就建议进行优化或者扩容了。



## 扩展

以下也涉及几种常见的影响MySQL性能的情况：

### 线程

```mysql
SHOW STATUS LIKE  'Threads%';
+-------------------+--------+
| Variable_name     | Value  |
+-------------------+--------+
| Threads_cached    | 1      | 
| Threads_connected | 217    |
| Threads_created   | 29     |
| Threads_running   | 88     |
+-------------------+--------+

SHOW VARIABLES LIKE 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 10     |
+-------------------+-------+
```

- Threads_cached 线程在缓存中的数量
- Threads_connected 当前打开的连接数
- Threads_created 创建用于处理连接的线程数。
- Threads_running  未休眠的线程数

如果Threads_created大，则可能要增加thread_cache_size值。缓存未命中率可以计算为Threads_created / Connections



### 查看表锁情况

```mysql
SHOW GLOBAL STATUS LIKE 'table_locks%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Table_locks_immediate | 90    |
| Table_locks_waited    | 0     |
+-----------------------+-------+
```

- Table_locks_immediate 立即获得表锁请求的次数
- Table_locks_waited 无法立即获得对表锁的请求的次数，需要等待。这个值过高说明性能可能出现了问题，并影响连接的释放



### 慢查询

```mysql
show variables like '%slow%';

+---------------------------+----------------------------------------------+
| Variable_name             | Value                                        |
+---------------------------+----------------------------------------------+
| slow_launch_time          | 2                                            |
| slow_query_log            | On                                           |
+---------------------------+----------------------------------------------+
```



 ###  线程详情

```mysql
## 查看每个线程的详细信息
SHOW PROCESSLIST;
+--------+----------+------------------+--------------+---------+-------+-------------+------------------+
| Id     | User     | Host             | db           | Command | Time  | State       | Info             |
+--------+----------+------------------+--------------+---------+-------+-------------+------------------+
|      3 | xxxadmin | localhost        | NULL         | Sleep   |     1 | cleaning up | NULL             |
|      4 | xxxadmin | localhost        | NULL         | Sleep   |     0 | cleaning up | NULL             |
|      5 | xxxadmin | localhost        | NULL         | Sleep   |     6 | cleaning up | NULL             |
+--------+----------+------------------+--------------+---------+-------+-------------+------------------+
```



## 总结

当然，以上只是一个大概的解决思路，无论使用哪一种方式，都需要结合实际业务场景去扩容。

另外，对于生产环境，设置恰当的告警阈值，也是很有必要的。

最后，在编程时，由于用MySQL语句调用数据库执行`SQL`，会分配一个线程操作MySQL，所以在结束调用后，需要回收连接，避免泄漏。



## 参考文章

https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_connections

https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html

https://dev.mysql.com/doc/refman/8.0/en/memory-use.html

[MySQL Calculator](http://www.mysqlcalculator.com/)

