# golang pprof 监控系列(3) —— memory，block，mutex 统计原理

大家好，我是蓝胖子。

在上一篇文章 [golang pprof监控系列（2） —— memory，block，mutex 使用](https://mp.weixin.qq.com/s/_MWtrXYWm5pzT8Rt3r_YDQ)里我讲解了这3种性能指标如何在程序中暴露以及各自监控的范围。也有提到memory，block，mutex 把这3类数据放在一起讲，是因为他们统计的原理是很类似的。今天来看看它们究竟是如何统计的。

先说下结论，**这3种类型在runtime内部都是通过一个叫做bucket的结构体做的统计**，bucket结构体内部有指针指向下一个bucket 这样构成了bucket的链表，每次分配内存，或者每次阻塞产生时，会判断是否会创建一个新的bucket来记录此次分配信息。

先来看下bucket里面有哪些信息。

## bucket结构体介绍
```go
// src/runtime/mprof.go:48
type bucket struct {
	next    *bucket
	allnext *bucket
	typ     bucketType // memBucket or blockBucket (includes mutexProfile)
	hash    uintptr
	size    uintptr
	nstk    uintptr
}
```
挨个详细解释下这个bucket结构体:
首先是两个指针，**一个next 指针，一个allnext指针**，allnext指针的作用就是形成一个链表结构，刚才提到的每次记录分配信息时，如果新增了bucket，那么这个bucket的allnext指针将会指向 bucket的链表头部。

bucket的链表头部信息是由一个全局变量存储起来的，代码如下:
```go
// src/runtime/mprof.go:140
var (
	mbuckets  *bucket // memory profile buckets
	bbuckets  *bucket // blocking profile buckets
	xbuckets  *bucket // mutex profile buckets
	buckhash  *[179999]*bucket
```

不同的指标类型拥有不同的链表头部变量，mbuckets 是内存指标的链表头，bbuckets 是block指标的链表头，xbuckets 是mutex指标的链表头。

**这里还有个buckethash结构，无论那种指标类型，只要有bucket结构被创建，那么都将会在buckethash里存上一份**，而buckethash用于解决hash冲突的方式则是将冲突的bucket通过指针形成链表联系起来，这个指针就是刚刚提到的next指针了。

至此，解释完了bucket的next指针，和allnext指针，我们再来看看bucket的其他属性。

```go
// src/runtime/mprof.go:48
type bucket struct {
	next    *bucket
	allnext *bucket
	typ     bucketType // memBucket or blockBucket (includes mutexProfile)
	hash    uintptr
	size    uintptr
	nstk    uintptr
}
```

**type** 属性含义很明显了，代表了bucket属于那种指标类型。

**hash** 则是存储在buckethash结构内的hash值，也是在buckethash 数组中的索引值。

**size** 记录此次分配的大小，对于内存指标而言有这个值，其余指标类型这个值为0。

**nstk** 则是记录此次分配时，堆栈信息数组的大小。还记得在上一讲[golang pprof监控系列（2） —— memory，block，mutex 使用](https://mp.weixin.qq.com/s/_MWtrXYWm5pzT8Rt3r_YDQ)里从网页看到的堆栈信息吗。

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
nstk 就是记录的堆栈信息数组的大小，看到这里，你可能会疑惑，**这里仅仅是记录了堆栈大小，堆栈的内容呢？关于分配信息的记录呢？**

要回答这个问题，得搞清楚创建bucket结构体的时候，内存是如何分配的。

首先要明白结构体在进行内存分配的时候是一块连续的内存，例如刚才介绍bucket结构体的时候讲到的几个属性都是在一块连续的内存上，当然，指针指向的地址可以不和结构体内存连续，但是指针本身是存储在这一块连续内存上的。

接着，我们来看看runtime是如何创建一个bucket的。
```go
// src/runtime/mprof.go:162
func newBucket(typ bucketType, nstk int) *bucket {
	size := unsafe.Sizeof(bucket{}) + uintptr(nstk)*unsafe.Sizeof(uintptr(0))
	switch typ {
	default:
		throw("invalid profile bucket type")
	case memProfile:
		size += unsafe.Sizeof(memRecord{})
	case blockProfile, mutexProfile:
		size += unsafe.Sizeof(blockRecord{})
	}

	b := (*bucket)(persistentalloc(size, 0, &memstats.buckhash_sys))
	bucketmem += size
	b.typ = typ
	b.nstk = uintptr(nstk)
	return b
}
```
上述代码是创建一个bucket时源码， 其中persistentalloc 是runtime内部一个用于分配内存的方法，底层还是用的mmap，这里就不展开了，只需要知道该方法可以分配一段内存，size 则是需要分配的内存大小。

persistentalloc返回后的unsafe.Pointer可以强转为bucket类型的指针，unsafe.Pointer是go编译器允许的 代表指向任意类型的指针 类型。所以关键是看 分配一个bucket结构体的时候，这个size的内存空间是如何计算出来的。

首先unsafe.Sizeof 得到分配一个bucket代码结构 本身所需要的内存长度，然后加上了nstk 个uintptr 类型的内存长度 ，uintptr代表了一个指针类型，还记得刚刚提到nstk的作用吗？nstk表明了堆栈信息数组的大小，而数组中每个元素就是一个uintptr类型，指向了具体的堆栈位置。

接着判断 需要创建的bucket的类型，如果是memProfile 内存类型 则又用unsafe.Sizeof 得到一个memRecord的结构体所占用的空间大小，如果是blockProfile，或者是mutexProfile 则是在size上加上一个blockRecord结构体占用的空间大小。memRecord和blockRecord 里承载了此次内存分配或者此次阻塞行为的详细信息。

```go
// src/runtime/mprof.go:59
type memRecord struct {
	active memRecordCycle
	future [3]memRecordCycle
}

// src/runtime/mprof.go:120
type memRecordCycle struct {
	allocs, frees           uintptr
	alloc_bytes, free_bytes uintptr
}
```
关于内存分配的详细信息最后是有memRecordCycle 承载的，里面有此次内存分配的内存大小和分配的对象个数。那memRecord 里的active 和future又有什么含义呢，为啥不干脆用memRecordCycle结构体来表示此次内存分配的详细信息？ 这里我先预留一个坑，放在下面在解释，现在你只需要知道，在分配一个内存bucket结构体的时候，也分配了一段内存空间用于记录关于内存分配的详细信息。

然后再看看blockRecord。
```go
// src/runtime/mprof.go:135
type blockRecord struct {
	count  float64
	cycles int64
}
```
blockRecord 就比较言简意赅，count代表了阻塞的次数，cycles则代表此次阻塞的周期时长，关于周期的解释可以看看我前面一篇文章[golang pprof监控系列（2） —— memory，block，mutex 使用](https://mp.weixin.qq.com/s/_MWtrXYWm5pzT8Rt3r_YDQ) ，简而言之，周期时长是cpu记录时长的一种方式。你可以把它理解成就是一段时间，不过时间单位不在是秒了，而是一个周期。

**可以看到，在计算一个bucket占用的空间的时候，除了bucket结构体本身占用的空间，还预留了堆栈空间以及memRecord或者blockRecord 结构体占用的内存空间大小**。

你可能会疑惑，这样子分配一个bucket结构体，那么如何取出bucket中的memRecord 或者blockRecord结构体呢？ 答案是 通过计算memRecord在bucket 中的位置，然后强转unsafe.Pointer指针。

拿memRecord举例，
```go
//src/runtime/mprof.go:187
func (b *bucket) mp() *memRecord {
	if b.typ != memProfile {
		throw("bad use of bucket.mp")
	}
	data := add(unsafe.Pointer(b), unsafe.Sizeof(*b)+b.nstk*unsafe.Sizeof(uintptr(0)))
	return (*memRecord)(data)
}
```
上面的地址可以翻译成如下公式:
```go
memRecord开始的地址 = bucket指针的地址 +  bucket结构体的内存占用长度 + 栈数组占用长度 
```
这一公式成立的前提便是 分配结构体的时候，是连续的分配了一块内存，所以我们当然能通过bucket首部地址以及中间的空间长度计算出memRecord开始的地址。

至此，bucket的结构体描述算是介绍完了，但是还没有深入到记录指标信息的细节，下面我们深入研究下记录细节，正戏开始。

## 记录指标细节介绍
由于内存分配的采样还是和block阻塞信息的采样有点点不同，所以我还是决定分两部分来介绍下，先来看看内存分配时，是如何记录此次内存分配信息的。

### memory

首先在上篇文章[golang pprof监控系列（2） —— memory，block，mutex 使用](https://mp.weixin.qq.com/s/_MWtrXYWm5pzT8Rt3r_YDQ) 我介绍过 MemProfileRate ，MemProfileRate 用于控制内存分配的采样频率，代表平均每分配MemProfileRate字节便会记录一次内存分配记录。

当触发记录条件时，runtime便会调用 mProf_Malloc 对此次内存分配进行记录，
```go
// src/runtime/mprof.go:340
func mProf_Malloc(p unsafe.Pointer, size uintptr) {
	var stk [maxStack]uintptr
	nstk := callers(4, stk[:])
	lock(&proflock)
	b := stkbucket(memProfile, size, stk[:nstk], true)
	c := mProf.cycle
	mp := b.mp()
	mpc := &mp.future[(c+2)%uint32(len(mp.future))]
	mpc.allocs++
	mpc.alloc_bytes += size
	unlock(&proflock)
	systemstack(func() {
		setprofilebucket(p, b)
	})
}
```
实际记录之前还会先获取堆栈信息，上述代码中stk 则是记录堆栈的数组，然后通过 stkbucket 去获取此次分配的bucket，stkbucket 里会判断是否先前存在一个相同bucket，如果存在则直接返回。而判断是否存在相同bucket则是看存量的bucket的分配的内存大小和堆栈位置是否和当前一致。
```go
// src/runtime/mprof.go:229
for b := buckhash[i]; b != nil; b = b.next {
		if b.typ == typ && b.hash == h && b.size == size && eqslice(b.stk(), stk) {
			return b
		}
	}
```
通过刚刚介绍bucket结构体，可以知道 buckhash 里容纳了程序中所有的bucket，通过一段逻辑算出在bucket的索引值，也就是i的值，然后取出buckhash对应索引的链表，循环查找是否有相同bucket。相同则直接返回，不再创建新bucket。

让我们再回到记录内存分配的主逻辑，stkbucket 方法创建或者获取 一个bucket之后，会通过mp()方法获取到其内部的memRecord结构，然后将此次的内存分配的字节累加到memRecord结构中。

不过这里并不是直接由memRecord 承载累加任务，而是memRecord的memRecordCycle 结构体。

```go
c := mProf.cycle
	mp := b.mp()
	mpc := &mp.future[(c+2)%uint32(len(mp.future))]
	mpc.allocs++
	mpc.alloc_bytes += size
```
这里先是从memRecord 结构体的future结构中取出一个memRecordCycle，然后在memRecordCycle上进行累加字节数，累加分配次数。

这里有必要介绍下mProf.cycle 和memRecord中的active和future的作用了。

我们知道内存分配是一个持续性的过程，内存的回收是由gc定时执行的，golang设计者认为，如果每次产生内存分配的行为就记录一次内存分配信息，那么很有可能这次分配的内存虽然程序已经没有在引用了，但是由于还没有垃圾回收，所以会造成内存分配的曲线就会出现严重的倾斜(因为内存只有垃圾回收以后才会被记录为释放，也就是memRecordCycle中的free_bytes 才会增加，所以内存分配曲线会在gc前不断增大，gc后出现陡降)。

所以，在记录内存分配信息的时候，是将当前的内存分配信息经过一轮gc后才记录下来，mProf.cycle 则是当前gc的周期数，每次gc时会加1，在记录内存分配时，将当前周期数加2与future取模后的索引值记录到future ，而在释放内存时，则将 当前周期数加1与future取模后的索引值记录到future，想想这里为啥要加1才能取到 对应的memRecordCycle呢？ 因为当前的周期数比起内存分配的周期数已经加1了，所以释放时只加1就好。

```go
// src/runtime/mprof.go:362
func mProf_Free(b *bucket, size uintptr) {
	lock(&proflock)
	c := mProf.cycle
	mp := b.mp()
	mpc := &mp.future[(c+1)%uint32(len(mp.future))]
	mpc.frees++
	mpc.free_bytes += size
	unlock(&proflock)
}
```

在记录内存分配时，只会往future数组里记录，那读取内存分配信息的 数据时，怎么读取呢？

还记得memRecord 里有一个类型为memRecordCycle 的active属性吗，在读取的时候，runtime会调用
mProf_FlushLocked()方法，将当前周期的future数据读取到active里。
```go

// src/runtime/mprof.go:59
type memRecord struct {
	active memRecordCycle
	future [3]memRecordCycle
}

// src/runtime/mprof.go:120
type memRecordCycle struct {
	allocs, frees           uintptr
	alloc_bytes, free_bytes uintptr
}


// src/runtime/mprof.go:305
func mProf_FlushLocked() {
	c := mProf.cycle
	for b := mbuckets; b != nil; b = b.allnext {
		mp := b.mp()

		// Flush cycle C into the published profile and clear
		// it for reuse.
		mpc := &mp.future[c%uint32(len(mp.future))]
		mp.active.add(mpc)
		*mpc = memRecordCycle{}
	}
}
```
代码比较容易理解，mProf.cycle获取到了当前gc周期，然后用当前周期从future里取出 当前gc周期的内存分配信息 赋值给acitve ，对每个内存bucket都进行这样的赋值。

赋值完后，后续读取当前内存分配信息时就只读active里的数据了，至此，算是讲完了runtime是如何对内存指标进行统计的。

接着，我们来看看如何对block和mutex指标进行统计的。
### block mutex

block和mutex的统计是由同一个方法，saveblockevent 进行记录的，不过方法内部针对这两种类型还是做了一点点不同的处理。
> 有必要注意再提一下，mutex是在解锁unlock时才会记录一次阻塞行为，而block在记录mutex锁阻塞信息时，是在开始执行lock调用的时候记录的 ，除此以外，block在select 阻塞，channel通道阻塞,wait group 产生阻塞时也会记录一次阻塞行为。

```go
// src/runtime/mprof.go:417
func saveblockevent(cycles, rate int64, skip int, which bucketType) {
	gp := getg()
	var nstk int
	var stk [maxStack]uintptr
	if gp.m.curg == nil || gp.m.curg == gp {
		nstk = callers(skip, stk[:])
	} else {
		nstk = gcallers(gp.m.curg, skip, stk[:])
	}
	lock(&proflock)
	b := stkbucket(which, 0, stk[:nstk], true)

	if which == blockProfile && cycles < rate {
		// Remove sampling bias, see discussion on http://golang.org/cl/299991.
		b.bp().count += float64(rate) / float64(cycles)
		b.bp().cycles += rate
	} else {
		b.bp().count++
		b.bp().cycles += cycles
	}
	unlock(&proflock)
}
```
首先还是获取堆栈信息，然后stkbucket() 方法获取到 一个bucket结构体，然后bp()方法获取了bucket里的blockRecord 结构，并对其count次数和cycles阻塞周期时长进行累加。
```go
// src/runtime/mprof.go:135
type blockRecord struct {
	count  float64
	cycles int64
}
```
注意针对blockProfile 类型的次数累加 还进行了特别的处理，还记得上一篇[golang pprof监控系列（2） —— memory，block，mutex 使用](https://mp.weixin.qq.com/s/_MWtrXYWm5pzT8Rt3r_YDQ)提到的BlockProfileRate参数吗，它是用来设置block采样的纳秒采样率的，如果阻塞周期时长cycles小于BlockProfileRate的话，则需要fastrand函数乘以设置的纳秒时间BlockProfileRate 来决定是否采样了，所以如果是小于BlockProfileRate 并且saveblockevent进行了记录阻塞信息的话，说明我们只是采样了部分这样情况的阻塞，所以次数用BlockProfileRate 除以 此次阻塞周期时长数，得到一个估算的总的 这类阻塞的次数。

读取阻塞信息就很简单了，直接读取阻塞bucket的count和周期数即可。

## 总结

至此，算是介绍完了这3种指标类型的统计原理，简而言之，就是通过一个携带有堆栈信息的bucket对每次内存分配或者阻塞行为进行采样记录，读取内存分配信息 或者阻塞指标信息的 时候便是所有的bucket信息读取出来。



