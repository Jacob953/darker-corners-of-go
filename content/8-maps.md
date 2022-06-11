---
weight: 8
title: "第 8 章 哈希"
draft: false
---

# 第 8 章 哈希

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 哈希的迭代顺序是随机的（实则不然）

技术上来说，哈希的迭代顺序是“未定义的”。在 Go 中，哈希的内部会使用一个哈希表，所以迭代通常是按照元素在该表中的顺序进行的。
但这个顺序是不可靠的，当新的元素被添加到哈希中时，这个顺序会随着哈希表的增长而改变。
在 Go 的早期时候，这对那些没有阅读说明，并且依赖按一定顺序迭代哈希的程序员来说，是一个巨坑。
为了帮助程序员尽早，而不是在生产中发现这些问题，Go 的开发者将哈希的迭代变得随机：

```Golang
package main

import "fmt"

func main() {
    // 添加顺序元素
    // 使哈希看起来像是按顺序迭代的
    m := map[int]int{0: 0, 1: 1, 2: 2, 3: 3, 4: 4, 5: 5}
    for i := 0; i < 5; i++ {
        for i := range m {
            fmt.Print(i, " ")
        }
        fmt.Println()
    }
    // 添加更多元素
    // 使哈希的哈希表增长并重新排序元素
    m[6] = 6
    m[7] = 7
    m[8] = 8
    for i := 0; i < 5; i++ {
        for i := range m {
            fmt.Print(i, " ")
        }
        fmt.Println()
    }
}
```

输出：

```Golang
3 4 5 0 1 2
5 0 1 2 3 4
0 1 2 3 4 5
1 2 3 4 5 0
0 1 2 3 4 5
0 1 3 6 7 2 4 5 8
1 3 6 7 0 4 5 8 2
2 4 5 8 0 1 3 6 7
0 1 3 6 7 2 4 5 8
0 1 3 6 7 2 4 5 8
```

