---
weight: 16
title: "第 16 章 内存管理"
draft: false
---

# 第 16 章 内存管理

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 结构体应该按值传递还是按引用传递？

在 Go 中，函数的参数总是按值传递。当一个结构体（或数组）类型的变量被传递到函数中时，整个结构体会被复制。
如果结构体的指针被传递，那么这个指针会被复制，但它所指向的结构体不会被复制。拷贝的是 8 个字节内存（对于 64 位架构），而不是该结构体的大小。
那么，这是否意味着将结构体作为指针传递会更好？经典回答——这要看情况。

分配一个结构体（或数组）的指针：

1. 将其放在堆中，而不是像通常情况下放到栈中；
2. 垃圾收集器来管理堆内存的分配。

如果你想复习一下栈与堆的关系，可以看看这个 [stackoverflow 帖子](https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap)。就本章而言，了解这些就足够了：堆栈——快，堆——慢。

这意味着如果分配结构体比传递它们更频繁，那么在栈中复制它们会更快：

```Golang
package test

import (
    "testing"
)

type myStruct struct {
    a, b, c int64
    d, e, f string
    g, h, i float64
}

func byValue() myStruct {
    return myStruct{
        a: 1, b: 1, c: 1,
        d: "foo", e: "bar", f: "baz",
        g: 1.0, h: 1.0, i: 1.0,
    }
}

func byReference() *myStruct {
    return &myStruct{
        a: 1, b: 1, c: 1,
        d: "foo", e: "bar", f: "baz",
        g: 1.0, h: 1.0, i: 1.0,
    }
}

func BenchmarkByValue(b *testing.B) {
    var s myStruct
    for i := 0; i < b.N; i++ {
        // 拷贝整个结构体
        // 但要通过栈内存来实现
        s = byValue()
    }
    _ = s
}

func BenchmarkByReference(b *testing.B) {
    var s *myStruct
    for i := 0; i < b.N; i++ {
        // 在堆上为结构体分配内存
        // 并只返回它的一个指针
        s = byReference()
    }
    _ = s
}
```

输出：

```
BenchmarkByValue-8 476965734 2.499 ns/op 0 B/op 0 allocs/op 
BenchmarkByReference-8 24860521 45.86 ns/op 96 B/op 1 allocs/op
```

在这个初级案例中，按值传递（不涉及堆或垃圾收集器）的速度是按引用传递的 18 倍。

为了说明这个观点，让我们做一个相反的初级案例，分配一次结构体，只把它传递给函数：

```Golang
var s = myStruct{
    a: 1, b: 1, c: 1,
    d: "foo", e: "bar", f: "baz",
    g: 1.0, h: 1.0, i: 1.0,
}

func byValue() myStruct {
    return s
}

func byReference() *myStruct {
    return &s
}
```

输出：

```
BenchmarkByValue-8 471494428 2.509 ns/op 0 B/op 0 allocs/op
BenchmarkByReference-8 1000000000 0.2484 ns/op 0 B/op 0 allocs/op
```

当变量只被传来传去，但不被分配时——通过引用会快很多。

想要了解更多细节，请查看这篇 Vincent Blanchon 的 [经典文章](https://medium.com/a-journey-with-go/go-should-i-use-a-pointer-instead-of-a-copy-of-my-struct-44b43b104963)。

虽然这一章是关于哪个更快，但在许多应用中，代码的清晰度和一致性将比性能更重要，当然，这又是一个单独的讨论。
总之，不要认为复制变量会很慢，如果性能很重要的话，请使用优秀的 Go 分析工具。

## 给 C 语言开发者的说明

在 Go 中，对内存管理的要求更为严格。指针运算是不允许的，也不可能有悬空的指针。
但像这样的事情是完全可以的：

```Golang
func byReference() *myStruct {
    return &myStruct{
        a: 1, b: 1, c: 1,
        d: "foo", e: "bar", f: "baz",
        g: 1.0, h: 1.0, i: 1.0,
    }
}
```

Go 的编译器很智能，会将该结构体移至堆中。
