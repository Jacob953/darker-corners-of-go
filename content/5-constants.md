# Go 鲜为人知的角落｜第 5 章 常量｜《Darker Corners of Go》独译（已授权）

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## iota

在 Go 中，iota 是起始常量编号。 它并不像人们想象的那样意味着 "从零开始"。它是当前 const 块中一个常数的索引：

```Golang
const (
    myconst = "c"
    myconst2 = "c2"
    two = iota // 2
)
```

使用两次 iota 并不能重置编号：

```Golang
const (
    zero = iota // 0
    one // 1
    two = iota // 2
)
```
