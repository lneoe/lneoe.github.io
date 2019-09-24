---
layout: post  
title: golang source code reading  
date: 2019-09-24 18:03 +0800  
categories: [golang, golang runtime 笔记]  
comments: true  
---

之前写过几篇 blog, 本来打算每月写一篇的， 并且 drafts 里面放了好几篇， 不过总是断断续续的写到最终竟然连想要表达的观点也忘记了只好一直放置了.  
最近阅读 golang 源码作一些笔记避免时间一长有些东西就忘记了, 算是暂时开了一个新坑, 可能会有许多错误的地方, 欢迎讨论  
这并不是一个系列，就只是看到那里于是就记下来.  

# Prerequisites
为了更好的理解你需要提前了解的知识点
- [go compiler directives](https://golang.org/cmd/compile/#hdr-Compiler_Directives)
- `unsafe.Pointer`

# Notes
- [go chan source code]({% post_url 2019-09-24-golang-chan-source-code %})