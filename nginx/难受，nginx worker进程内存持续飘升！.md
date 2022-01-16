---
typora-copy-images-to: ./images
typora-root-url: ../nginx
---

# 难受，nginx worker进程内存持续飘升！

## 背景

前两篇文章讲了云主机上`lua openresty`项目容器化的历程，在测试环境经过一段时间的验证，一切都比较顺利，就在线上开始灰度。

但是，好景不长。灰度没多久，使用`top pod`查看时，发现内存满了，最开始怀疑k8s的`resources limit memory(2024Mi)`分配小了，放大后(4096Mi)，重启pod，没多久又满了。

紧接着，怀疑是放量较大，负载高了引起的，扩大`hpa`，再重启，好家伙，不到两炷香时间，又满了。 

到此，就把目光投向了`pod`内部的程序了，估摸着又是哪里出现内存泄漏了。



## 坑在哪里

1. 每过一段时间，内存忽上忽下，并且nginx worker（子进程）id号不断增加，是谁杀的进程？
2. 云主机没有出现这样的问题，难道是k8s引起的？
3. `pod`的`resources limit memory`增加和`hpa`增加都没有解决问题。
4. `nginx -s reload`可以释放内存，但是没多久后又满了。



## 解决办法

### 谁杀的进程？

命令：`ps aux`

```shell
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 252672  5844 ?        Ss   Jun11   0:00 nginx: master process /data/openresty/bin/openresty -g daemon off;
nobody     865 10.1  0.3 864328 590744 ?       S    14:56   7:14 nginx: worker process
nobody     866 13.0  0.3 860164 586748 ?       S    15:13   7:02 nginx: worker process
nobody     931 15.6  0.2 759944 486408 ?       R    15:31   5:37 nginx: worker process
nobody     938 13.3  0.1 507784 234384 ?       R    15:49   2:23 nginx: worker process
```

发现woker进程号已经接近1000了，那么肯定不断的被kill，然后再调起，那究竟是谁干的呢？ 通过 `dmesg`命令：

```shell
[36812300.604948] dljgo invoked oom-killer: gfp_mask=0xd0, order=0, oom_score_adj=999
[36812300.648057] Task in /kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pode4ad18fa_3b8f_4600_a557_a2bc853e80d9.slice/docker-c888fefbafc14b39e42db5ad204b2e5fa7cbfdf20cbd621ecf15fdebcb692a61.scope killed as a result of limit of /kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pode4ad18fa_3b8f_4600_a557_a2bc853e80d9.slice/docker-c888fefbafc14b39e42db5ad204b2e5fa7cbfdf20cbd621ecf15fdebcb692a61.scope
[36812300.655582] memory: usage 500000kB, limit 500000kB, failcnt 132
[36812300.657244] memory+swap: usage 500000kB, limit 500000kB, failcnt 0
[36812300.658931] kmem: usage 0kB, limit 9007199254740988kB, failcnt 0
……
[36675871.040485] Memory cgroup out of memory: Kill process 16492 (openresty) score 1286 or sacrifice child
```

发现当cgroup内存不足时，Linux内核会触发[cgroup OOM](https://v1-17.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/assign-memory-resource/)来选择一些进程kill掉，以便能回收一些内存，尽量继续保持系统继续运行。

虽然kill掉了nginx worker进程后释放了内存，短暂性的解决了问题，但根本性问题还没解决。



### 为啥云主机没问题？

将云主机的lua代码拷贝到本地进行对比，发现代码本身并没有什么问题。

那只能是其他方面的问题引起了。



### 如何是好？

以上两个点，都没能很好的定位问题问题，看来只能通过通过 `top`、`pmap`、`gdb`等命令去排查问题了。

#### 1. top内存泄露的进程

通过`top` 查看哪个进程占用内存高

```shell
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                                               
  942 nobody    20   0  618.9m 351.7m   4.2m S  18.0  0.2   4:05.72 openresty                                                                                                                                                             
  943 nobody    20   0  413.8m 146.7m   4.2m S  11.7  0.1   1:18.93 openresty                                                                                                                                                             
  940 nobody    20   0  792.0m 524.9m   4.2m S   7.0  0.3   6:25.81 openresty                                                                                                                                                             
  938 nobody    20   0  847.4m 580.2m   4.2m S   3.7  0.3   7:15.97 openresty 
    1 root      20   0  246.8m   5.7m   3.9m S   0.0  0.0   0:00.24 openresty                                                                                                                                               
```



#### 2. pmap查看进程的内存分配

通过 `pmap -x pid` 找出该进程的内存分配，发现`0000000000af7000`存在大内存的分配。

```shell
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000    1572     912       0 r-x-- nginx
0000000000788000       4       4       4 r---- nginx
0000000000789000     148     128     116 rw--- nginx
00000000007ae000     140      28      28 rw---   [ anon ]
0000000000a15000     904     900     900 rw---   [ anon ]
0000000000af7000  531080  530980  530980 rw---   [ anon ]
0000000040048000     128     124     124 rw---   [ anon ]
……
```



#### 3. /proc/pid/smaps 定位内存泄露的地址范围

获得大内存的内存地址后，通过 `cat /proc/pid/smaps`命令，查看内存段的具体起始位置：

```shell
00af7000-21412000 rw-p 00000000 00:00 0                                  [heap]
Size:             533612 kB
Rss:              533596 kB
Pss:              533596 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:    533596 kB
Referenced:       533596 kB
Anonymous:        533596 kB
AnonHugePages:         0 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
VmFlags: rd wr mr mw me ac sd
```



#### 4. gcore 转存进程映像及内存上下文

```shell
gcore pid
```

得到 "core.pid"文件。



#### 5.`gdb` 加载内存信息

```shell
gdb core.pid

sh-4.2$ gdb core.942
GNU gdb (GDB) Red Hat Enterprise Linux 7.*
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
[New LWP pid]
Core was generated by `nginx: worker process'.
#0  0x00007ff9435e1463 in ?? ()
"/tmp/core.942" is a core file.
Please specify an executable to debug.
(gdb) 
```

进入以上命令窗口



#### 6. `dump binary` 导出泄露内存的内容

第3点的时候已经得到了内存起始地址了，这时候我们内存地址导出：

```shell
dump binary memory worker-pid.bin 0x00af7000 0x21412000
```



查看导出的文件大小

```shell
sh-4.2$ du -sh worker-pid.bin
511M    worker-pid.bin
```



#### 7. 二进制文件分析

用`hex`工具打开二进制文件，发现里面大量的json对象，通过分析json对象，发现pod上的so加密库为旧的（git上并没有跟新），而云主机依赖的是新的。

替换so库，重启pod，问题解决！～。



## 总结

生产环境上的内存泄漏是一个比较头疼的问题，不妨通过以上的方式去分析，也是一个不错的思路。