在上面的例子中，当哈希被初始化时，元素 1 到 5 被按顺序添加到哈希表中。
前五行打印的数字都是按顺序写的 0 到 5。在 Go 中，这只是随机从某个元素开始迭代。
向哈希中添加更多的元素会使哈希表增长，从而重新排列整个哈希表的顺序。
打印最后 5 行时，就不再有任何明显的顺序。如果必要，你可以在 [Go 的 Map 源代码](https://github.com/golang/go/blob/master/src/runtime/map.go) 中找到所有的信息。

## 检查哈希的键是否存在

访问哈希中不存在的元素时，会返回哈希值类型的零值。如果是一个整型的哈希，它将返回 0，对于引用类型，它将返回 nil。
为了检查元素是否存在于哈希中，有时一个零值就足够了。
例如，如果是一个值类型为指向结构体的指针的哈希，那么在访问哈希时，如果得到一个 nil 值，这意味着你寻找的元素不在哈希中。
但是，如果是一个值类型为布尔值的哈希，因为默认零值为“false”，所以它不足以判断元素的值为“false”，还是元素根本不存在于哈希中。
因此，访问哈希元素时，会返回一个可选的第二参数，以显示该元素是否真的在哈希中：

```Golang
package main

import "fmt"

func main() {
    m := map[int]bool{1: false, 2: true, 3: true}
    
    // 打印为 false 时
    // 不清楚该元素的值是 false，或者哈希中不存在此元素
    // 因为返回的默认零值为 false
    fmt.Println(m[1])
    val, exists := m[1]
    fmt.Println(val, exists) // 打印 false true
}
```

## 哈希是指针

虽然切片类型是一个结构体（值类型），它有一个指向数组的指针，但哈希本身就是一个指针。
切片的零值完全是可用的。你可以使用 append 函数来添加元素，也可以获得切片的长度。
而哈希则不同，尽管 Go 的开发者希望让哈希的零值完全可用，但没找到有效的途径来实现这一点。
在 Go 中，map 关键字是 *runtime.hmap 类型的别名。它的零值是 nil，nil 哈希可以被读取，但不能被写入：

```Golang
package main

import "fmt"

func main() {
    var m map[int]int // 一个 nil 哈希
    // 读取 nil 哈希的长度是可以的，打印 0
    fmt.Println(len(m))
    // 读取 nil 哈希也是确定的，打印 0（哈希的值类型的默认值）
    fmt.Println(m[10])
    m[10] = 1 // 警告: 分配内存给 nil 哈希中的元素
}
```

nil 哈希可以被读取，因为哈希的元素是通过像这样的函数（来自 runtime/map.go）来访问的：

```Golang
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
```

这个函数检查哈希是否为 nil，如果是，则返回哈希值类型的零值。注意，它不能创建一个哈希。如果要创建完全可用的哈希，必须使用 make：

```Golang
package main

import "fmt"

func main() {
    m := make(map[int]int)
    m[10] = 11         // 现在就都没问题了
    fmt.Println(m[10]) // 打印 11
}
```

由于哈希是指针，把它传递给函数时，就会传递指向同一个 map 数据结构的指针：

```Golang
package main

import "fmt"

func f1(m map[int]int) {
    m[5] = 123
}

func main() {
    m := make(map[int]int)
    f1(m)
    fmt.Println(m[5]) // 打印 123
}
```

当哈希的指针被传递给函数时，指针的值会被复制（Go 通过值传递一切，包括指针）。在函数中创建一个新的哈希会改变指针副本的值，所以这是不可行的：

```Golang
package main

import "fmt"

func f1(m map[int]int) {
    m = make(map[int]int)
    m[5] = 123
}

func main() {
    var m map[int]int
    f1(m)
    fmt.Println(m[5])     // 打印 0
    fmt.Println(m == nil) // true
}
```

## struct{} 类型

在 Go 中，没有集合这个数据结构（类似于有键但无值的哈希，如 C++ 的 std::set 或 C# 的 HashSet）。
使用哈希来代替是很容易的。一个小技巧是使用 struct{} 类型作为哈希值类型：

```Golang
package main

import (
    "fmt"
)

func main() {
    m := make(map[int]struct{})
    m[123] = struct{}{}
    _, keyexists := m[123]
    fmt.Println(keyexists) // true
}
```

通常，这里会使用布尔值，但如果使用 struct{} 值类型的哈希会节省一点内存。struct{} 类型实际上就是零字节大小：

```Golang
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    fmt.Println(unsafe.Sizeof(false)) // 1
    fmt.Println(unsafe.Sizeof(struct{}{})) // 0
}
```

## 哈希的容量

哈希是一个相当复杂的数据结构。虽然在创建它时，可以指定它的初始容量，但之后就不可能得到它的容量了（至少不能用 cap 函数）：

```Golang
package main

import (
    "fmt"
)

func main() {
    m := make(map[int]bool, 5) // 初始化容量为 5
    fmt.Println(len(m)) // len 函数是可以的
    fmt.Println(cap(m)) // 对于 cap 函数来说，m (map[int]bool 类型) 是非法参数 
}
```

## 哈希值是不可寻址的

在 Go 中，哈希是以哈希表的形式实现的，而哈希表需要在哈希增长或缩小时移动其元素。由于这个原因，Go 不允许获取哈希元素的地址：

```Golang
package main

import "fmt"

type item struct {
    value string
}

func main() {
    m := map[int]item{1: {"one"}}
    fmt.Println(m[1].value) // 读取结构值是可以的
    addr := &m[1]           // 错误: 无法读取 m[1] 的地址
    
    // 错误: 在哈希中，不能赋值给结构体字段 m[1].value
    m[1].value = "two"      
}
```

有一个 [允许赋值给一个结构体字段的提议](https://github.com/golang/go/issues/3117)（m[1].value = "two"），因为在这种情况下，只通过赋值，值字段的指针不会被保留。
但由于 "subtle corner cases"，目前还没有具体计划关于此提议何时或是否会实施。

有一种变通方法，但整个结构体需要重新被分配到哈希中：

```Golang
package main

type item struct {
    value string
}

func main() {
    m := map[int]item{1: {"one"}}
    tmp := m[1]
    tmp.value = "two"
    m[1] = tmp
}
```

另外，指向结构体的指针映射也可以成功。在这种情况下，m[1] 的值是一个 *item 类型。
Go 不需要获取指向映射值的指针，因为该值本身已经是指针了。
哈希表会在内存中移动指针，但是如果你复制一个 m[1] 的值，它将一直指向同一个元素，所以这也是一样的：

```Golang
package main

import "fmt"

type item struct {
    value string
}

func main() {
    m := map[int]*item{1: {"one"}}

    // Go 在这里不需要访问 m[1] 的地址。
    // 因为它已经是指针
    m[1].value = "two"      
    fmt.Println(m[1].value) // two
    addr := &m[1] // 同样的错误: 不能访问 m[1] 的地址
}
```

值得注意的是，切片和数组不存在这个问题：

```Golang
package main
import "fmt"
func main() {
    slice := []string{"one"}
    saddr := &slice[0]
    *saddr = "two"
    fmt.Println(slice) // [two]
}
```

## 数据竞争

在 Go 中，普通哈希对于并发访问并不安全。哈希经常被用来在 goroutine 之间共享数据，
但对哈希的访问必须通过 sync.Mutex、sync.RWMutex，其他内存锁进行同步，或者与 Go 的通道协调以防止并发访问，但以下情况除外：

> 只有当更新发生时，哈希访问才是不安全的。只要所有的 goroutine 只是在阅读查找哈希中的元素，包括使用 for-range 循环遍历哈希，
> 而不是通过向元素赋值或进行删除来改变哈希，那么它们在不同步的情况下并发访问哈希就是安全的。
>
> https://golang.org/doc/faq

```Golang
package main

import (
    "math/rand"
    "time"
)

func readWrite(m map[int]int) {
    // 对哈希做一些随机的读和写
    for i := 0; i < 100; i++ {
        k := rand.Int()
        m[k] = m[k] + 1
    }
}

func main() {
    m := make(map[int]int)
    // 启动 goroutine 来同时读写哈希
    for i := 0; i < 10; i++ {
        go readWrite(m)
    }
    time.Sleep(time.Second)
}
```

输出：

```
致命错误: 同时进行的哈希读取和写入
致命错误: 同时进行的哈希写入
...
```

在这种情况下，可以使用 Mutex 同步访问哈希。下面的代码将按预期工作：

```Golang
package main

import (
    "math/rand"
    "sync"
    "time"
)

var mu sync.Mutex

func readWrite(m map[int]int) {
    mu.Lock()
    // defer unlock mutex 将解锁 mutex
    // 即使这个 goroutine 会警告
    defer mu.Unlock()
    for i := 0; i < 100; i++ {
        k := rand.Int()
        m[k] = m[k] + 1
    }
}

func main() {
    m := make(map[int]int)
    for i := 0; i < 10; i++ {
        go readWrite(m)
    }
    time.Sleep(time.Second)
}
```

## sync.Map

在 sync 包中，哈希有一个专门版本，对于多个 goroutine 的并发使用是安全的。
然而 Go 文档建议在大多数情况下使用带有锁或者协调的普通哈希。
因为 sync.Map 不是类型安全的，它类似于 map[interface{}]interface{}。如 sync.Map 文档所说：

> 哈希类型针对两种常见的使用情况进行了优化：
> (1)当一个给定键的条目只被写入一次，但被多次读取，就像在只会增长的缓存中，
> 或者(2)当多个 goroutine 读取、写入和覆盖不相干的键集的条目时。
> 在这两种情况下，与 Go 中的普通哈希搭配单独的 Mutex 或 RWMutex 相比，使用 sync.map 可以大大减少锁争用。
> 
> https://github.com/golang/go/blob/master/src/sync/map.go
