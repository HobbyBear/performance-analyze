# golang pprof监控系列（2） ——  memory，block，mutex 使用

大家好，我是蓝胖子。

> profile的中文被翻译轮廓，对于计算机程序而言，抛开业务逻辑不谈，它的轮廓是是啥呢？不就是cpu，内存，各种阻塞开销，线程，协程概况 这些运行指标或环境。golang语言自带了工具库来帮助我们描述，探测，分析这些指标或者环境信息，让我们来学习它。

在上一篇[golang pprof 监控系列(1) —— go trace 统计原理与使用](https://mp.weixin.qq.com/s/2BY_93w8iAJOGpsCdgbGtA)  里我讲解了下golang trace 工具的统计原理，它能够做到对程序运行中的各种事件进行耗时统计。而今天要将的memory,block,mutex 统计的原理与之不同，这是一个在一段时间内进行采样得到的累加值。

还记得之前用go trace 生成的网页图吗?

![trace 网页显示](https://s2.loli.net/2023/03/24/LVef5C3QvgsdSj8.png)

里面是不是也有 3个名字带有blocking的 profile的轮廓图，分别是Network blocking profile，Synchronization blocking profile，Syscall blocking profile ，而它与今天要说的block 统计有没有什么关联？ **先说下结论，block统计的内容有重合，但是不多，并且统计数据来源是不同的**。

> Synchronization blocking profile 和 今天的block统计 是一致的

然后之所以 memory，block，mutex 把这3类数据放在一起讲，是因为他们统计的原理是很类似的，好的,在了解统计原理之前，先简单看下如何使用golang提供的工具对这些数据类型进行分析。

## 如何使用

在看使用代码前，还需要了解清楚对这3种类型的指标对哪些数据进行统计。


两种方式都比较常见，首先来看下http 接口暴露这些性能指标的方式。
### http 接口暴露的方式
```go
package main
import (
	"log"
		"net/http"
	_ "net/http/pprof"
)
func main() {
		log.Println(http.ListenAndServe(":6060", nil))
}
```
使用方式相当容易，直接代码引入net/http/pprof ，便在默认的http处理器上注册上了相关路由，引入包的目的就是为了调用对应包的init方法注册路由。下面就是引用包的init方法。
```go
// src/net/http/pprof/pprof.go:80
func init() {
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
}
```
接下来访问路由 http://127.0.0.1:6060/debug/pprof/  便能看到下面网页内容了。

![image.png](https://s2.loli.net/2023/03/28/dJB3NVW8u5nOUL1.png)

标注为红色的部分就是今天要讲的内容，点击它们的链接会跳到对应的网页看到统计信息。我们挨个来看下。

#### allocs ,heap

这两个值都是记录程序内存分配的情况。

```go
heap profile: 7: 5536 [110: 2178080] @ heap/1048576
2: 2304 [2: 2304] @ 0x100d7e0ec 0x100d7ea78 0x100d7f260 0x100d7f78c 0x100d811cc 0x100d817d4 0x100d7d6dc 0x100d7d5e4 0x100daba20
#	0x100d7e0eb	runtime.allocm+0x8b		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:1881
#	0x100d7ea77	runtime.newm+0x37		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:2207
#	0x100d7f25f	runtime.startm+0x11f		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:2491
#	0x100d7f78b	runtime.wakep+0xab		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:2590
#	0x100d811cb	runtime.resetspinning+0x7b	/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:3222
#	0x100d817d3	runtime.schedule+0x2d3		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:3383
#	0x100d7d6db	runtime.mstart1+0xcb		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:1419
#	0x100d7d5e3	runtime.mstart0+0x73		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:1367
#	0x100daba1f	runtime.mstart+0xf		/Users/lanpangzi/goproject/src/go/src/runtime/asm_arm64.s:117
```

在 go1.17.12 这两者的信息输出其实是一样的，实现的代码也基本一致，仅仅是在名称上做了区分

![image.png](https://s2.loli.net/2023/03/28/a6wIiRxsPDhSo5v.png)

按pprof http服务器启动网页上的显示来说，allocs 是过去内存的采样，heap 是存活对象的采用记录，然而实际上在代码实现上两者是一致的。并没有做区分。

下面来讲下网页输出内容的含义

```go
heap profile: 7: 5536 [110: 2178080] @ heap/1048576
```
输出的第一行含义分别是:

7 代表 当前活跃的对象个数

5536 代表 当前活跃对象占用的字节数

110 代表 所有(包含历史的对象)对象个数

2178080 代表 所有对象(包含历史的对象)占用的对象字节数

1048576  控制了内存采样的频率，1048576 是两倍的内存采样频率的大小，默认采样频率是512kb 即平均每512kb就会采样一次，注意这个值512kb不是绝对的达到512kb就会进行采样，而是从一段时间内的采样来看的一个平均值。

接下来就是函数调用堆栈信息，来看第一行

```go
2: 2304 [2: 2304] @ 0x100d7e0ec 0x100d7ea78 0x100d7f260 0x100d7f78c 0x100d811cc 0x100d817d4 0x100d7d6dc 0x100d7d5e4 0x100daba20
```

从左往右看:

2 代表 在该函数栈上当前活跃的对象个数

2304 代表 在该函数栈上当前活跃的对象所占用的字节数

方括号内的2   代表 在该函数栈上所有(包含历史的对象)对象个数

方括号内的2304 代表 在该函数栈上所有(包含历史的对象)对象所占用的字节数

然后是栈上pc寄存器的值。再往后就是具体的栈函数名信息了。


#### block

接下来看看block会对哪些行为产生记录，每次程序锁阻塞发生时，select 阻塞，channel通道阻塞,wait group 产生阻塞时就会记录一次阻塞行为。对阻塞行为的记录其实是和trace 的Synchronization blocking profile 是一致的，但是和其他两个blocking profile 不一样。

要得到block的输出信息首先要开启下记录block的开关.
```go
   // 1 代表 全部采样，0 代表不进行采用， 大于1则是设置纳秒的采样率
	runtime.SetBlockProfileRate(1)
``` 

这个采样率是怎样计算的，我们来看下具体代码。
```go
// src/runtime/mprof.go:409
func blocksampled(cycles, rate int64) bool {
	if rate <= 0 || (rate > cycles && int64(fastrand())%rate > cycles) {
		return false
	}
	return true
}
```
cycles你可以把它理解成也是一个纳秒级的事件，rate就是我们设置的纳秒时间，如果 cycles大于等于rate则直接记录block行为，如果cycles小于rate的话，则需要fastrand函数乘以设置的纳秒时间rate 来决定是否采样了。


然后回过头来看看网页的输出信息
```go
--- contention:
cycles/second=1000000000
180001216583 1 @ 0x1002a1198 0x1005159b8 0x100299fc4
#	0x1002a1197	sync.(*Mutex).Lock+0xa7	/Users/lanpangzi/goproject/src/go/src/sync/mutex.go:81
#	0x1005159b7	main.main.func2+0x27	/Users/lanpangzi/goproject/src/go/main/main.go:33
```

contention 是为这个profile文本信息取的名字，总之中文翻译是争用。

cycles/second 是每秒钟的周期数，用它来表示时间也是为了更精确，其实你可以发现在我的机器上每秒是10的9次方个周期，换算成纳秒就是1ns一个周期时间了。

接着的180001216583 是阻塞的周期数，其实周期就是cputicks，那么180001216583除以 cycles/second 1000000000得到的就是阻塞的秒数了。

接着 1代表阻塞的次数。

无论是阻塞周期时长还是次数，都是一个累加值，即在相同的地方阻塞会导致这个值变大，并且次数会增加。剩下的部分就是函数堆栈信息了。

## mutex

接着来看mutex相关内容，block也在锁阻塞时记录阻塞行为，那么mutex与它有什么不同？

直接说下结论，mutex是在解锁unlock时才会记录一次阻塞行为，而block在记录mutex锁阻塞信息时，是在开始执行lock调用的时候记录的。

要想记录mutex信息，和block类似，也需要开启mutex采样开关。

```go
   // 0 代表不进行采用， 1则全部采用，大于1则是一个随机采用
   	runtime.SetMutexProfileFraction(1)
```
来看看采样的细节代码，代码版本go1.17.12
```go
if rate > 0 && int64(fastrand())%rate == 0 {
		saveblockevent(cycles, rate, skip+1, mutexProfile)
	}
```
可以看到fastrand() 与rate取模等于0才会去采样，rate如果设置成1，则任何数与整数与1取模都会得到0，所以设置为1为完全采用，大于1比如rate设置为2，则要求fastrand必须是2的倍数才能被采样。


接着来看下网页的输出信息。
```go
--- mutex:
cycles/second=1000000812
sampling period=1
180001727833 1 @ 0x100b9175c 0x100e05840 0x100b567ec 0x100b89fc4
#	0x100b9175b	sync.(*Mutex).Unlock+0x8b	/Users/lanpangzi/goproject/src/go/src/sync/mutex.go:190
#	0x100e0583f	main.main+0x19f			/Users/lanpangzi/goproject/src/go/main/main.go:39
#	0x100b567eb	runtime.main+0x25b		/Users/lanpangzi/goproject/src/go/src/runtime/proc.go:255
```

第一行mutex就是profile文本信息的名称了，同样也和block一样，采用cpu周期数计时，但是多了一个sampling period ，这个就是我们设置的采用频率。

接下来的数据都和block类似，180001727833就是锁阻塞的周期， 1为解锁的次数。然后是解锁的堆栈信息。


介绍完利用http服务查看pprof性能指标的方式来看看，如何用代码来生成这些pprof的二进制或者文本文件。
### 代码生成profile文件的方式

首先来看看如何对内存信息生成pprof文件。
```go
os.Remove("heap.out")
	f, _ := os.Create("heap.out")
	defer f.Close()

	err := pprof.Lookup("heap").WriteTo(f, 1)
	if err != nil {
		log.Fatal(err)
	}
```
lookup 传递的heap参数也可以改成allocs，不过输出的内容是一致的。

WriteTo 第二个参数叫做debug，传1代表 会生成可读的文本文件，就和http服务看heap allocs的网页一样的。传0代表是要生成一个二进制文件。生成的二进制文件可以用go tool pprof pprof文件名 去分析，关于go tool pprof 工具的使用网上有相当多的资料，这里就不再展开了。

然后来看看如何生成block的pprof文件。
```go
runtime.SetBlockProfileRate(1)
	os.Remove("block.out")
	f, _ := os.Create("block.out")
	defer f.Close()

	err := pprof.Lookup("block").WriteTo(f, 1)
	if err != nil {
		log.Fatal(err)
	}
```
pprof.Lookup方法传递block即可生成文件了，使用方式和生成内存分析文件一致，，传1代表 会生成可读的文本文件，就和http服务看block的网页一样的。传0代表是要生成一个二进制文件。生成的二进制文件可以用go tool pprof pprof文件名 去分析。

最后看看mutex，和前面两者基本一致仅仅是Lookup传递的参数有变化，这里就不再叙述了。
```go
	runtime.SetMutexProfileFraction(1)
	os.Remove("mutex.out")
	f, _ := os.Create("mutex.out")
	defer f.Close()

	err := pprof.Lookup("mutex").WriteTo(f, 1)
	if err != nil {
		log.Fatal(err)
	}
```

## 总结

经过上述的讲解，应该是能够明白相关的指标究竟是采集了哪些内容并且何时采集的了，而go tool pprof 工具的使用不是此文的重点，所以我直接省略掉了，也是没想到即使省略了，还是内容有那么多，那就把这部分的统计原理留到下一篇文章了，下一篇，memory，block，mutex  统计的原理介绍。




