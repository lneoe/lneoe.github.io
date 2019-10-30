---
layout: post
title: golang-benchmark
categories: [golang]
---

```go
package main_test
import (
    "testing"
    "time"
)


func BenchmarkTimeFormatHour(b *testing.B) {
 for i := 0; i < b.N; i++ {
    time.Now().Format("15:00")
 }
}
```

`go test -test.bench='.*' -test.benchtime=10x`
