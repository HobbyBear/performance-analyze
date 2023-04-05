# mysql invalid conn排查

## 问题背景
服务使用golang ，客户端库是go-mysql-driver ,系统**测试环境频繁**但是不总是报出invalid conn 错误，但实际拿sql执行时却是正常执行。


## 排查思路

### 原因分析
### 客户端使用了无效连接
由于连接无效，首先考虑客户端使用了过期连接。

mysql服务器端和客户端都能配置各自连接的最大生命周期。

如果客户端配置的连接最大生命周期大于服务端，并且客户端没有无效连接重连的机制，则会导致服务端的连接在过期以后，客户端使用已经过期的连接，从而引发invalid conn。

查询mysql 连接的最大空闲生命周期 s为单位
```shell
show global variables like 'wait_timeout';
```
![image.png](https://s2.loli.net/2023/03/07/ZVMGw4hLNgTWe2y.png)
客户端最大生命周期配置
```go
db.SetConnMaxLifetime(time.Duration(timeout) * time.Hour) //reconnect after 1 hour

```
mysql server的最大生命周期大于golang客户端的连接最大生命周期，所以排除此原因。

### 连接被莫名的原因关掉了，抓包分析
一时间陷入僵局，连接究竟是为何无效。直接抓包分析。
```shell
sudo tcpdump -i lo port 3306 -w conn.pcap
```
#### 专家系统发现异常包
将pcap文件用wireshark打开，用专家系统对抓到的包做一个整体的分析。
看到有8个异常的RST包，正常的连接开关是不会有RST产生的，着重分析下RST产生的原因。

![image.png](https://s2.loli.net/2023/03/07/OCtGT1ezBIExidF.png)
#### 发现问题
![image.png](https://s2.loli.net/2023/03/07/RdkHFIy4Z8rCgpO.png)看到具体发生异常的地方，mysql服务器在回复客户端一个ACK消息后，客户端等了10s对mysql服务器发起了Request Quit(mysql 协议)关闭连接的命令。

具体看下异常前后。
选择菜单栏的Statistics->Flow Graph，就可以打开数据流图窗口。
![image.png](https://s2.loli.net/2023/03/07/MT7hVeYKDcqru2H.png)可以看到过程如下：
1，11:55:49.05 在客户端向mysql 发起 Request Execute Statement 执行sql的命令，
2，mysql 恢复Ack
3, 但是mysql并没有把执行结果返回给客户端，所以客户端等待了10s后发起关闭连接的命令。
4，mysql 在11:55:59.27才返回了执行结果。
5,但是这个时候客户端已经把连接关闭了，对已经关闭的连接发送数据触发了RST信号，所以客户端回应服务器RST

看到这里，已经可以很明确的任务是服务器执行sql超时了，或者说是服务器返回结果时超时了。

### 回到客户端代码确认报错原因
在进一步，是不是超时导致的报警呢？
可以看到在读取缓冲区数据时，有可能报出ErrInvalidConn
```go
// read packet header
		data, err := mc.buf.readNext(4)
		if err != nil {
			if cerr := mc.canceled.Value(); cerr != nil {
				return nil, cerr
			}
			errLog.Print(err)
			mc.Close()
			return nil, ErrInvalidConn
		}
```
继续查看mc.buf.readNext，发现超时代码
```shell
if b.timeout > 0 {
			if err := b.nc.SetReadDeadline(time.Now().Add(b.timeout)); err != nil {
				return err
			}
		}
```

### mysql受到什么因素导致sql过长
因为是数据库，首先想到了磁盘，再次回到top，iostat ,iotop分析。
此次发现异常。
![image.png](https://s2.loli.net/2023/03/07/ZzY4CWsxVnk3OXb.png)发现磁盘io使用率偶尔会飙高，所以mysql执行速率受到了影响。

### 解决问题
占用磁盘使用率的几个项目主要是视频转码项目，转码的时间不是固定的，所以当转码的时候，在同一台主机上的mysql受到了影响，引发了超时，导致应用层报出invalid conn 错误。

经过确认，测试环境可暂时关闭转码功能，关闭后，再也没有报警出现。

正常情况下，当抓包看到mysql返回超时是，应该立即就去看系统io等硬件指标了，这里由于是测试环境，所以我还是去看了go-mysql-client 库的代码，进一步确认。

## 反思
测试环境监控体系不够完善，没有监控界面，不然应该一眼能看出磁盘io异常。

其次是回归到应用层看代码的时机较晚，还是被invalid conn迷惑了，其实如果go-mysql库报错为超时错误可能会更符合这个场景。

其实从一开始，就用了top和iostat 去分析系统磁盘占用情况，但是由于那段时间内指标并无异常，加上报错信息是invalid conn 导致我首先把问题定位到是网络连接上出的问题，并抓包分析。
看到这种偶发的错误，应该持续性的观察系统指标，这次只是看了十几秒钟就转而去分析其他因素了，实属不该。

### 测试环境开启慢日志

```shell
show variables like 'slow_query%';
```
![image.png](https://s2.loli.net/2023/03/07/WqzoEQpnjvxmg4R.png)
开启慢日志
```shell
set global slow_query_log=1 
```