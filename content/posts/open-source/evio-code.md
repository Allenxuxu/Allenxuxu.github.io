---
title: "Golang 高性能网络库 evio 源码解析"
date: 2019-08-06T11:22:38+08:00
draft: false
tags: ["Go", "Open Source"]
---

> 阅读前提：了解 epoll

[evio](https://github.com/tidwall/evio) 是一个基于事件驱动的网络框架，它非常轻量而且相比 Go net 标准库更快。其底层使用epoll 和 kqueue 系统调度实现。 

![echo.png](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/echo.png)

---

## 原理

evio 是 Reactor 模式的简单实现。Reactor 本质就是“non-blocking IO + IO multiplexing”，通过非阻塞IO+ IO 多路复用来处理并发。程序运行一个或者多个事件循环，通过在事件循环中注册回调的方式实现业务逻辑。

evio 将所有文件描述符设为非阻塞，并注册到事件循环（ epoll / kqueue ）中。相较于传统的 per thread per connection 的处理方法，线程使用更少，线程资源利用率更高。

evio 需要在服务启动前，注册回调函数，当事件循环中有事件到来时，会调用回调函数处理。

## 使用示例

先从一个简单的 echo server 的例子来了解 evio 。

```go
package main

import (
	"flag"
	"fmt"
	"log"
	"strings"

	"github.com/tidwall/evio"
)

func main() {
	var port int
	var loops int
	var udp bool
	var trace bool
	var reuseport bool
	var stdlib bool

	flag.IntVar(&port, "port", 5000, "server port")
	flag.BoolVar(&udp, "udp", false, "listen on udp")
	flag.BoolVar(&reuseport, "reuseport", false, "reuseport (SO_REUSEPORT)")
	flag.BoolVar(&trace, "trace", false, "print packets to console")
	flag.IntVar(&loops, "loops", 0, "num loops")
	flag.BoolVar(&stdlib, "stdlib", false, "use stdlib")
	flag.Parse()

	var events evio.Events
	events.NumLoops = loops
	events.Serving = func(srv evio.Server) (action evio.Action) {
		log.Printf("echo server started on port %d (loops: %d)", port, srv.NumLoops)
		if reuseport {
			log.Printf("reuseport")
		}
		if stdlib {
			log.Printf("stdlib")
		}
		return
	}
	events.Data = func(c evio.Conn, in []byte) (out []byte, action evio.Action) {
		if trace {
			log.Printf("%s", strings.TrimSpace(string(in)))
		}
		out = in
		return
	}
	scheme := "tcp"
	if udp {
		scheme = "udp"
	}
	if stdlib {
		scheme += "-net"
	}
	log.Fatal(evio.Serve(events, fmt.Sprintf("%s://:%d?reuseport=%t", scheme, port, reuseport)))
}
```

上面的例子主要就是注册了两个回调函数： events.Serving 和 events.Data 。

当 server 启动时，会来执行注册的 events.Serving 回调函数；
当有数据到来时，执行  events.Data 回调函数。

程序最后调用 evio.Serve 方法开启事件循环，程序在此处不断循环检测是否有事件发生并处理（有数据到来，有数据要发送...)。

evio 都是通过回调函数来执行业务逻辑的。 当客户端有数据发送过来时，调用用户注册的 events.Data 函数。

需要发送数据给客户端时，只可以通过注册的回调函数的返回值来返回，evio 框架来负责发送（有bug）。

回调函数的返回值主要有两个 `out []byte, action evio.Action` , out 就是需要发送给客户端的， Action 就是返回一些状态，用来关闭连接，或者服务器退出啥的操作。主要状态如下：

```go
const (
	// None indicates that no action should occur following an event.
	None Action = iota
	// Detach detaches a connection. Not available for UDP connections.
	Detach
	// Close closes the connection.
	Close
	// Shutdown shutdowns the server.
	Shutdown
)
```

## evio 的事件循环

### evio.Serve

我们先来看下  evio.Serve  方法的实现

