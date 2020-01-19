## 问题描述

表现一：组内使用golang开发的stable proxy程序跑着跑着就会反应变慢，甚至卡死无响应（程序中的http服务的接口调用超时失败，表现就是程序卡死无响应）。但是只要一重启就会变正常。

表现二：使用组内的golang网络框架tcp，收到一个protobuf的数据包，使用proto.Unmarshal(pack.Item.Data, request)的时候总是失败 - unexcept EOF

## 问题分析

### 问题可能一： 大量的CLOSE_WAIT状态导致服务不可用
针对表现一：stable proxy 反应变慢 - 这里的表现是http服务接口访问变慢，之前遇到这个问题是因为系统下面的 CLOSE_WAIT 状态的socket过多导致的。过多的CLOSE_WAIT状态的socket会导致服务的可用socket资源耗。当服务出现异常的时候，登录服务器去查看socket的状态，发现socket的状态试是正常的，CLOSE_WAIT的数量只有二三十个。CLOSE_WAIT状态是tcp的被动关闭方，在发送最后一个FIN包的一个正常状态。服务重启就会好-进程释放CLOSE_WAIT状态的socket自然重启就好了。

```
netstat -anp | grep CLOSE_WAIT
```

如果说是因为CLOSE_WAIT状态引起的，那么可以分析模拟一下：
proxy的tcp链接只存在于两个地方，一是和build的tcp链接，另一个就是和mysql数据库的链接。

什么时候build和mysql会主动断开链接呢？
1. build和mysql数据库进程退出
2. build和mysql的TCP链接超时

build的tcp请求确实设置了超时30s，但是并没有超时候断开的逻辑。
至于mysql的超时，因为XDB的db-proxy是作为服务端应该也是没有设置客户端超时断开的逻辑

下面是网络上的一个实例-几百个CLOSE_WAIT状态的socket导致了服务不可用，只能重启进程。进程重启那么就会释放进程资源-socket，所以重启对于这种情况是有效的。但是为什么只有几百个CLOSE_WAIT的socket就会导致不可用呢？一台机器的文件描述符不是可以开到65535嘛？

https://mp.weixin.qq.com/s?__biz=MzI4MjA4ODU0Ng==&mid=402163560&idx=1&sn=5269044286ce1d142cca1b5fed3efab1&3rd=MzA3MDU4NTYzMw==&scene=6#rd

关于为什么不能开到65535，是和网上这个例子有关系的，他的tcp server，使用单进程的epoll模型，通过注册事件回调来进行读写的处理，回调函数通过协程调度，并没有放在一个子线程中执行，但是handle的过程中又同步请求了第三方服务，导致协程的yeild调度函数失效，这样一个严重的问题就是回调函数和loop函数在同一个线程里面，，回调函数直接限制了loop函数的处理，loop函数的链接请求队列大小只有128，也就导致客户端请求超时-断开,服务端CLOSE_WAIT状态又不会主动消失，不能在这个端口创建新的链接引起服务停止。CLOSE_WAIT一般会持续两个小时。

### 问题可能二：大量的TIME_WAIT状态的socket导致服务不可用
虽然不是上面的CLOSE_WAIT的问题但是发现了很多TIME_WAIT的状态，大约有几百个

```
netstat -anp | grep TIME_WAIT
```

TIME_WAIT的状态是tcp主动断开的一方的状态，在等待被动关闭的一方发送FIN包的一种正常状态。一般这种状态会持续2min。处于TIME_WAIT状态的socket如果数量过多的话，会导致新来的客户端链接因为端口资源不足而导致连接失败。那我们使用SO_REUSEADDR参数不行吗，复用TIME_WAIT状态的socket。这样也是不行的因为客户端的这写socket都是操作系统随机分配的。在高并发的大量短链接系统中这种情况会出现。但是分析了一下proxy的代码发现并不是这个原因，因为和proxy只提供三个服务，一个tcp服务和两个http服务，而且http服务的客户端都是使用http1.1的keepalive来进行链接的，并不短链接，而且并发量也不高。那这些timewait是哪里产生的呢？是bns的查询接口产生的。

