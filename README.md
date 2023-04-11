# performance analyze


大家好，我是蓝胖子。

性能排查，服务监控方面的知识往往涉及量广且比较零散，如何较为系统化的分析和解决问题，建立其对性能排查，性能优化的思路，我将在这个系列里给出我的答案。

整个系列会囊括自己对性能排查的一些思路以及性能分析工具的使用，原理的介绍，也会包含很多线上真实的性能排查案例， 愿往后的性能排查不再抓瞎。



## golang pprof tools  <img src="https://img.shields.io/badge/golang-pprof-green.svg">


> profile的中文被翻译轮廓，对于计算机程序而言，抛开业务逻辑不谈，它的轮廓是是啥呢？不就是cpu，内存，各种阻塞开销，线程，协程概况 这些运行指标或环境。golang语言自带了工具库来帮助我们描述，探测，分析这些指标或者环境信息，让我们来学习它。

本着知其然更知其所以然的想法，本系列也是想在运用go pprof 系列工具的基础之上，明白其中的统计原理，知晓golang里面是如何统计pprof的指标信息的。

以下是内容大纲

[golang pprof 监控系列(1) —— go trace 统计原理与使用](pprof_tools/pprof监控系列(1)——go_trace统计原理与使用.md)


[golang pprof监控系列（2） —— memory，block，mutex 使用]( pprof_tools/pprof监控系列(2)——memory,block,mutex的使用.md)

[golang pprof 监控系列(3) —— memory，block，mutex 统计原理](pprof_tools/pprof监控系列(3)——memory,block,mutex统计原理.md)


[golang pprof 监控系列(4) —— goroutine thread 统计原理]( pprof_tools/pprof监控系列(4)——goroutine_thread统计原理.md)

[golang pprof 监控系列(5) —— cpu 使用 统计原理]( pprof_tools/pprof监控系列(5)——cpu使用率统计原理.md)

[[golang pprof 监控系列(6) 内置的 metric指标 统计原理.md](pprof_tools/pprof监控系列(6)——内置的metric指标统计原理.md)


## 性能排查基础知识 <img src="https://img.shields.io/badge/performance-basic-red.svg">


要想对性能问题进行排查，知晓计算机底层原理已经常用的排查问题的工具很重要，我会在这个系列里给出一些常见的性能问题排查思路，也会介绍大量的工具帮助我们分析性能问题。

大纲如下:

[网络问题排查手段](performance_analysis_basics/(1)网络问题排查手段.md)

[cpu和内存的性能问题 分析思路](performance_analysis_basics/(2)cpu和内存的性能问题分析思路.md)

[io性能问题 分析思路]( performance_analysis_basics/(3)io性能问题分析思路.md)


## 性能排查案例 <img src="https://img.shields.io/badge/performance-cases-orange.svg">


我个人认为性能排查是很考验工程师的水平与经验的，每次性能问题的排查经历都值得认真的复盘与总结，我在这个系列里给出了平时工作中实际遇到的一些性能问题以及我的排查思路。大纲如下:

[mysql invalid conn排查](performance_analysis_cases/(1)mysql_invalid_conn排查.md)

[一次goroutine 泄漏排查案例](performance_analysis_cases/(2)一次goroutine泄漏排查案例.md)


[一次系统延迟性优化案例]( performance_analysis_cases/(3)一次系统延迟性优化案例.md)

[一次排查某某云上的redis读超时经历](performance_analysis_cases/(4)一次排查某某云上的redis读超时经历.md)

[我又和redis超时杠上了]( performance_analysis_cases/(5)我又和redis超时杠上了.md)


## 提问与纠错
如果有疑问或者发现错误，可以在相应的 Issues 进行提问或勘误。

## 公众号

![image.png](https://s2.loli.net/2023/04/11/JvF4AylsPKINegu.jpg)