```go
func Serve(events Events, addr ...string) error {

	var lns []*listener
	defer func() {
				// 这个函数如果推出，需要关闭所有 listener
		for _, ln := range lns {
			ln.close()
		}
	}()
	var stdlib bool
				// 可以选择使用 stdlib（stdlib 主要是为了支持 非 *unix 平台）
	for _, addr := range addr {
	// 生成 listener
		var ln listener
		var stdlibt bool
		ln.network, ln.addr, ln.opts, stdlibt = parseAddr(addr)
		if stdlibt {
			stdlib = true
		}
		if ln.network == "unix" {
			os.RemoveAll(ln.addr)
		}
		var err error
		if ln.network == "udp" {
			if ln.opts.reusePort {
				ln.pconn, err = reuseportListenPacket(ln.network, ln.addr)
			} else {
				ln.pconn, err = net.ListenPacket(ln.network, ln.addr)
			}
		} else {
			if ln.opts.reusePort {
				ln.ln, err = reuseportListen(ln.network, ln.addr)
			} else {
				ln.ln, err = net.Listen(ln.network, ln.addr)
			}
		}
		if err != nil {
			return err
		}
		if ln.pconn != nil {
			ln.lnaddr = ln.pconn.LocalAddr()
		} else {
			ln.lnaddr = ln.ln.Addr()
		}
		if !stdlib {
			if err := ln.system(); err != nil {
				return err
			}
		}
		lns = append(lns, &ln)
	}
	if stdlib {
		return stdserve(events, lns)
		// 使用 std net 库 启动server
	}
	return serve(events, lns)
				// 使用 epoll or kqueue 启动server
}
```

从 Serve 函数签名中可以看出 evio 是支持绑定多地址监听的

```go
func  Serve(events Events, addr ...string)  error
```

使用方式如下：

```go
evio.Serve(events, "tcp://localhost:5000", "tcp://192.168.0.10:5001");
```

现在我们看看 evio 的核心部分: serve(events, lns)
，这里会启动 evio 的 sever 。

```go
func serve(events Events, listeners []*listener) error {
	numLoops := events.NumLoops				// 确定启动的事件循环数量
	if numLoops <= 0 {
		if numLoops == 0 {
			numLoops = 1
		} else {
			numLoops = runtime.NumCPU()
		}
	}

	s := &server{}
	s.events = events
	s.lns = listeners
	s.cond = sync.NewCond(&sync.Mutex{})
	s.balance = events.LoadBalance
	s.tch = make(chan time.Duration)

	//println("-- server starting")
	if s.events.Serving != nil {					// 如果注册了回调函数，就执行
		var svr Server
		svr.NumLoops = numLoops
		svr.Addrs = make([]net.Addr, len(listeners))
		for i, ln := range listeners {
			svr.Addrs[i] = ln.lnaddr
		}
		action := s.events.Serving(svr)
		switch action {
		case None:
		case Shutdown:
			return nil
		}
	}

	defer func() {									// server 退出后的清理工作
		// wait on a signal for shutdown
		s.waitForShutdown()

		// notify all loops to close by closing all listeners
		for _, l := range s.loops {
			l.poll.Trigger(errClosing)
		}

		// wait on all loops to complete reading events
		s.wg.Wait()

		// close loops and all outstanding connections
		for _, l := range s.loops {
			for _, c := range l.fdconns {
				loopCloseConn(s, l, c, nil)
			}
			l.poll.Close()
		}
		//println("-- server stopped")
	}()

	// create loops locally and bind the listeners.
	for i := 0; i < numLoops; i++ {
		l := &loop{
			idx:     i,
			poll:    internal.OpenPoll(),
			packet:  make([]byte, 0xFFFF), 	// event loop 的 read 缓冲区
			fdconns: make(map[int]*conn),
		}
		for _, ln := range listeners {
			l.poll.AddRead(ln.fd)			// 将 fd 注册到 epoll 中并监听可读事件
		}
		s.loops = append(s.loops, l)
	}
	// start loops in background
	s.wg.Add(len(s.loops))
	for _, l := range s.loops { 				// 启动所有的 event loop
		go loopRun(s, l)
	}
	return nil
}
```

serve 主要做这些事：

1. 根据配置启动指定数量的 event loop，如果传入配置的 loop 数量为 0 则设置启动一个事件循环，如果传入配置小于 0 则设置为运行平台的CPU核心数量
2. 如果设置了回调函数 events.Serving ，运行它
3. 按照指定 event loop 数量，创建 epoll 句柄生成 loop ，并将所有的 listener 注册到 epoll 监听可读事件（有客户端连接）
4. 启动所有事件循环(一个事件循环一个 goroutine)

