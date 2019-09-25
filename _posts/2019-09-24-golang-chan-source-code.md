---
layout: post  
title: golang chan source code reading  
date: 2019-09-24 18:04 +0800  
categories: [golang, golang runtime 笔记]  
comments: true  
---

# file
`runtime/chan.go`

# prerequisites
- `unsafe.Pointer`
- `waitq`
- `runtime.mutex`
- `sugog`
- `g`


# hchan
- `type hchan struct` 是 chan 的实现  

- 简单理解的话就是使用两个 queue 分别处理 `receive waiters` 和 `send waiters`, 然后用一个 `buf unsafe.Pointer` 来作具体数据的缓冲  


# chansend
- 如果发现有 `waiting receiver` 会把数据直接发给 `receiver` 不用去 `buf` 绕一圈  

- 如果 `buf` 足够就发送到 `buf`

- 如果 `buf` 已经满了就 `sendq.enqueue(sg)`, 就会阻塞当前 `g`


# chanrecv
逻辑跟 `chansend` 类似


# `select` 的处理  
- `selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool)`  
这段注释解释的非常清楚了
```golang
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
```
- `selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool)`  
跟 `selectnbsend` 同样的处理方式, 编译时对代码做了一次转换

- `selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool)`  
稍微有一些不同
```go
// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if c != nil && selectnbrecv2(&v, &ok, c) {
//		... foo
//	} else {
//		... bar
//	}
//
```

# Conclusion
`chan` 的实现逻辑非常清晰, 但是具体实现过程中有非常多的细节并不简单,  
比如 `lock` 获取/释放,  关于 `sudog` 可能之后还会作一些笔记
