# 一次goroutine 泄漏排查案例

## 背景
这是一个比较经典的golang协程泄漏案例。

背景是这样，今天看到监控大盘数据发现协程的数量监控很奇怪。呈现上升趋势，然后骤降。虽然对协程数量做了报警机制，但是协程数量还是没有达到报警阈值，所以没有报警产生。

![image.png](https://s2.loli.net/2023/03/07/pWR8FOqnde5xGSz.png)

不过有经验的开发应该应该能一眼看出，这个肯定是协程泄漏了，因为协程数量一直在上涨，没有下降趋势，，中间下降的曲线其实是服务器重启造成的。

## pprof分析
为了直接确认是哪里导致的协程泄漏，用golang的pprof工具去对协程数量比较多的堆栈进行排查，关于golang pprof的使用以及统计原理可以看我的这个系列[golang pprof 的使用](https://github.com/HobbyBear/performance-analyze/tree/main/golang%2520pprof%2520tools)。

以下是采样到的goroutine的profile文件。
![image.png](https://s2.loli.net/2023/03/07/9K4JhmyxXv5poLl.png)

可以发现主要是transport.go这个文件里产生的协程没有被释放，transport.go这个文件是golang里用于发起http请求的文件，并且定位到了具体的协程泄漏代码位置 是writeloop 和readloop 函数。

> 熟悉golang的同学应该能立马想到，协程没有释放的原因极大可能是请求的响应体没有关闭。这也算是golang里面的一个坑了。

在分析之前，还是先说下结论，**resp.Body在被完整读取时，即使不显示的进行关闭也不会造成协程泄漏，只有读取部分resp.Body时，不显示关闭才会引发协程泄漏问题**。

现在我们还是 具体分析下为啥resp body不关闭，会造成协程泄漏。


## 请求发送与接收流程

我们先来看看golang里面是如何发送以及接收http请求的。下面这张图完整的展示了一个请求被发送以及其响应被接收的过程，我们基于它然后结合代码分析下。

![image.png](https://s2.loli.net/2023/04/10/IGPgtKClxOnMvw8.png)

如图所示，在我们用http.Get 方法发送请求时，底层追踪下去，会调用到roundtrip 函数进行请求的发送与响应的接收。roundtrip位置在源码的位置如下，代码基于golang1.17版本进行分析，
```go
// src/net/http/transport.go:2528
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) 
```
在代码里，用persistConn这个结构体代表了一个http连接，这个连接可以从连接池中获取，也可以被新建。
```go
// src/net/http/transport.go:1869 reqch 和writech 都是连接的属性
type persistConn struct {
.....
reqch     chan requestAndChan // written by roundTrip; read by readLoop
writech   chan writeRequest   // written by roundTrip; read by writeLoop
...
}
```
在roundtrip函数中，会往persistConn 的writech和reqch两个chan 通道内发送数据。代码如下:
```go
// src/net/http/transport.go:2528
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    ....
    // src/net/http/transport.go:2594
    	pc.writech <- writeRequest{req, writeErrCh, continueCh}
    ...
    // src/net/http/transport.go:2598
    pc.reqch <- requestAndChan{
		req:        req.Request,
		cancelKey:  req.cancelKey,
		ch:         resc,
		addedGzip:  requestedGzip,
		continueCh: continueCh,
		callerGone: gone,
	}
}
```
#### 请求发送过程
writech 通道和请求的发送有关，通道里的请求真正发送到网卡则是由persistConn的**writeloop**方法完成的。

persistConn的writeloop 方法是连接被dialConn方法创建的时候，就会用一个协程去调度执行的方法。代码如下：
```go
// src/net/http/transport.go:1560
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {
    .... 省略了部分代码
   // src/net/http/transport.go:1747 
   go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}
```

在pconn.writeLoop里，会不断的轮询persistConn的writech通道里的消息，然后通过wr.req.Request.write发送到互联网中。

```go
// src/net/http/transport.go:2383 
func (pc *persistConn) writeLoop() {
	defer close(pc.writeLoopDone)
	for {
		select {
		case wr := <-pc.writech:
			startBytesWritten := pc.nwrite
			err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
			.... 省略部分代码
}
```

知道请求时如何发送出去的了，那么连接persistConn是如何接收请求的响应呢？

#### 响应接收的流程

我们再回到roundtrip函数逻辑里，除了赋值persistConn的writech属性值，roundtrip函数还会为persistConn的reqch属性赋值，persistConn在被创建时，同样会启动一个协程去调度执行一个叫做readloop的方法。代码其实已经在上面展示过了，不过为了方便看，我在此处再列举一次，
```go
// src/net/http/transport.go:2528
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    .... 省略部分代码
    // src/net/http/transport.go:2598
    pc.reqch <- requestAndChan{
		req:        req.Request,
		cancelKey:  req.cancelKey,
		ch:         resc,
		addedGzip:  requestedGzip,
		continueCh: continueCh,
		callerGone: gone,
	}
	    .... 省略部分代码
}

// src/net/http/transport.go:1560
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {
    .... 省略了部分代码
   // src/net/http/transport.go:1747 
   go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}
```

readloop 方法会 读取persistConn 读缓冲区中的数据，读到后就将响应信息放到reqch通道里，最终reqch通道里的响应信息就能被roundtrip函数获取到然后返回给应用层代码了。

readloop读取缓冲区数据大致流程如下:
```go
// src/net/http/transport.go:2052
func (pc *persistConn) readLoop() {
    .... 省略部分代码
    for alive {
		
		... 省略部分代码
		
		rc := <-pc.reqch
		trace := httptrace.ContextClientTrace(rc.req.Context())

		var resp *Response
		if err == nil {
		   // 读取响应
			resp, err = pc.readResponse(rc, trace)
		} else {
			err = transportReadFromServerError{err}
			closeErr = err
		}

		...... 
		
		waitForBodyRead := make(chan bool, 2)
		body := &bodyEOFSignal{
			body: resp.Body,
			earlyCloseFn: func() error {
				waitForBodyRead <- false
				<-eofc // will be closed by deferred call at the end of the function
				return nil

			},
			fn: func(err error) error {
				isEOF := err == io.EOF
				waitForBodyRead <- isEOF
				if isEOF {
					<-eofc // see comment above eofc declaration
				} else if err != nil {
					if cerr := pc.canceled(); cerr != nil {
						return cerr
					}
				}
				return err
			},
		}

		resp.Body = body
		
		.......
       select {
      //  rc 是pc.reqch的引用，这里将响应结果传递给了这个通道
		case rc.ch <- responseAndError{res: resp}:
		case <-rc.callerGone:
			return
		} 
      // 阻塞等待响应信息被读取完毕或者应用层关闭resp.Body 
		select {
		case bodyEOF := <-waitForBodyRead:
			replaced := pc.t.replaceReqCanceler(rc.cancelKey, nil) 			alive = alive &&
				bodyEOF &&
				!pc.sawEOF &&
				pc.wroteRequest() &&
				replaced && tryPutIdleConn(trace)
			if bodyEOF {
				eofc <- struct{}{}
			}
		case <-rc.req.Cancel:
			alive = false
			pc.t.CancelRequest(rc.req)
		case <-rc.req.Context().Done():
			alive = false
			pc.t.cancelRequest(rc.cancelKey, rc.req.Context().Err())
		case <-pc.closech:
			alive = false
		}
	}
}
```

readloop 通过**pc.readResponse** 读取一次http响应后，会将响应体发送到pc.reqch ，roundtrip函数阻塞等待pc.reqch里有数据到达后，则将pc.reqch里的响应体取出来返回给应用层代码。

注意readloop函数在读取一次响应后，会阻塞等待响应体被读取完毕，或者响应体被Close掉后，才会将persistConn重新放回连接池，然后等待读下一个http的响应体。 应用层会调用resp.Body的Close方法,从readloop源码可以看出，resp.body实际是个bodyEOFSignal类型，bodyEOFSignal的Close方法如下:

```go
func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {
	
	....省略部分代码 
	
	n, err = es.body.Read(p)
	if err != nil {
		es.mu.Lock()
		defer es.mu.Unlock()
		if es.rerr == nil {
			es.rerr = err
		}
		err = es.condfn(err)
	}
	return
}

func (es *bodyEOFSignal) Close() error {
	es.mu.Lock()
	defer es.mu.Unlock()
	if es.closed {
		return nil
	}
	es.closed = true
	if es.earlyCloseFn != nil && es.rerr != io.EOF {
		return es.earlyCloseFn()
	}
	err := es.body.Close()
	return es.condfn(err)
}

// caller must hold es.mu.
func (es *bodyEOFSignal) condfn(err error) error {
	if es.fn == nil {
		return err
	}
	err = es.fn(err)
	es.fn = nil
	return err
}
```

调用**bodyEOFSignal.Close**方法最终会调到bodyEOFSignal的fn方法或者earlyCloseFn方法，earlyCloseFn在Close响应体的时候，发现响应体还没有被完全读取时会被调用。
调用**bodyEOFSignal.Read**方法时，当read读取完毕后err将会是 io.EOF，此时err不为空将会调用condfn 方法对fn方法进行调用。

fn,earlyCloseFn函数是在哪里声明的呢？还记得readloop源码里bodyEOFSignal的声明吗，我这里再展示一下上述的源码部分:
```go
// src/net/http/transport.go:2166
body := &bodyEOFSignal{
			body: resp.Body,
			earlyCloseFn: func() error {
				waitForBodyRead <- false
				<-eofc // will be closed by deferred call at the end of the function
				return nil

			},
			fn: func(err error) error {
				isEOF := err == io.EOF
				waitForBodyRead <- isEOF
				if isEOF {
					<-eofc // see comment above eofc declaration
				} else if err != nil {
					if cerr := pc.canceled(); cerr != nil {
						return cerr
					}
				}
				return err
			},
		}
```
声明响应体body的时候就定义好了者两个函数，这两个函数都是往waitForBodyRead通道发送消息，readloop会阻塞等待waitForBodyRead的消息到达。**消息到达后说明resp.Body 被读取完毕或者主动关闭了，然后调用tryPutIdleConn将连接重新放回连接池中** 完整的代码还是在上述readloop的源码片段里，我这里只展示下readloop部分代码。
```go
// src/net/http/transport.go:2207
select {
		case bodyEOF := <-waitForBodyRead:
			replaced := pc.t.replaceReqCanceler(rc.cancelKey, nil) // before pc might return to idle pool
			alive = alive &&
				bodyEOF &&
				!pc.sawEOF &&
				pc.wroteRequest() &&
				// tryPutIdeConn 将连接重新放入连接池
				replaced && tryPutIdleConn(trace)
			if bodyEOF {
				eofc <- struct{}{}
			}
```


现在再来看我们go协程泄漏的代码在那里，是在readloop和writelooop函数中，**泄漏的原因就在于读取响应体后没有将persistConn重新放回连接池里，执行的readloop函数的协程一直阻塞等待waitForBodyRead消息的到达，而后续的请求又新建了连接，从而新起了readloop协程，writeloop协程，同样也阻塞在这里，导致协程数量越来越多，从而有协程泄漏的现象**。

*一般情况下，我们都会完整的读取完resp.Body，所以即使不显示的关闭body，也不会有泄漏问题产生*，但我们的程序刚好有段逻辑需要只需要读取body的前10字节，代码如下:

```go
_, err = ioutil.ReadAll(io.LimitReader(resp.Body, 10))
	if err != nil && err != io.EOF {
		t.Fatal(err)
	}
```
读取完后也没有关闭resp.Body 并且类似的请求越来越多，导致我们的协程数量越来越多了。

修复这个bug也很简单，即对resp body关闭即可。
```shell
resp.body.Close()
```

## 反思

golang resp body 还是一定要记得关闭，不然就会引发协程泄漏问题，这次由于同事对此类问题没有过多重视，导致了这个问题，好在有监控大盘，及时发现，不然后果不堪设想。

