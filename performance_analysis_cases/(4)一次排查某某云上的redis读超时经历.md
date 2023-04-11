# 一次排查某某云上的redis读超时经历

## 问题背景
最近一两天线上老是偶现的redis读超时报警，并且是**业务低峰期间**，甚是不解，于是开始着手排查。

以下是我的排查思路。
## 排查思路

### 查阅 redis 慢查询日志
既然是redis超时，首先想到的还是 对于redis的操作命令存在慢查询导致的。
![image.png](https://s2.loli.net/2023/03/07/GafTLxnlztUBO6Y.png)
redis的慢查询阈值是10ms,唯一的慢查询是备份时的bgrewriteaof语句，并不是业务命令，既然从慢查询很日志看不出端倪，那就从redis服务器本身查找问题，所以我又去看了redis服务机器的各项硬件指标。

### 检查 报警期间 redis 各项负载指标
看了下各项监控指标，cpu，内存，qps等等，毫无意外的正常。(这里就不放图了，应为太正常了，分析意义不大)

既然服务端看不出毛病，那是不是客户端的问题，于是我又去检查了我们ecs服务器这边机器的情况。
### 检查报警期间 ecs服务器的各项指标数据
cpu,内存，带宽等等也是正常且处于较低水平。

### 排查到这里，重新思考慢查询日志究竟是什么？
慢查询记录的真的是redis命令执行的所有时间吗？redis命令完整的执行过程究竟是怎样的？
慢查询日志仅仅是记录了命令的执行时间，而整个redis命令的生命周期是这样。
```shell
客户端命令发送->redis服务器接收到命令，放入队列排队执行->命令执行->返回给客户端结果
```
#### 那么有没有办法监控到redis的延迟呢?如何才能知道redis的命令慢不是因为执行慢，而是这个过程当中的其他流程慢导致的呢?
redis 提供了监控延迟的工具
开启延迟，设置延迟阀值
```shell
CONFIG SET latency-monitor-threshold 100
```
查阅延迟原因
```shell
latency latest
```
但是这个工具真正实践起来的效果并不能让我满意，因为似乎他并不能把网络因素考虑其中，实践起来的效果它应该是只能将命令执行过程中可能导致延迟的原因分析出来，但是执行前以及执行后的命令生命周期的阶段并没有考虑进去。

当我开启这个工具监控线上redis情况，当又有读超时出现时，latency latest 并没有返回任何延迟异常。

### 再思考究竟读超时是个什么问题？
客户端发出去了命令，然后阻塞等待redis服务端读的结果，如果没有结果返回，就会触发读超时发生。在go里面代码是如何实现的。

> 我们用的redis客户端是go-redis
```go
func (c *baseClient) pipelineProcessCmds(cn *pool.Conn, cmds []Cmder) (bool, error) {
	err := cn.WithWriter(c.opt.WriteTimeout, func(wr *proto.Writer) error {
		return writeCmd(wr, cmds...)
	})
	if err != nil {
		setCmdsErr(cmds, err)
		return true, err
	}
    // 看到将读取操作用WithReader装饰了一下
	err = cn.WithReader(c.opt.ReadTimeout, func(rd *proto.Reader) error {
		return pipelineReadCmds(rd, cmds)
	})
	return true, err
}
```
读取redis服务端响应的方法 是用cn.WithReader进行了装饰。
```go
// 读取之前设置了超时时间
func (cn *Conn) WithReader(timeout time.Duration, fn func(rd *proto.Reader) error) error {
	_ = cn.setReadTimeout(timeout)
	return fn(cn.rd)
}
```
而cn.WithReader 里，首先便是设置此次读取的超时时间。如果在规定的超时时间内，需要读取的结果没有全部返回也会导致读超时的发生，**那么会不会是由于返回结果过多导致读取耗时验证呢?**

具体的分析了下报警出错的命令，有些命令比如set命令不需要返回结果都有超时的情况，所以排除掉了返回结果过大的问题。

### 再次深入思考golang 里的读超时触发过程
go协程在碰到网络读取时，协程会挂起，等待网络包到达后，由go runtime唤醒协程，然后协程继续进行读取操作，当唤醒时会检查超时时间，如果到达了超时限制，那么将直接报读超时错误。(有机会可以详细分析下golang的netpoll源码) 源码如下，
```go
src/runtime/netpoll.go:303
for !netpollblock(pd, int32(mode), false) {
		errcode = netpollcheckerr(pd, int32(mode))
		if errcode != pollNoError {
			return errcode
		}
		// Can happen if timeout has fired and unblocked us,
		// but before we had a chance to run, timeout has been reset.
		// Pretend it has not happened and retry.
	}
```
netpollblock 不阻塞协程时，首先执行了netpollcheckerr，netpollcheckerr检查是否有超时情况发生。

从唤醒到协程被调度执行的这个时间称为协程的调度延迟，如果这个延迟过高，那么是有可能发生读超时的。
于是我又看了go进程中协程的调度延迟，在golang里 内置了一个/sched/latencies:seconds 指标，代表协程调度延迟，目前的prometheus client 已经对这个指标进行了兼容，所以我们是直接利用它 将延迟耗时在grafana里进行了展示。

超时期间，grafana里的 协程调度延迟只有几毫秒。而超时时间设置的200ms，显然也不是协程调度延迟的问题。

### 用上终极武器-抓包
以上的思路都行不通了，所以只能用上终极武器，抓包。 看看触发200ms超时时 究竟是哪些包没有到达。因为只能在客户端测抓包，所以接下来的抓包文件都是客户端测的抓包。

> 很对时候抓包都是解决问题特别是网络问题的最终手段，你能通过抓包清楚的看到客户端，服务端在做什么事情。

为了百分百确认并且定位问题，我一共抓取了3个文件，首先来看下第一个文件。
![超时时间为200ms时的抓包文件](https://s2.loli.net/2023/03/07/J3FeqOStaX4QHiW.png)

6379端口号是目的端口，也就是redis的端口，36846是我客户端的端口。

从抓包文件中，发现760054号报文发生了超时重传，如果客户端发了包，但是**服务端没有回应ack消息就会触发超时重传**，重传之后，客户端也没有收到服务端的消息，并且可以看到发送挥手信号和前一个正常发送的包之间刚好是隔了差不多200ms，而200ms正是客户端设置的超时时间，应用层触发超时后，将调用close方法关闭链接，所以在760055号包里 客户端发送了一个fin 挥手信号代表客户端要关闭链接了。 客户端发送fin信号挥手之后呢，服务端才发来携带数据的ack消息，不过由于此时客户端已经处于要关闭状态，所以直接发送rst信号了。

整个发包和接包流程可以用下面的流程图来展示。

![image.png](https://s2.loli.net/2023/04/08/qxry9KEVHfCDZFt.png)

接着来看第二个抓包文件。
![200ms抓包分析](https://s2.loli.net/2023/03/07/tB2RxOVcGCYHlkQ.png)

抓包中出现大量TCP Dup Ack 的消息，客户端一直在向端口为6379的服务端发送ack的序号为 13364573,代表客户端已经接收到服务端序号13364573之前的包了,然而服务端连续发送的包序号seq都大于了13364573 ，所以客户端认为服务端序号seq是13364573的包丢了，所以随着服务端每一次发送消息过来，都告诉服务端，我应该接收序号是13364573开始的包，赶紧发送过来。

最终在1777232号包，客户端又一次发送了TCP Dup Ack 消息，**催促服务端赶紧把丢掉的包发过来，不过这次服务端连回应都不回应了，最终触发了客户端应用层200ms的超时时间**，调用close方法关闭了连接。所以在1778166号包里，可以看到客户端发送fin挥手信号，1778166 号包的发送时间和1777232号包的发送时间正式相隔了200ms。

整个发包和接包流程可以用下面的流程图来展示。
![image.png](https://s2.loli.net/2023/04/08/XNh5iAsTVegwYSb.png)

再来看第三个抓包文件，第三个抓包文件是我将客户端超时时间设置为500ms后出现超时情况时抓到的。
![500ms超时抓包](https://s2.loli.net/2023/03/07/4U185SolBjexVyn.png)


首先看第一个红色箭头处，也就是911752号包，它被wireshark标记为 Tcp Previous segment not captured，表示这个包之前的包没有被捕获到，说明这个包seq序号之前的包存在丢包情况。发现它前一个包也就是911751号包也是服务端发来的包，并且next seq 是18428124，说明911751号包下一个包的seq应该是18428124，但是911752的seq是18429584，**进一步说明 来自服务端的包序号在18428124到18429584之间的包存在丢包情况**。

接着是客户端对911751号包的ack消息，说明序号是18428124之前的包已经全部接收到。然后接受到了911754号这个来自服务端的包，并且这包的开始seq序号是18433964，而最近一次来自服务端的911752号包的next seq是18432504，**说明在18432504 和18433964之间，也存在服务端的丢包情况**，所以911754号包也被标记为了Tcp Previous segment not captured。

接下来就是服务端重传包，客户端继续回应Ack的过程，但是这个过程直到914025号时就停止了，因为整个读取服务端响应的过程从开始读 到当前时间已经达到了应用层设置的500ms，所以在应用层直接就关闭掉了这个链接了。

看到现在，我有了充足的理由相信，是云服务提供商那边的问题，中间由于网络丢包的原因，且延迟较大导致了redis的读超时。拿着这些证据也说服了他们，并最终圆满解决。

#### 提工单，云服务商的排查支持

![image.png](https://s2.loli.net/2023/03/07/6H8c5Ja7UCfbp2d.png)

圆满解决