### 问题可能三： 系统资源耗尽导致卡顿或者退出
是不是CPU打满了，还是有内存泄露导致服务跑不动了？
在系统出问题的时候我登录了机器查看了进程的资源，并没有任何的异常，CPU和内存非常的健康。top。本来就是一个纯io的服务，并不存在CPU打满的情况。
那么是socket资源用完了？ 下面列几个查看进程socket状态的命令：
查看进程的端口占用情况
```
ll /proc/pid/fd | wc -l
lsof -p pid
strace -p pid >> log.log 2>&1
ss -s
netstat -anp | grep
```

通过查看该进程的socket状态发现有一些socket的状态是can't identify protocol，这是一种很不正常的socket状态。既不是CLOSE，ESTABLISHED，TIME_WAIT，CLOSE_WAIT状态。那么这些状态是什么状态呢？为什么会出现这些状态呢？

这些状态的出现标识，socket有泄露，不能被lsof命令所示别。比如只执行了bind但是没有connect或者listen操作的socket，还有没有正常close的socket。还有socket 为什么会出现这些状态的socket呢？

具体怎么去查询进程的socket泄露可以使用 strace 命令进行分析，可以参考下面的内容：

https://blog.csdn.net/u011493488/article/details/83212967

### 问题可能四：http服务就是卡，处理慢
通过对某一个http接口进行压测，当我们感知到proxy卡死的时候，就是网页打开卡直到打不开，一直在转圈，随便测试一个本来处理速度应该很快的接口也是非常慢，直到客户端超时。好，那么现在怀疑http server出问题了，发现代码中使用的是golang原生的http server,是不是用法不对呢？

程序中使用的是golang原生的http服务并且没有设置读写超时。http服务就算没设置读写超时，客户端也会设置读超时。不会导致程序出现反应吃顿，出现假死的状况。排除，当然这里还是要去设置一下服务端的读写超时


### 问题可能五：数据库的设置有问题

golang的mysql是数据库连接池，proxy只是设置了连接池的大小和空闲链接数量的最大值，并没有设置超时时间。那么这样的话就会出现，mysql服务端设置了超时时间，但是调用方的超时间大于mysql端的时间，mysql端已经断开链接了，但是适用房的连接池这一端发现还有空闲的链接在链接池子里面躺着，就会去用这个已经被mysql服务端断开的链接，那么这样会有什么后果呢？proxy认为这个链接还能用于是会使用这个链接进行发送，但是这个时候会报错，而不是导致程序hang死
```
[mysql] 2020/01/19 17:08:15 packets.go:36: unexpected EOF
error:  invalid connection
```
```
设置本地的sql，进行猜想的验证 - 设置超时为 60s interactive_timeout 是终端交互的超时时间 
set wait_timeout=60;
set interactive_timeout=60;
set global wait_timeout=60;
set global interactive_timeout=60;

show global variables like '%timeout%';
show variables like '%timeout%';
```

这里啊还是要设置mysql的connect timeout 的。

### 可能问题六：数据库的连接池有泄露

最可能的事情就是，数据库的连接池中有链接泄露。如果连接池中的资源耗尽，那么会导致在进行查询的时候，阻塞在 QUERY 这个函数上面。至于上面说的 http 的服务接口卡住，因为http的服务接口都会去请求数据库。这样所有的现象就都对上了，一开始没网这上面想是因为，当程序卡死的时候我去查数据库的链接，发现系统中和数据库的链接并没有那么多，200的最大连接数，可是使用命令 netstat -anp | grep 5668 发现连接数只有几个，所以我判断这里不是发生了泄露。因为线下写的泄露的例子都是连接数上涨的，因为mysql的wait_timeout时间是8小时，所以这里非常奇怪。

