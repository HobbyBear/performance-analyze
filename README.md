# performance analyze
大家好，我是蓝胖子。
性能优化，服务监控方面的知识往往涉及量广且比较零散，希望将这部分知识整理成册，愿往后性能排查不再抓瞎。


## golang pprof tools

> profile的中文被翻译轮廓，对于计算机程序而言，抛开业务逻辑不谈，它的轮廓是是啥呢？不就是cpu，内存，各种阻塞开销，线程，协程概况 这些运行指标或环境。golang语言自带了工具库来帮助我们描述，探测，分析这些指标或者环境信息，让我们来学习它。

本着知其然更知其所以然的想法，本系列也是想在运用go pprof 系列工具的基础之上，明白其中的统计原理，知晓golang里面是如何统计pprof的指标信息的。

以下是内容大纲

[golang pprof 监控系列(1) —— go trace 统计原理与使用](https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(1)%20%E2%80%94%E2%80%94%20go%20trace%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86%E4%B8%8E%E4%BD%BF%E7%94%A8.md)

[golang pprof监控系列（2） —— memory，block，mutex 使用](https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(2%EF%BC%89%20%E2%80%94%E2%80%94%20%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E4%BD%BF%E7%94%A8.md )

[golang pprof 监控系列(3) —— memory，block，mutex 统计原理](https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(3)%20%E2%80%94%E2%80%94%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md )

[golang pprof 监控系列(4) —— goroutine thread 统计原理]( https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(4)%20%E2%80%94%E2%80%94%20goroutine%20thread%20%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md)

[golang pprof 监控系列(5) —— cpu 使用 统计原理]( https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(5)%20%E2%80%94%E2%80%94%20cpu%20%E4%BD%BF%E7%94%A8%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md )


## 性能排查基础知识

要想对性能问题进行排查，知晓计算机底层原理已经常用的排查问题的工具很重要，我会在这个系列里给出一些常见的性能问题排查思路，也会介绍大量的工具帮助我们分析性能问题。

大纲如下:

[网络问题排查手段]( https://github.com/HobbyBear/pprof/blob/main/Performance%20Troubleshooting%20Basics/%E7%BD%91%E7%BB%9C%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5%E6%89%8B%E6%AE%B5.md )

[mysql性能分析手段](https://github.com/HobbyBear/pprof/blob/main/Performance%20Troubleshooting%20Basics/mysql%20%E6%80%A7%E8%83%BD%E5%88%86%E6%9E%90.md )


## 性能排查案例

我个人认为性能排查是很考验工程师的水平与经验的，每次性能问题的排查经历都值得认真的复盘与总结，我在这个系列里给出了平时工作中实际遇到的一些性能问题以及我的排查思路。大纲如下:

[mysql invalid conn排查](https://github.com/HobbyBear/pprof/blob/main/Performance%20troubleshooting%20case/mysql%20invalid%20conn%E6%8E%92%E6%9F%A5.md)

[一次goroutine 泄漏排查案例](https://github.com/HobbyBear/pprof/blob/main/Performance%20troubleshooting%20case/%E4%B8%80%E6%AC%A1goroutine%20%E6%B3%84%E6%BC%8F%E6%8E%92%E6%9F%A5%E6%A1%88%E4%BE%8B.md)

[一次排查某某云上的redis读超时经历](https://github.com/HobbyBear/pprof/blob/main/Performance%20troubleshooting%20case/%E4%B8%80%E6%AC%A1%E6%8E%92%E6%9F%A5%E6%9F%90%E6%9F%90%E4%BA%91%E4%B8%8A%E7%9A%84redis%E8%AF%BB%E8%B6%85%E6%97%B6%E7%BB%8F%E5%8E%86.md)

[一次系统延迟性优化案例]( https://github.com/HobbyBear/pprof/blob/main/Performance%20troubleshooting%20case/%E4%B8%80%E6%AC%A1%E7%B3%BB%E7%BB%9F%E5%BB%B6%E8%BF%9F%E6%80%A7%E4%BC%98%E5%8C%96%E6%A1%88%E4%BE%8B.md)

[我又和redis超时杠上了]( https://github.com/HobbyBear/pprof/blob/main/Performance%20troubleshooting%20case/%E6%88%91%E5%8F%88%E5%92%8Credis%E8%B6%85%E6%97%B6%E6%9D%A0%E4%B8%8A%E4%BA%86.md)

