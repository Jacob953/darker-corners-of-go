---
weight: 14
title: "第 14 章 继承"
draft: false
---

# 第 14 章 继承

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 重定义类型 vs 嵌入类型

Go 的类型系统可能更加实用。它不像 C++ 或 Java 那样是面向对象的。在 Go 中，你不能真正地继承结构体或接口（因为没有子类），但你可以把它们放在一起（嵌入），形成更复杂的结构体或接口。

> 嵌入与子类有一个重要的不同之处：当我们嵌入一个类型时，该类型的方法会成为外类型的方法；但是当它们被调用时，方法的接收者是内类型，而不是外类型。
>
> https://golang.org/doc/effective_go

除了嵌入类型，Go 还允许重新定义一个类型。 重定义会继承一个类型的字段，但没有继承其方法：

```Golang
package main

type t1 struct {
    f1 string
}

func (t *t1) t1method() {
}

// 嵌入类型
type t2 struct {
    t1
}

// 重定义类型
type t3 t1

func main() {
    var mt1 t1
    var mt2 t2
    var mt3 t3

    // 字段在所有情况下都会被继承
    _ = mt1.f1
    _ = mt2.f1
    _ = mt3.f1

    // 正常运行
    mt1.t1method()
    mt2.t1method()

    // mt3.t1method 未定义（t3 类型没有字段或者 t1method 方法）
    mt3.t1method()
}
```
