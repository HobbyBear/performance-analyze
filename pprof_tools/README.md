# golang pprof tools

大家好，我是蓝胖子。

> profile的中文被翻译轮廓，对于计算机程序而言，抛开业务逻辑不谈，它的轮廓是是啥呢？不就是cpu，内存，各种阻塞开销，线程，协程概况 这些运行指标或环境。golang语言自带了工具库来帮助我们描述，探测，分析这些指标或者环境信息，让我们来学习它。

本着知其然更知其所以然的想法，本系列也是想在运用go pprof 系列工具的基础之上，明白其中的统计原理，知晓golang里面是如何统计pprof的指标信息的。

以下是内容大纲

[golang pprof 监控系列(1) —— go trace 统计原理与使用](https://github.com/HobbyBear/performance-analyze/blob/119f4e6cabe5190eb692e79983e0766cab8481f5/pprof_tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(1)%20%E2%80%94%E2%80%94%20go%20trace%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86%E4%B8%8E%E4%BD%BF%E7%94%A8.md )

[golang pprof监控系列（2） —— memory，block，mutex 使用]( https://github.com/HobbyBear/performance-analyze/blob/119f4e6cabe5190eb692e79983e0766cab8481f5/pprof_tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(2%EF%BC%89%20%E2%80%94%E2%80%94%20%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E4%BD%BF%E7%94%A8.md )

[golang pprof 监控系列(3) —— memory，block，mutex 统计原理](https://github.com/HobbyBear/performance-analyze/blob/119f4e6cabe5190eb692e79983e0766cab8481f5/pprof_tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(3)%20%E2%80%94%E2%80%94%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md)

[golang pprof 监控系列(4) —— goroutine thread 统计原理]( https://github.com/HobbyBear/performance-analyze/blob/119f4e6cabe5190eb692e79983e0766cab8481f5/pprof_tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(4)%20%E2%80%94%E2%80%94%20goroutine%20thread%20%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md) 

[golang pprof 监控系列(5) —— cpu 使用 统计原理]( https://github.com/HobbyBear/performance-analyze/blob/119f4e6cabe5190eb692e79983e0766cab8481f5/pprof_tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(5)%20%E2%80%94%E2%80%94%20cpu%20%E4%BD%BF%E7%94%A8%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md )

[[golang pprof 监控系列(6) 内置的 metric指标 统计原理.md](https://github.com/HobbyBear/performance-analyze/blob/119f4e6cabe5190eb692e79983e0766cab8481f5/pprof_tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(6)%20%E5%86%85%E7%BD%AE%E7%9A%84%20metric%E6%8C%87%E6%A0%87%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md)