需要注意的是，evio 将所有的 listener 的 fd 在每一个事件循环的 epoll 中都注册了。也就是说，如果有三个事件循环，一个 listener ，那么这个 listener 的 fd 会注册到三个 epoll 中。这就会出现 epoll 的惊群现象，感兴趣的可以自己搜索了解下。

evio 当一个新连接到来时，所有的事件循环都会唤醒，但是最终只有一个线程可以accept调用返回成功，其他线程（协程）的accept函数调用返回EAGAIN错误 。

### loopRun

下面我们看看 loopRun 的内部实现

```go
func loopRun(s *server, l *loop) {
	defer func() {
		//fmt.Println("-- loop stopped --", l.idx)
		s.signalShutdown()
		s.wg.Done()
	}()

	if l.idx == 0 && s.events.Tick != nil {
		go loopTicker(s, l)
	}

	//fmt.Println("-- loop started --", l.idx)
	l.poll.Wait(func(fd int, note interface{}) error {
		if fd == 0 {
			return loopNote(s, l, note)
		}
		c := l.fdconns[fd]
		switch {
		case c == nil:
			return loopAccept(s, l, fd)
		case !c.opened:
			return loopOpened(s, l, c)
		case len(c.out) > 0:
			return loopWrite(s, l, c)
		case c.action != None:
			return loopAction(s, l, c)
		default:
			return loopRead(s, l, c)
		}
	})
}
```

l.poll.Wait 传入一个回调函数作为参数，当 epoll 收到事件通知时，会执行这个回调函数。

在这个函数中接受客户端连接，读取客户端数据，调用客户回调函数处理业务逻辑...

我们先来看下 poll.Wait 的内部实现，再看看 loopAccept，loopOpened，loopWrite 等函数。
loopRun 方法中最重要的就是 poll.Wait ，我们看看 Linux 下 epoll 的实现

```go
func (p *Poll) Wait(iter func(fd int, note interface{}) error) error {
	events := make([]syscall.EpollEvent, 64)
	for {
		n, err := syscall.EpollWait(p.fd, events, -1)
		if err != nil && err != syscall.EINTR {
			return err
		}
		if err := p.notes.ForEach(func(note interface{}) error {
			return iter(0, note)
		}); err != nil {
			return err
		}
		for i := 0; i < n; i++ {
			if fd := int(events[i].Fd); fd != p.wfd {
				if err := iter(fd, nil); err != nil {
					return err
				}
			} else {

			}
		}
	}
}
```

这个函数中是一个死循环，程序会阻塞在此处等待 epoll 的”通知“，然后处理就绪的 fd （读取/发送数据、执行用户注册的回调函数）。

当有 fd 就绪的时候，syscall.EpollWait 函数返回，并且将就绪的 fd 通过 events 传出，返回值 n 为就绪 fd 的个数。

然后循环逐个遍历就绪的 fd，调用回调函数处理。

```go
for i := 0; i < n; i++ {
	if fd := int(events[i].Fd); fd != p.wfd {
		if err := iter(fd, nil); err != nil {
			return err
		}
	} else {

		}
}
```

## evio 的事件处理

```go
l.poll.Wait(func(fd int, note interface{}) error {
		if fd == 0 {
			return loopNote(s, l, note)
		}
		c := l.fdconns[fd]
		switch {
		case c == nil:
			return loopAccept(s, l, fd)
		case !c.opened:
			return loopOpened(s, l, c)
		case len(c.out) > 0:
			return loopWrite(s, l, c)
		case c.action != None:
			return loopAction(s, l, c)
		default:
			return loopRead(s, l, c)
		}
})
```

当 epoll 检测到有就绪的 fd 时，会逐个调用上面的回调函数，evio 的主要逻辑也在这里。

当 fd == 0 时，会执行 loopNote 函数。loopNote 主要是用来处理一些非 fd 就绪的事件，比如定时任务、强制退出等。当然，我们都知道 fd 为 0 是标准输入，所以此处并不是真的去处理 fd 为 0 的文件描述符（注册到 epoll 的文件描述 >= 3）。作者知道 epoll 返回的就绪 fd 中不会有为 0 的情况，所以此处 fd 为 0，是作者调用时传入，用来表示一种特殊的唤醒场景。

```go
func (p *Poll) Wait(iter func(fd int, note interface{}) error) error {
...
	p.changes = p.changes[:0]
		if err := p.notes.ForEach(func(note interface{}) error {
			return iter(0, note)
...
```