最后没办法我去修改了golang关于sql这块的源代码，在每次query的时候我都打印出连接池的状态，最后发现跑了两个小时之后，连接数果然有泄露，是43。这里我大胆判断应该是有些逻辑跑到了-数据库的连接数只增没减，但是链接被正常复用了.现在也没搞清楚为什么有这种现象。。。。

修复了未能正产关闭的 rows，果然卡死的现象再也没有发生。奇怪的是，numopen增加，但是netstat -anp 查看数据库相应端口的链接却没有增加或者异常，真奇怪。。

### 小插曲

在压测的过程中，两台机器上面的proxy服务在压测时候的表现不一样。我是针对一个tcp服务接口进行压测的。在一台服务器上面，压测的频率上来之后,打印日志的时候会有延迟，还在打印1分钟之前的日志。一开始我以为是硬盘的问题，或者socket缓冲区的问题，后来和另一台机器对比之后发现并没不是这个问题。

后来使用 iftop 这个工具，查看了一下这台机器的网络出口流量，对比那台好的机器他的王楼出口流量要小的多，也就是网络状况不好。所以存在接收延迟的现象。

## 问题描述
组内的gonet网络框架(基于protobuf协议-并不是使用protobuf库写的RPC框架，只是一个异常简单的支持protobuf的极简tcp网络框架而已)。

发送端 ==》 proto.marshal(proto.Message) ==> 接收端 ==》proto.unmarshal(readbuffer) 

接收端 unmarshal的时候就会出现 EOF，说明数据不对了

## 问题分析

接收端数据出错，首先我怀疑的就是数据在发送的时候出了问题，但是这个问题是偶现的，通过tcpdump 抓包分析，出错的包和正常的包是没有任何区别的。排除发送的时候数据包序列化出错的可能。

那么接收端在数据处理的时候肯定出了问题，是不是处理接受的函数不是一个可重入函数，并且出现了线程安全问题或者，自身线程被中断处理，操作了未保护的数据。看了下这个机器简单的网络框架，并不存在这个问题。

经过代码的仔细排查发现，每次headleread的时候会向chan中发送一个接受到的数据包。这个数据包中包含一个slice，愚蠢的事情发生了，传入这五个slice的时候不是 make + copy 的，而是全部复用socket接收buf的这个slice，slice不管怎么处理和切割都是共享底层数组的，所以，返回的这个 slice 就相当于一个全局变量，prtoxy业务每一个数据请求新建一个 go 程来处理，这样就造成了竞态
```
func (conn *Connection) HandleRead(PackageConnCh chan *PackageConn, exitCh chan *Connection){
    buf := make([]byte, 10240)
    defer conn.Close()
	for {
		n, err := conn.Conn.Read(buf)
		if err != nil || n <= 0 {
			// log.Printf("[ERROR] connection[%s] disconnect [error:%s]", conn.RemoteAddr, err.Error())
			exitCh <- conn
			return
		}
        // readbuf 是一个 切片缓冲器  bytes.Buffer, 它构建的时候就是共享了 buf的内存
        conn.readBuf.Write(buf[:n])
		for conn.readBuf.Len() >= 8 {
            // 这个函数在处理的时候并没有新建slice给p，导致所有返回的p的Data结构都共享 buf 的底层数组
			p, err := conn.ParsePackage()
			if err != nil {
                buf = nil
				log.Printf("[ERROR] illegal msg(%s) from buffer(%s) of %s, exit\n", err.Error(), buf, conn.RemoteAddr)
				exitCh <- conn
				return
			}
			// package is not complete
			if p == nil {
                buf = nil
				//log.Printf("[DEBUG] Package from [%s] is incomplete, continue to recv", conn.RemoteAddr)
				break
			}
            buf = nil
			pconn := new(PackageConn)
			pconn.Item = p
			pconn.Conn = conn
			PackageConnCh <- pconn
		}
        buf = make([]byte, 4096)
	}
}
```