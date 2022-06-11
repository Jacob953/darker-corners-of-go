---
weight: 18
title: "第 18 章 时间"
draft: false
---

# 第 18 章 时间

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 使用 time.LoadLocation 从文件中读取数据

在 Go 中，这是我个人而言最喜欢的一个坑。在时区之间转换，首先要需要加载位置信息。事实证明，每次调用 time.LoadLocation 都会从一个文件中读取数据。在格式化大量 CSV 报告的每一行时，这不是最好的做法：

```Golang
package main

import (
    "testing"
    "time"
)

func BenchmarkLocation(b *testing.B) {
    for n := 0; n < b.N; n++ {
        loc, _ := time.LoadLocation("Asia/Kolkata")
        time.Now().In(loc)
    }
}

func BenchmarkLocation2(b *testing.B) {
    loc, _ := time.LoadLocation("Asia/Kolkata")
    for n := 0; n < b.N; n++ {
        time.Now().In(loc)
    }
}
```

输出：

```
BenchmarkLocation-8 16810 76179 ns/op 58192 B/op 14 allocs/op
BenchmarkLocation2-8 188887110 6.97 ns/op 0 B/op 0 allocs/op
```