我们跳到调用它的地方，可以看到只有在  p.notes.ForEach 这个函数中注册的回调函数中才会传入 fd 为 0 来执行 iter 回调函数。 

` notes noteQueue `

noteQueue 的实现在 internal 目录中的 notequeue.go , 是一个无锁队列。我们不详细分析，只看下 ForEach 这个方法：

```go
func (q *noteQueue) ForEach(iter func(note interface{}) error) error {
	q.mu.Lock()
	if len(q.notes) == 0 {
		q.mu.Unlock()
		return nil
	}
	notes := q.notes
	q.notes = nil
	q.mu.Unlock()
	for _, note := range notes {
		if err := iter(note); err != nil {   // 执行回调函数
			return err
		}
	}
	return nil
}
```

当队列中有数据时， 会执行回调函数，即

```go
func(note interface{}) error {
	return iter(0, note)
}
```

从上面的分析中可以我们已经知道为什么会有 fd 为 0 ，下面我们看下 loopNote 做什么。

### loopNote

```go
func loopNote(s *server, l *loop, note interface{}) error {
	var err error
	switch v := note.(type) {
	case time.Duration:
		delay, action := s.events.Tick()
		switch action {
		case None:
		case Shutdown:
			err = errClosing
		}
		s.tch <- delay
	case error: // shutdown
		err = v
	case *conn:
		// Wake called for connection
		if l.fdconns[v.fd] != v {
			return nil // ignore stale wakes
		}
		return loopWake(s, l, v)
	}
	return err
}
```

传入的 note 是 interface{} ，首先对 note 进行类型判断。

当 note 是 time.Duration 时，调用回调函数 events.Tick() ，这是 evio 提供的定时任务接口。

在 loopRun 函数中，如果设置了定时回调函数，会启动一个协程来来运行 loopTicker

```go
if l.idx == 0 && s.events.Tick != nil {
	go loopTicker(s, l)
}
```

loopTicker 实现如下，可以看出会定时去触发 l.poll.Trigger，并且传入 time.Duration(0)

```go
func loopTicker(s *server, l *loop) {
	for {
		if err := l.poll.Trigger(time.Duration(0)); err != nil {
			break
		}
		time.Sleep(<-s.tch)
	}
}
```

我们跳到 poll.Trigger 的 linux 下的实现，可以发现 evio 在此处 p.notes.Add(note) ，也就是 time.Duration(0)

```go
func (p *Poll) Trigger(note interface{}) error {
	p.notes.Add(note)
	_, err := syscall.Write(p.wfd, []byte{0, 0, 0, 0, 0, 0, 0, 1})
	return err
}
```

poll.Trigger 这个函数不仅仅是在 p.notes 里增加了一个 note，还唤醒了事件循环。

当 epoll 中注册 fd 都没有就绪事件时，线程会挂起，epoll 的 wait 方法会处于阻塞状态。evio 使用 
linux 提供的 eventfd 来实现事件循环的唤醒，也就是代码上中的 `syscall.Write(p.wfd, []byte{0, 0, 0, 0, 0, 0, 0, 1})` ,往 p.wfd 这个文件描述符中写入了 8 个字节的数据。

p.wfd 是一个 eventfd , 是 Poll 结构体的成员，在 OpenPoll 时赋值，即打开一个 eventfd 代码如下：

```go
type Poll struct {
	fd    int // epoll fd
	wfd   int // wake fd
	notes noteQueue
}

func OpenPoll() *Poll {
	l := new(Poll)
	p, err := syscall.EpollCreate1(0)
	if err != nil {
		panic(err)
	}
	l.fd = p
	r0, _, e0 := syscall.Syscall(syscall.SYS_EVENTFD2, 0, 0, 0)
	if e0 != 0 {
		syscall.Close(p)
		panic(err)
	}
	l.wfd = int(r0)
	l.AddRead(l.wfd)
	return l
}
```

`syscall.Syscall(syscall.SYS_EVENTFD2, 0, 0, 0)` 创建了一个 eventfd ，然后将这个 eventfd 注册到了 epoll 监听可读事件。当  `syscall.Write(p.wfd, []byte{0, 0, 0, 0, 0, 0, 0, 1})` 时候，epoll 就会唤醒。

但是，我翻了好久，也没有找到 evio 在哪里读取 eventfd 写入的8个字节（epoll）。这是一个 bug，所以在 linux 机器上，这是不能用的。

> 这个bug会造成 epoll 不断唤醒，cpu被长期占用

