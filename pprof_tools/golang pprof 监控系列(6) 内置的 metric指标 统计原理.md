# golang pprof 监控系列（6） 内置的 metric指标 统计原理

大家好，我是蓝胖子，前面几个系列介绍完了golang pprof 工具各种指标的统计原理。


[golang pprof 监控系列(1) —— go trace 统计原理与使用](https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(1)%20%E2%80%94%E2%80%94%20go%20trace%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86%E4%B8%8E%E4%BD%BF%E7%94%A8.md)

[golang pprof监控系列（2） —— memory，block，mutex 使用](https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(2%EF%BC%89%20%E2%80%94%E2%80%94%20%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E4%BD%BF%E7%94%A8.md )

[golang pprof 监控系列(3) —— memory，block，mutex 统计原理](https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(3)%20%E2%80%94%E2%80%94%20memory%EF%BC%8Cblock%EF%BC%8Cmutex%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md )

[golang pprof 监控系列(4) —— goroutine thread 统计原理]( https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(4)%20%E2%80%94%E2%80%94%20goroutine%20thread%20%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md)

[golang pprof 监控系列(5) —— cpu 使用 统计原理]( https://github.com/HobbyBear/pprof/blob/main/golang%20pprof%20tools/golang%20pprof%20%E7%9B%91%E6%8E%A7%E7%B3%BB%E5%88%97(5)%20%E2%80%94%E2%80%94%20cpu%20%E4%BD%BF%E7%94%A8%20%E7%BB%9F%E8%AE%A1%E5%8E%9F%E7%90%86.md )



对于pprof 工具的理解应该能够建立起比较完善的知识体系了，这一节算是分析golang内置监控工具的一个彩蛋。除了使用golang pprof 工具以外对go程序进行监控，golang还内置了很多metric指标信息。如下是这些metric初始化的部分。

```go
// src/runtime/metrics.go:88
metrics = map[string]metricData{
		"/gc/cycles/automatic:gc-cycles": {
			deps: makeStatDepSet(sysStatsDep),
			compute: func(in *statAggregate, out *metricValue) {
				out.kind = metricKindUint64
				out.scalar = in.sysStats.gcCyclesDone - in.sysStats.gcCyclesForced
			},
		},
		"/gc/cycles/forced:gc-cycles": {
			deps: makeStatDepSet(sysStatsDep),
			compute: func(in *statAggregate, out *metricValue) {
				out.kind = metricKindUint64
				out.scalar = in.sysStats.gcCyclesForced
			},
		},
.....
```
指标反映的监控范围涉及到gc，内存，协程调度等runtime内部的信息，这里不去追究具体某个指标在哪里进行统计，而是去分析golang如何去设计metric指标监控这一模块的。
## metrics 设计原理

不过，在分析设计原理之前先来看看如何在代码里 获取到这些指标信息。

```go
const myMetric = "/memory/classes/heap/free:bytes"

	// Create a sample for the metric.
	sample := make([]metrics.Sample, 1)
	sample[0].Name = myMetric

	// Sample the metric.
	metrics.Read(sample)
```
首先定义一个metrics.Sample 类型的数组，将数组传入metrics.Read 方法，然后方法内部就会根据传入数组中的每个元素的Name值，为其赋值上Value 值，Name是指标的名字，例如 /gc/cycles/automatic:gc-cycles ，Value就是我们需要的具体指标信息了。

使用方式很简单，调用metrics.Read方法读取就行，接下来我们仔细分析下metrics的设计原理，好的，接下来，正戏开始。

metrics.Sample类型如下,

```go
// src/runtime/metrics/sample.go:13
type Sample struct {
	Name string
	Value Value
}
```
metrics.Sample 的Value 值是个metrics.Value类型,如下，

```go
// src/runtime/metrics/value.go:15 
const (
	KindBad ValueKind = iota
	KindUint64
	KindFloat64
	KindFloat64Histogram
)
type Value struct {
	kind    ValueKind
	scalar  uint64         // contains scalar values for scalar Kinds.
	pointer unsafe.Pointer // contains non-scalar values.
}
```
从源码可以看出 metrics.Value 分为4个种类，准确说是3种类型，KindBad 是当获取的指标没有被runtime定义时，填充kind字段 时才会用到，实际真正的指标分为3种类型，KindUint64 代表值是一个uint64类型，KindFloat64 代表值是一个float64类型，KindFloat64Histogram 代表值是一个float64的直方图类型。

但其实在runtime内部用到的类型不是metrics.Value ，metrics.Sample类型，而是metricSample，和metricValue类型，它们在内存布局上是一致的。结构如下，
```go
// src/runtime/metrics.go:516
type metricSample struct {
	name  string
	value metricValue
}

// metricValue is a runtime copy of runtime/metrics.Sample and
// must be kept structurally identical to that type.
type metricValue struct {
	kind    metricKind
	scalar  uint64         // contains scalar values for scalar Kinds.
	pointer unsafe.Pointer // contains non-scalar values.
}
```
由于在内存布局上一致，所以metrics.Read 方法通过unsafe.Pointer指针将metrics.Sample 强转成metricSample了。

搞懂了metrics.Sample 外部使用类型和metrics.Sample runtime内部使用类型的关系后，我们来仔细看看metrics.Read 方法是如何获取指标信息的。

```go
// src/runtime/metrics/sample.go:45
func Read(m []Sample) {
	runtime_readMetrics(unsafe.Pointer(&m[0]), len(m), cap(m))
}
```

metrics.Read 调用了metrics.runtime_readMetrics 方法，而metrics.runtime_readMetrics 是靠go:linkname标签将readMetrics 方法链接过去的。所以来看下readMetrics的逻辑。

```go
// src/runtime/metrics.go:564
//go:linkname readMetrics runtime/metrics.runtime_readMetrics
func readMetrics(samplesp unsafe.Pointer, len int, cap int) {
	sl := slice{samplesp, len, cap}
	samples := *(*[]metricSample)(unsafe.Pointer(&sl))
	metricsLock()
	initMetrics()
	agg = statAggregate{}
	for i := range samples {
		sample := &samples[i]
		// initMetrics  初始化了metrics数组，metrics存储runtime暴露的指标信息
		data, ok := metrics[sample.name]
		if !ok {
			sample.value.kind = metricKindBad
			continue
		}
		agg.ensure(&data.deps)
		data.compute(&agg, &sample.value)
	}
	metricsUnlock()
}
```
先看**readMetrics**的参数 samplesp， samplesp便是我们传递进来的metrics.Sample 数组了，由于metrics.Sample和metricSample结构内存布局一致，所以可以将数组强转为metricSample 数组类型。

接着调用了**initMetrics**方法对对指标信息进行了初始化，注意这里的初始化是指明runtime内部有哪些指标暴露出来，计算指标时依赖的 底层如内存大小，gc时间这些信息是程序运行过程中统计的。代码中metrics 这个map结构便是initMetrics调用时 初始化的。metrics存储runtime暴露的指标信息。

initMetrics 初始化指标信息后，是 定义了 一个**statAggregate** 类型，
```go
//src/runtime/metrics.go:472
type statAggregate struct {
	ensured   statDepSet
	heapStats heapStatsAggregate
	sysStats  sysStatsAggregate
}
```
**statAggregate** 是确认 用户需要获取的指标中 依赖了哪些底层统计信息，例如内存释放字节数或者gc周期次数等等，并在开始计算指标值之前，提前获取到这些信息。**agg.ensure** 方法就是 确认并获取底层信息的方法。同样类型的底层类型不会重复获取，这里分为了两种类型，一个heap类型，一个是sys类型。

接着**data.compute**就是调用特定指标compute 方法将这些底层统计信息进行归纳汇总 形成指标值。
例如下面这段逻辑便是名为/sched/goroutines:goroutines 的指标 获取的值则为当前的协程数量，类型是metricKindUint64类型。
```go
// src/runtime/metrics.go:297
"/sched/goroutines:goroutines": {
			compute: func(_ *statAggregate, out *metricValue) {
				out.kind = metricKindUint64
				out.scalar = uint64(gcount())
			},
		},
```

## 总结
至此，golang runtime 是如何设计 metrics 这个模块的？这个问题应该能回答了。

简而言之，runtime将metrics归为3种类型，其实也可以说成是2种：

一种是数值类型，数值类型分别是整形和浮点类型，分别是 metricKindUint64和metricKindFloat64，数值类型可以是累加值，也可以是当前值。

一种是直方图类型，这类型一般用于统计百分位数据，比如协程调度延迟时间，gc停顿时间 p99，p95的值，诸如此类。

> 数值类型比较好理解，没有过多需要讲的，但golang在设计metrics的直方图这部分比较巧妙，下一篇再特别讲讲这部分的实现思路。

之后，在需要读取指标信息时，才会将底层的统计数据 进行归纳聚合，形成特定的指标值。

