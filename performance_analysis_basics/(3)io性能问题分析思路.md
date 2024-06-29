# io 性能问题分析思路

之前我们分析了 [网络](./(1)网络问题排查手段.md) 和 [cpu和内存](./(2)cpu和内存的性能问题分析思路.md)的性能问题排查思路，今天我们来看看针对磁盘io的性能问题 应该如何去排查。

排查的总体思路不变，我这里再列举下：

1，系统层面发现问题


2，定位到具体异常进程

3，定位到进程中引发异常的代码段


好的，接下来，开始着手分析，先从系统层面看io情况。

## 系统层面看io 情况

我们可以用iostat 去观察在系统层面的上io情况。命令如下:

```go
iostat  -x 
Linux 3.10.0-957.21.3.el7.x86_64 (hw-sg1-test-0001)     04/11/2023      _x86_64_        (12 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.12    0.00    0.66    0.14    0.00   97.07

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.01    49.08    1.86   55.53    37.62   757.19    27.70     0.07    1.66   26.38    0.83   0.29   1.65

```

-x 将以扩展模式执行iostat命令，使用它我们能展示更多的信息。现在我们来分析下输出的信息。



| rrqm/s      | 每秒合并读操作的次数                                                                    |
|-------------|:---------------------------------------------------------------------------------------:|
| wrqm/s      | 每秒合并写操作的次数                                                                    |
| r/s         | 每秒读操作的次数                                                                        |
| w/s         | 每秒写操作的次数                                                                        |
| rKB/s       | 每秒读取的千字节数                                                                      |
| wKB/s       | 每秒写入的千字节数                                                                      |
| avgrq-sz    | 每个I/O的平均扇区数                                                                     |
| avgqu-sz    | 平均未完成的I/O请求数量，也是队列里的平均I/O请求数量                                     |
| await       | 每个I/O平均所需的时间（不仅包括硬盘设备处理I/O的时间，还包括了在kernel队列中等待的时间。）  |
| r_await     | 每个读操作平均所需的时间（不仅包括硬盘设备读操作的时间，还包括了在kernel队列中等待的时间） |
| w_await     | 每个写操作平均所需的时间（不仅包括硬盘设备写操作的时间，还包括了在kernel队列中等待的时间） |
| svctm       | 已被废弃的指标，没什么意义                                                               |
| %util       | 该硬盘设备的繁忙比率（io设备处于工作的自然时间除以采样的自然时间）                        |

这里可能特别要注意的是%util的含义，%util只能代表设备在这段期间有没有被使用，举个例子，如果在1s内，硬盘设备都在工作，那么%util将会是100%，但是100%并不能说明硬盘饱和了，因为现在硬盘基本都有多io队列并行处理的能力，硬盘在1s内都在工作，可能仅仅是在处理一个io队列，如果此时有一个新io队列等待被执行，那么硬盘也是能够处理的，也就是说，硬盘处理两个io队列和一个io队列可能用时都是1s。%util都将是100%。


## 进程维度看io

通过iostat 能够看到系统层面的io情况，那么当发现系统层面io读写异常时，如何定位到具体进程呢？这里我推荐用iotop命令。iotop命令可以实时的看到进程读写磁盘的速率。

```go
sudo iotop 

// 输出如下
Total DISK READ :       0.00 B/s | Total DISK WRITE :    1224.74 K/s
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       2.16 M/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                         
 2408 be/3 root        0.00 B/s    7.58 K/s  0.00 %  3.50 % [jbd2/vda1-8]
15114 be/4 mysql       0.00 B/s  716.65 K/s  0.00 %  1.81 % mysqld --defaults-file=/home/mysq~=/home/mysql/data/base/mysql/3306
 5948 be/4 mysql       0.00 B/s    0.00 B/s  0.00 %  0.29 % mysqld --defaults-file=/home/mysq~=/home/mysql/data/base/mysql/3306
14131 be/4 webserve    0.00 B/s   26.54 K/s  0.00 %  0.14 % java -Xms4g -Xmx4g -XX:+UseG1GC -~lasticsearch -d [elasticsearch[t]
14114 b
```

输出的前面两行是系统的读写磁盘速率，接着是每个进程的读写磁盘速率从大到小的排列，排在首位的为读写磁盘最多的进程。

## 从代码角度看磁盘io情况

这里我还是用golang来举例，golang内置有go trace工具，能够对程序中的系统调用延迟进行采样分析，其实和排查由于代码引起的网络问题类似，如果io是由于进程代码不合理的逻辑达到瓶颈，那么必然用go trace去对进程代码进行分析的时候，会发现大部分延迟采点应该都集中在代码那处不合理的读写磁盘逻辑上。

关于go trace的原理以及使用，我在[golang pprof 监控系列(1) —— go trace 统计原理与使用](../pprof_tools/pprof监控系列(1)——go_trace统计原理与使用.md)