当我们注册了 evio 的定时任务 Tick 回调函数，程序启动后会往 eventfd 里写入 8 个字节数据，但是 evio 并没有读取，并且 evio 使用的是 epoll 的默认模式 LT，即只要可读缓冲区里还有数据，epoll 会一直不断唤醒，这是一个严重的 bug，作者应该没有在 linux 环境下严格测试过。

我们抛开这个 bug， 继续来看 note 为 error 类型的情况。在 serve 函数中，当函数退出时，通过 `l.poll.Trigger(errClosing)` 来通知每个事件循环退出。

```go
func  serve(events Events, listeners []*listener) error {
...

defer func() {
	// wait on a signal for shutdown
	s.waitForShutdown()

	// notify all loops to close by closing all listeners
	for _, l := range s.loops {
		l.poll.Trigger(errClosing)
	}

	// wait on all loops to complete reading events
	s.wg.Wait()

	// close loops and all outstanding connections
	for _, l := range s.loops {
		for _, c := range l.fdconns {
			loopCloseConn(s, l, c, nil)
		}
		l.poll.Close()
	}
	//println("-- server stopped")
}()

...
```

当 note 为 *conn 这种情况，是用来提供给使用者主动唤醒当前事件循环

```go
func (c *conn) Wake() {
	if c.loop != nil {
		c.loop.poll.Trigger(c)
	}
}
```

### loopAccept

```go
c := l.fdconns[fd]
	switch {
	case c == nil:
		return loopAccept(s, l, fd)
```

```go
type loop struct {
	idx     int            // loop index in the server loops list
	poll    *internal.Poll // epoll or kqueue
	packet  []byte         // read packet buffer
	fdconns map[int]*conn  // loop connections fd -> conn
	count   int32          // connection count
}
```

fdconns 是用来存储已连接的TCP connection 信息，key 为 fd， value 为 *conn 。

当 epoll 唤醒时，如果 fd 不在当前事件循环的连接，那就说明它是新连接，则执行 loopAccept 。

```go
func loopAccept(s *server, l *loop, fd int) error {
	for i, ln := range s.lns {
		if ln.fd == fd {
			if len(s.loops) > 1 {
				switch s.balance {
				case LeastConnections:
					n := atomic.LoadInt32(&l.count)
					for _, lp := range s.loops {
						if lp.idx != l.idx {
							if atomic.LoadInt32(&lp.count) < n {
								return nil // do not accept
							}
						}
					}
				case RoundRobin:
					idx := int(atomic.LoadUintptr(&s.accepted)) % len(s.loops)
					if idx != l.idx {
						return nil // do not accept
					}
					atomic.AddUintptr(&s.accepted, 1)
				}
			}
			if ln.pconn != nil {
				return loopUDPRead(s, l, i, fd)
			}
			nfd, sa, err := syscall.Accept(fd)
			if err != nil {
				if err == syscall.EAGAIN {
					return nil
				}
				return err
			}
			if err := syscall.SetNonblock(nfd, true); err != nil {
				return err
			}
			c := &conn{fd: nfd, sa: sa, lnidx: i, loop: l}
			l.fdconns[c.fd] = c
			l.poll.AddReadWrite(c.fd)
			atomic.AddInt32(&l.count, 1)
			break
		}
	}
	return nil
}
```

因为 evio 支持多地址监听，所以会存在多个 listener ，也就是 s.lns 。

第一步，先遍历所有的 listener 看看当前 epoll 中就绪的 fd 是哪一个 listener ，然后执行客户端的负载策略，决定新的客户端连接放在哪一个事件循环中。

这里关于客户端的负载策略，evio 利用了 epoll 的惊群效果，所有的事件循环都会唤醒进入loopAccept，不符合负载策略直接 return nil。 关于这边的更多细节，可以看我的另一篇文章 [【Golang 网络库 evio 一些问题/bug和思考】](/posts/open-source/evio-code-bug/)。

接下来就是常规操作了，` syscall.Accept(fd)` 接受连接，然后 ` syscall.SetNonblock(nfd, true)` 设置成非阻塞模式，`	l.poll.AddReadWrite(c.fd)` 最后加入事件循环，注册可读可写事件。

### loopOpened

