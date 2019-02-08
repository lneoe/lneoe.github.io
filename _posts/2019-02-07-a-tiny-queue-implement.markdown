---
layout: post
title: a tiny queue implement
date: 2019-02-07 23:45 -0600
categories: golang
comments: true
---

> TL;DR  
使用 `slice` 缓冲消息, 利用 `sync.Cond` 实现管道内有消息时通知 consumer  

在做棋牌游戏的时候遇到需要队列来实现处理游戏消息的情况, 但是又不想再起一个 MQ 来增加架构复杂度, 于是自己实现了一个进程内的队列

#### 1. 缓冲区实现
直接使用 golang `slice` 来实现消息缓冲, 缺点是 golang 的 `slice` 实现会导致开支变大, 不过使用场景内问可以忽略  
```golang
// queue.go

// Message interface
type Message interface {
    GetID() int64
}

// Queue structure
type Queue struct {
	queue  []Message
	lock sync.Mutex
}
// Append 发送消息
func (q *Queue) Append() bool {}
// Pop 消费消息
func (q *Queue) Pop() Message {}

// NewQ return Queue
func NewQ() *Queue {
    return &Queue{queue: make([]Message,10)}
}

// Q exportable Queue var
var Q *Queue

init() {
    Q = NewQ()
}

// consumer.go
func main() {
    for {
        msg := queue.Q.Pop()
        fmt.Print("MessageID: ", msg.GetID())
    }
}

// producer.go
func main() {
    var i int64
    for ;i<10;i++ {
        m := Message{id: i+1}
        queue.Q.Append(m)
        time.sleep(500 * time.Millisecond)
    }
}
```
但是要想 `Queue.Pop` 在有消息的时候直接返回, 没有消息的时候一直阻塞直到新消息进入, 这样的实现是不够的  
  

#### 2. 使用 `sync.Cond`
使用标准库的 `sync.Cond` 如果缓冲区没有消息就一直等待, 知道有消息, `sync.Cond` 的使用方式也很简单  
```golang 
// goroutine 1
cond.Lock()
cond.Wait() // cond.Signal()
cond.Unlock()

// goroutine 2
cond.Signal()
```  
需要注意的是是 `sync.Cond` 在 `cond.Wait()` 前需要 `cond.Lock()` 之后需要 `cond.Unlock`

下面是添加了 `sync.Cond` 的 Queue 实现  

``` golang
// queue.go

package main

import (
	"sync"
)

type Message interface {
	GetID() int64
}

type Queue struct {
	closed bool
	queue  []Message

	lock sync.Mutex

	monitor *sync.Cond
}

func NewQ() *Queue {
	q := Queue{queue: nil}
	q.monitor = sync.NewCond(&sync.Mutex{})

	return &q
}

func (q *Queue) Append(msg Message) bool {
	if q.closed {
		return false
	}

	q.lock.Lock()
	defer q.lock.Unlock()

	q.queue = append(q.queue, msg)

	q.monitor.Signal()
	return true
}

func (q *Queue) Close() {
	if q.closed {
		return
	}

	q.closed = true
	q.monitor.Signal()
}

func (q *Queue) Pop() (Message, bool) {
	q.lock.Lock()
	for len(q.queue) > 0 {

		msg := q.queue[0]
		q.queue = q.queue[1:]

		q.lock.Unlock()
		return msg, true
	}
	q.lock.Unlock()

	q.monitor.L.Lock()
	q.monitor.Wait()
	q.monitor.L.Unlock()
	if q.closed {
		return nil, false
	}

	q.lock.Lock()
	msg := q.queue[0]
	q.queue = q.queue[1:]
	q.lock.Unlock()
	return msg, true
}

```

`consumer.go` 代码
```golang
// consumer.go

type message struct {
	id int64
}

func (m message) GetID() int64 {
	return m.id
}

func main() {
	q := NewQ()
	go func() {
		var i int64 = 0
		for ; i < 5; i++ {
			msg := message{id: i + 1}
			q.Append(msg)
			time.Sleep(10 * time.Millisecond)
		}

		time.Sleep(5 * time.Second)
		q.Close()
	}()

	go func() {
		for {
			msg, live := q.Pop()
			if !live {
				fmt.Println("closed")
				break
			}

			fmt.Printf("got message id: %d\n", msg.GetID())
		}
	}()

	time.Sleep(10 * time.Second)
}

```

源码下载 [tiny_queue](https://github.com/lneoe/tiny_queue)
