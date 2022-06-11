---
weight: 15
title: "第 15 章 平等性"
draft: false
---

# 第 15 章 平等性

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## Go 的平等性

在 Go 中，有很多不同的方法来比较平等性，但没有一个是完美的。

## == 和 != 操作符

在 Go 中，== 运算符是最简单、最有效的比较方法，但它只对某些类型有效。
最值得注意的是，它对切片或哈希不起作用。如果采用这种方式，切片和哈希只能与 nil 进行比较。

你可以使用 == 比较基本类型，如 int 和 string，还有数组和结构体中的元素本身也可以使用 == 进行比较：

```Golang
package main

import "fmt"

type compareStruct1 struct {
    A int
    B string
    C [3]int
}

func main() {
    s1 := compareStruct1{}
    s2 := compareStruct1{}
    fmt.Println(s1 == s2) // 正常运行，打印 true
}
```

在结构体中，一旦添加了一个不能使用 == 比较的属性，就需要用另一种方式来比较：

```Golang
package main

import "fmt"

type compareStruct2 struct {
    A int
    B string
    C []int // 将 C 的类型从数组改为切片
}

func main() {
    s1 := compareStruct2{}
    s2 := compareStruct2{}
    // 无效操作: s1 == s2
    // (包含 []int 的结构体不能被比较)
    fmt.Println(s1 == s2)
}
```

## 编写专门的比较代码

如果性能很重要，而且需要比较稍微复杂的类型，最好的选择可能是手写比较代码：

```Golang
type compareStruct struct {
    A int
    B string
    C []int
}

func (s *compareStruct) Equals(s2 *compareStruct) bool {
    if s.A != s2.A || s.B != s2.B || len(s.C) != len(s2.C) {
        return false
    }
    for i := 0; i < len(s.C); i++ {
        if s.C[i] != s2.C[i] {
            return false
        }
    }
    return true
}
```

像上面代码中的比较函数可以自动生成，但在写这篇文章时，我还不知道有什么工具可以做到这一点。

## reflect.DeepEqual

在 Go 中，DeepEqual 是最通用的比较方法，它可以处理大部分平等性比较。但这里有一个问题：

```Golang
var (
    c1 = compareStruct{
        A: 1,
        B: "hello",
        C: []int{1, 2, 3},
    }
    c2 = compareStruct{
        A: 1,
        B: "hello",
        C: []int{1, 2, 3},
    }
)

func BenchmarkManual(b *testing.B) {
    for i := 0; i < b.N; i++ {
        c1.Equals(&c2)
    }
}

func BenchmarkDeepEqual(b *testing.B) {
    for i := 0; i < b.N; i++ {
        reflect.DeepEqual(c1, c2)
    }
}
```

输出：

```
BenchmarkManual-8 217182776 5.51 ns/op 0 B/op 0 allocs/op
BenchmarkDeepEqual-8 2175002 559 ns/op 144 B/op 8 allocs/op
```

在该结构体的比较例子中，DeepEqual 比手动编写的代码来要慢100倍。

请注意，DeepEqual 会比较结构体中未导出的（小写的）字段。
另外，两个不同的类型永远不会被认为是深度相等的，即使它们是具有相同字段和值的不同结构体。

## 不可比较性

有些存在是不能被比较的，甚至被认为是与自己不相等的，例如具有 NaN 值的浮点变量或 func 类型。
例如，如果你在一个结构体中拥有这样的字段，那么如果使用 DeepEqual 进行比较，该结构体将不等于其自身：

```Golang
func TestF(t *testing.T) {
    x := math.NaN
    fmt.Println(reflect.DeepEqual(x, x)) // false
    fmt.Println(reflect.DeepEqual(TestF, TestF)) // false
}
```

## bytes.Equal

bytes.Equal 是专门为字节切片设计的一种比较方法。它比简单地用 for 循环比较两个切片要快得多。

值得注意的是，bytes.Equal 函数认为空切片和 nil 切片是相等的，而 reflect.DeepEqual 则相反。
