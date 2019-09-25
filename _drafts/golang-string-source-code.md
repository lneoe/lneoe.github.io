---
layout: post  
title: golang-string-source-code  
categories: [golang]
tags: [golang, golang runtime 笔记]  
comments: true  
---

# stringStruct
golang string 的实现
```go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
# rawstring
