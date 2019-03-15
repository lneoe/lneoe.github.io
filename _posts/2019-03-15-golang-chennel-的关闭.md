---
layout: post
title: golang chennel 的关闭
date: 2019-03-15 11:44 +0800
---

> TL;DR
- 避免多个 `goroutine` 共享 `channel write` 权限  
- 只在一个地方 close
- 不关闭

大概一个月前跟几个朋友一起讨论过关于 golang chnanel 关闭的问题, 最近清闲一些总一下简单的总结  

这次目的并不在于介绍 block 和 non-block channel 之间的区别, 而是要探讨一下如果安全的关闭 channel.  

使用 golang 开发过程中在 goroutine 的通信或同步中经常要使用的 channel  

channel 虽然让并发编程变得更方便, 然而在我的经历中多个 goroutine 共享 channel 的时候关闭起来却让人相当头疼, 我们先来看一个常见的例子  
```golang
package main

import "fmt"

func writeCh(ch chan<- int) {
	for i := 0; i < 10; i++ {
		ch <- i
	}

	close(ch)
}

func main() {
	ch := make(chan int)
	go writeCh(ch)

outer:
	for {
		select {
		case i, ok := <-ch:
			if !ok {
				fmt.Println("channel closed")
				break outer
			}

			fmt.Println("receive from channel: ", i)
		}
	}
}
```
在这个例子中, 我们只有一个 `writeCh` 并且他只会背调用一次, `writeCh` 最后执行了 `close(ch)`, 并且我们使用 `i,ok := <-ch` 接收到了`close`事件, 所以程序运行正常.  

然而在实际生产中的我们的逻辑可能并不是这么简单就能表述清楚, 虽然我们应该尽量向这个例子的逻辑靠近  

再来来假设一个复杂点的应用场景  
> 现在我们要做一个 tcpConn 的监听 并且在 `io.EOF` 的时候关闭连接, 或者在外部命令要求关闭的时候关闭监听  

```golang
// server.go

func server() {
	listener, err := net.Listen("tcp", "127.0.0.1:6000")
	if err != nil {
		fmt.Println("server: ", err)
		os.Exit(1)
	}

	conn, err := listener.Accept()
	if err != nil {
		fmt.Println("server: ", err)
		os.Exit(1)
	}

	go func() {
		for {
			data := make([]byte, 512)
			n, err := conn.Read(data)
			if err != nil {
				fmt.Println(err)
				break
			}

			//fmt.Println("server: ", string(data[:n]))
			conn.Write(data[:n])
		}
	}()
}
```
以上是 `server.go` 我们可以先跳过不看, 这里只是临时开启一个监听 tcp 的服务  

我们看看 `client.go`,  我们重点关注 `listener` 和 `client` 这两个函数  
```golang
// client.go
func write(conn net.Conn) {
	for {
		s := "hello"
		conn.Write([]byte(s))
		time.Sleep(3 * time.Second)
	}
}

func listener(conn net.Conn, recv chan<- []byte) {
	buf := make([]byte, 1024)
	for {
		n, err := conn.Read(buf)
		if err != nil && err == io.EOF {
			break
		}
		recv <- buf[:n]
	}

	close(recv)
}

func client() {
	conn, err := net.Dial("tcp", "127.0.0.1:6000")
	if err != nil {
		fmt.Println("client: ", err)
		os.Exit(1)
	}

	go write(conn)

	ch := make(chan []byte)
	go listener(conn, ch)

	for data := range ch {
		s := string(data)
		fmt.Println("client: ", s)
		if s == "kick_out" {
			tcpConn, _ := conn.(*net.TCPConn)
            // 这里关闭了 tcpConn 的 read, 导致 listener 退出循环
			tcpConn.CloseRead()

			close(ch)
		}
	}
}
```
我们看到 `listener` 在发现读取到 `err` 的时候会退出循环, 并且执行了 `close(recv)`, 这跟我们上一个例子并没有多少特殊的地方,   
但是当我们接收到的消息是 `kick_out` 的时候 `client` 除了 `CloseRead` 还执行了 `close(ch)` 最终会导致 `listener` panic  
```text
client:  hello
client:  hello
client:  kick_out
panic: close of closed channel
```

现在这个例子中 `channel` 的关闭导致了程序 panic, 那么怎么去解决这种问题?  

- 不用 `channel`  
我们可以使用其他的东西去接收消息, 使用 slice 或者消息队列, 这个取决于系统架构的复杂度以及对处理时间的要求  
不过显然我不会采用这个方案

- 只在一个地方关闭 `channel`  
在这个例子里面我们可以只在 `listener` 执行 `close` 就可以安全的退出, 但是这里的两次 `close` 也属于场景比较简单的情况,   
至于执行 `close` 原则: 建议是在 write 的地方去关闭, 虽然可能提供了一个 `cli.Stop()` 的接口, 但是总是会被不同的地方被调用而导致重复调用  
为了避免重复的 `close` 我们可以使用 `sync.Once`, 但是依然还可能会面临到 `send on closed channel` 的问题

- 避免多个 `goroutine` 共享 channel write 权限  
因为这回导致你的不确定应该去哪个 `goroutine` 执行 `close`, 也很难保证 `ch <- ` 的安全性

- 不关闭, 等待 gc 回收  
为了避免出现 `send on closed channel` 可能也会采取不关闭的方式,  不过这看起来是一种逃避的解决方式, 虽然不会导致 panic, 但是额外浪费了资源并不是一种好的编码风格  
虽然我们可以不关闭 channel, 但是依然需要对 channel 进行排空, 以便与 gc 即时进行回收

#  Conclusion
实际开发中可能还会用到 `context` 进行一些辅助, 但是如何避免 `close of closed channel` 和 `send on closed channel` 依然是一个让我很头疼的问题, 甚至会怀疑我的代码是不是不应该这样写