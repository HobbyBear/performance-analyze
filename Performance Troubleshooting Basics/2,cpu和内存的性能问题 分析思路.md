# cpu和内存的性能问题 分析思路

在之前一篇[网络性能问题排查思路](https://github.com/HobbyBear/performance-analyze/blob/de48368f6fbbc1a9de7f98ab710d3482447f13ce/Performance%20Troubleshooting%20Basics/%E7%BD%91%E7%BB%9C%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5%E6%89%8B%E6%AE%B5.md)里，我将性能分析步骤分为3个步骤，这里我再将这3个步骤列举出来：

1，系统层面发现问题

2，定位到具体异常进程

3，定位到进程中引发异常的代码段

现在来看看如何在每个步骤里分析cpu的使用情况。

你将会像侦探一样，一层层抽丝剥茧，排查过程十分有趣。

之所以把cpu和内存放到一起来讲排查思路，是因为它们的排查思路基本一致。
## 系统以及进程角度看cpu，内存使用情况

我们一般可以用top命令就能得到这系统和进程的cpu以及内存信息，可能你已经很熟悉top命令了，不过我还是将top命令与cpu，内存相关的输出简单阐述下。

top 命令的前面部分如下，反映了整个系统的使用情况。
```go
interface: eth0
IP address is: 192.168.0.2
MAC address is: fa:16:3e:7a:bd:31
top - 18:25:22 up 145 days,  2:46, 42 users,  load average: 0.36, 0.56, 0.57
Tasks: 336 total,   1 running, 335 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.4 us,  0.6 sy,  0.0 ni, 96.9 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 24521316 total,   547992 free, 16009260 used,  7964064 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6852168 avail Mem 
```
cpu 各个指标含义如下：

us 用户态程序占用cpu  ，sy 内核占用cpu ，id 空闲cpu，wa 等待cpu执行耗时，hi 硬中断耗时，si 软中断耗时

这里对硬中断和软中断再说说我的理解。硬件中断比较好理解，外部硬件向cpu引脚发送信号就会触发硬件中断，cpu会根据信号选择预先设置好的中断函数去执行。软件通过int指令其实也是向cpu引脚发送信息，有些文章说软件触发的中断就是软中断，我认为是不恰当的，因为软件同样也可以使用int指令。

再来谈谈软中断，软中断的实现逻辑大致来概括下，内核针对每个cpu核心都创建了一个进程，进程会不断检查系统内是不是有软中断的信号，如果有的话，那么就寻找软中断信号对应的处理函数去执行。

内存各个指标的含义如下:

total 是总的内存字节数，free 是空闲的内存字节数，used 是使用的内存字节数，buff/cache 是内存中用于page cache和块缓存buffer 的字节数

> 说说我对page cache和buffer的理解，page cache针对于文件系统而言，内核读文件是一页一页的读取，读取出来的结果会暂存在内存中以便下次直接读取内存，buffer针对于块设备而言，内核中使用bio这个结构代表一个块，而buffer就是多个块的缓存结果，以便下次读取块设备直接读取从内存中获取到。

在内存下一行是交换空间的大小，交换空间其实是磁盘上的一片区域，当内存放不下时，本来会触发oom，但是为了在容忍瞬时内存使用超过内存上限时，不对进程oom，我们可以开启交换空间，当内存放不下时，内核会将内核中一部分数据置换到磁盘上，等用到的时候再换回来。


top命令的下半部分是进行列表，我们可以在top输出界面按大写P将会按照cpu从大到小排序，或者按大写M进行内存从高到低的排序。
```go
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                    
30283 webserv+  20   0  927496 183500   4128 S  22.9  0.7  18708:29 zdisk-sync                                 
 5788 mysql     20   0 8027552   3.7g   1960 S  12.0 15.8   2491:06 mysqld                                     
 5990 nemo      20   0 1466340  79072   2848 S   6.3  0.3  12289:55 nsproxy 
``` 

## 从进程角度看cpu使用情况
cpu的性能排查可以说相对来说比较容易，一个top便可以将系统和进程的cpu情况展示出来，假设此时你已经发现某个进程的cpu比较高，那么如何找到具体是哪段代码消耗cpu或者内存比较多呢？

由于我比较熟悉golang，所以我还是用go程序来举例，golang中内置的pprof工具可以通过采样的方式分析程序的cpu或者内存占用。生成cpu的性能分析文件的方式可以采用http生成网页的方式也可以用程序代码，具体的通过pprof查看cpu的使用和统计原理 可以看[golang pprof 监控系列(5) —— cpu 使用 统计原理
]( https://github.com/HobbyBear/performance-analyze/blob/de48368f6fbbc1a9de7f98ab710d3482447f13ce/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(5)%20%E2%80%94%E2%80%94%20cpu%20%E4%BD%BF%E7%94%A8%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md ) 以及 [golang pprof监控系列（2） —— memory，block，mutex 使用](https://github.com/HobbyBear/performance-analyze/blob/de48368f6fbbc1a9de7f98ab710d3482447f13ce/golang%20pprof%20tools/golang%20pprof%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(2%EF%BC%89%20%E2%80%94%E2%80%94%20%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E4%BD%BF%E7%94%A8.md )

至此，我们介绍完了从系统到进程再到具体代码看cpu以及内存使用率的方式。