```go
func loopOpened(s *server, l *loop, c *conn) error {
	c.opened = true
	c.addrIndex = c.lnidx
	c.localAddr = s.lns[c.lnidx].lnaddr
	c.remoteAddr = internal.SockaddrToAddr(c.sa)
	if s.events.Opened != nil {
		out, opts, action := s.events.Opened(c)
		if len(out) > 0 {
			c.out = append([]byte{}, out...)
		}
		c.action = action
		c.reuse = opts.ReuseInputBuffer
		if opts.TCPKeepAlive > 0 {
			if _, ok := s.lns[c.lnidx].ln.(*net.TCPListener); ok {
				internal.SetKeepAlive(c.fd, int(opts.TCPKeepAlive/time.Second))
			}
		}
	}
	if len(c.out) == 0 && c.action == None {
		l.poll.ModRead(c.fd)
	}
	return nil
}
```

loopOpened 是在 loopAccept 执行完成后，epoll 会立马再次唤醒然后执行的。

因为在 loopAccept 中最后将新的客户端连接加入 epoll 管理时注册的是可读可写事件，当前的内核写缓冲区肯定是为空的，所以 epoll 会再次唤醒。

```go
...
case !c.opened:
	return loopOpened(s, l, c)
...
```

 唤醒后会执行到这个 case  `case !c.opened:`，因为在 loopAccept 中并没有去设置这个值。

loopOpened 内部的操作，主要就是设置一下 conn 的属性，然后调用客户注册的回调函数 `events.Opened` 。

如果在回调函数中，没有给客户端发送数据，则需要重新注册，只注册可读事件，不然 epoll 会一直唤醒（可写事件）。

### loopAction

```go
func loopAction(s *server, l *loop, c *conn) error {
	switch c.action {
	default:
		c.action = None
	case Close:
		return loopCloseConn(s, l, c, nil)
	case Shutdown:
		return errClosing
	case Detach:
		return loopDetachConn(s, l, c, nil)
	}
	if len(c.out) == 0 && c.action == None {
		l.poll.ModRead(c.fd)
	}
	return nil
}
```

```go
case c.action != None:
	return loopAction(s, l, c)
```

loopAction 会在 `case c.action != None:` 的情况下执行， c.action 是执行完用户回调函数后会被赋值的状态。

在会有 action 的 loopXXX 中都会有如下类似操作。

```go
if len(c.out) != 0 || c.action != None {
	l.poll.ModReadWrite(c.fd)
}
```

也就是说 loopAction 依赖于 epoll 被可写事件再次唤醒来执行，这样会不会有问题呢？ 内核缓冲区满了？？ 

loopAction 内部的主要操作就是根据 action 做一些处理，关闭连接等等。

### loopRead 和  loopWrite

loopRead 和  loopWrite 主要就是调用系统调用读取和发送数据，并且调用用户回调函数，根据回调函数返回值来重新注册 epoll 的可读可写事件。

```go
func loopRead(s *server, l *loop, c *conn) error {
	var in []byte
	n, err := syscall.Read(c.fd, l.packet)
	if n == 0 || err != nil {
		if err == syscall.EAGAIN {
			return nil
		}
		return loopCloseConn(s, l, c, err)
	}
	in = l.packet[:n]
	if !c.reuse {
		in = append([]byte{}, in...)
	}
	if s.events.Data != nil {
		out, action := s.events.Data(c, in)
		c.action = action
		if len(out) > 0 {
			c.out = append([]byte{}, out...)
		}
	}
	if len(c.out) != 0 || c.action != None {
		l.poll.ModReadWrite(c.fd)
	}
	return nil
}
```

调用 `n, err := syscall.Read(c.fd, l.packet)` 读取内核缓冲区的数据，如果返回出错 `err == syscall.EAGAIN` 意思是再试一次，直接返回。

如果 n == 0 或者 err 错误不为 syscall.EAGAIN ，则说明对方关闭了连接或是其他错误，直接 loopCloseConn 。

然后调用用户回调函数 s.events.Data ，根据返回值做相应操作。`c.action = action` 

如果 out 里有数据，则赋给 c.out , 并且注册可读可写事件。

如果 `c.action != None` ，同样需要注册可读可写事件，原因上面已经说过了。

loopWrite 操作也大同小异，就不细说了。

但是其实关于 loopWrite 和 loopRead 的处理是会有 bug 的，详情可以看另一篇文章。

## 推荐库

- [gev](https://github.com/Allenxuxu/gev) 一个轻量、快速的基于 Reactor 模式的非阻塞 TCP 网络库。
