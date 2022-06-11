---
weight: 5
title: "第 6 章 切片 & 数组"
draft: false
---

# 第 6 章 切片 & 数组

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 切片和数组

在 Go 中，切片和数组有类似的目的。它们的声明方式也几乎相同：

```Golang
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3}
    array := [3]int{1, 2, 3}

    // 让编译器来计算数组的长度
    // 这将是一个等同于 [3]int
    array2 := [...]int{1, 2, 3}
    fmt.Println(slice, array, array2)
}
```

输出：
```
[1 2 3] [1 2 3] [1 2 3]
```

切片就像上层附带有用功能的数组。在实现过程中，他们在内部使用指向数组的指针。
但是，切片是如此的方便，以至于我们很少在 Go 中直接使用数组。

## 数组

数组是固定长度内存的同类型序列。不同长度的数组被认为是不同的不兼容类型。
与 C 语言不同，Go 在创建数组时，数组元素被初始化为零值，因此，不需要明确地这样做。
同样，与 C 语言不同，Go 的数组是一种值类型。它不是一个指向内存块中第一个元素的指针。
如果把一个数组传入一个函数，整个数组将被复制。当然，仍然可以传递一个指向数组的指针来避免它被复制。

<p align="center">
    <img  width="70%" src="../static/images/Picture1.png" alt="Picture1.png"></img>
</p>

## 切片

切片是数组段的描述符。它是一个非常有用的数据结构，但可能有点不寻常。
有几种方法可以让你在使用它的时候踩坑，但如果你知道切片的内部工作原理，就可以避免踩这些坑。
下面是 Go 源代码中关于切片的实际定义：

```Golang
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

<p align="center">
    <img  width="70%" src="../static/images/Picture2.png" alt="Picture2.png"></img>
</p>

切片的定义十分有趣。切片本身是一个值类型，但是它用一个指针引用它所使用的数组。
与数组不同的是，如果你把一个切片传递给一个函数，你会得到一个数组指针、长度和容量属性的拷贝（上图中的第一个块），但数组本身的数据不会被复制。
两个切片的副本都会指向同一个数组。当你“切割”一个分片时，也会发生同样的事情。进行切割时，会创建一个新的切片，它仍然指向同一个数组：

```Golang
package main

import "fmt"

func f1(s []int) {
    // 进行切割时，会产生一个新的切片
    // 但不复制数组数据
    s = s[2:4]
    // 修改子切片
    // 也改变了主函数中切片的数组
    for i := range s {
        s[i] += 10
    }
    fmt.Println("f1", s, len(s), cap(s))
}

func main() {
    s := []int{1, 2, 3, 4, 5}
    // 将一个切片作为参数传递
    // 复制切片的属性（指针、长度和容量）
    // 但该副本共享相同的数组
    f1(s)
    fmt.Println("main", s, len(s), cap(s))
}
```

输出：

```Golang
f1   [13 14] 2 3
main [1 2 13 14 5] 5 5
```

<p align="center">
    <img  width="70%" src="../static/images/Picture3.png" alt="Picture3.png"></img>
</p>

如果你不知道切片是什么，你可能会认为它是一个值类型，并对 f1 “破坏”了主函数中切片的数据而感到惊讶。

## 获得一个带有数据的切片副本

为了得到一个带有数据的切片副本，你需要做一些工作。你可以手动复制元素到一个新的切片，或者使用复制或追加函数：

```Golang
package main

import "fmt"

func f1(s []int) {
    s = s[2:4]
    s2 := make([]int, len(s))
    copy(s2, s)

    // 或者如果你喜欢一个更简洁，但效率较低的版本：
    // s2 := append([]int{}, s[2:4]...)
    for i := range s2 {
        s2[i] += 10
    }
    fmt.Println("f1", s2, len(s2), cap(s2))
}

func main() {
    s := []int{1, 2, 3, 4, 5}
    f1(s)
    fmt.Println("main", s, len(s), cap(s))
}
```

输出：

```Golang
f1   [13 14] 2 3
main [1 2 3 4 5] 5 5
```

## 用 append 扩容切片

切片的所有副本都共享同一个数组，因此，如果对切片的捣乱，会对指针、长度和容量产生影响。除非他们不共享。
切片最有用的特性是它可以管理数组的扩容。当它需要扩容到超过现有数组的容量时，需要分配一个全新的数组。
如果你希望两份切片副本共享数组数据，这也可能是一个坑：

```Golang
package main

import "fmt"

func main() {
    // 做一个长度为 3、容量为 4 的切片
    s := make([]int, 3, 4)
    
    // 初始化为 1,2,3
    s[0] = 1
    s[1] = 2
    s[2] = 3
    
    // 数组的容量是 4
    // 在初始数组中增加一个适合的数字
    s2 := append(s, 4)
    
    // 修改数组中的元素
    // s 和 s2 仍然共享同一个数组
    for i := range s2 {
        s2[i] += 10
    }
    fmt.Println(s, len(s), cap(s))    // [11 12 13] 3 4
    fmt.Println(s2, len(s2), cap(s2)) // [11 12 13 14] 4 4
    
    // 这种扩容会使数组的容量增加，超过它的容量
    // 必须为 s3 分配新的数组
    s3 := append(s2, 5)
    
    // 修改数组中的元素以查看结果
    for i := range s3 {
        s3[i] += 10
    }
    fmt.Println(s, len(s), cap(s)) // 依然是旧的数组 [11 12 13] 3 4
    fmt.Println(s2, len(s2), cap(s2)) // 旧数组 [11 12 13 14] 4 4
    
    // 数组在最后一次扩容时被复制 [21 22 23 24 15] 5 8
    fmt.Println(s3, len(s3), cap(s3))
}
```

<p align="center">
    <img  width="70%" src="../static/images/Picture4.png" alt="Picture4.png"></img>
</p>

## nil 切片

不需要检查切片是否为 nil，也不必将其初始化。因为，诸如 len、cap 和 append 等函数在一个 nil 切片上是可以正常工作：

```Golang
package main

import "fmt"

func main() {
    var s []int // nil 数组
    fmt.Println(s, len(s), cap(s)) // [] 0 0
    s = append(s, 1)
    fmt.Println(s, len(s), cap(s)) // [1] 1 1
}
```

空切片与 nil 切片不是一回事：

```Golang
package main

import "fmt"

func main() {
    var s []int // 这是一个 nil 切片
    s2 := []int{} // 这是一个空切片
    
    // 在这里看起来是一回事:
    fmt.Println(s, len(s), cap(s)) // [] 0 0
    fmt.Println(s2, len(s2), cap(s2)) // [] 0 0
    
    // 但 s2 实际上被分配到了某个地方
    fmt.Printf("%p %p", s, s2) // 0x0 0x65ca90
}
```

如果你非常关心性能、内存使用等问题，初始化空分片可能不如使用 nil 切片来得理想。

## make 的陷阱

你可以使用 make 创建一个新切片，参数是切片的初始化类型、长度和容量。其中，容量参数是可选的：

```Golang
func make([]T, len, cap) []T
```

这样做似乎有点太容易了：

```Golang
package main

import (
    "fmt"
)

func main() {
    s := make([]int, 3)
    s = append(s, 1)
    s = append(s, 2)
    s = append(s, 3)
    fmt.Println(s)
}
```

输出：

```Golang
[0 0 0 1 2 3]
```

“不，这绝不会发生在我身上。我知道，对切片的第二个论据是长度，而不是容量...”我仿佛听到你这样说。

## 未使用的切片数组数据

因为切割数组时，会创建一个新切片，但它们共享底层数组，所以有可能在内存中保留更多的数据，而这可能正是你想要或期望的。这里有一个愚蠢的例子：

```Golang
package main

import (
    "bytes"
    "fmt"
    "io/ioutil"
    "os"
)

func getExecutableFormat() []byte {
    // 将我们自己的可执行文件读入内存
    bytes, err := ioutil.ReadFile(os.Args[0])
    if err != nil {
        panic(err)
    }
    return bytes[:4]
}

func main() {
    format := getExecutableFormat()
    if bytes.HasPrefix(format, []byte("ELF")) {
        fmt.Println("linux executable")
    } else if bytes.HasPrefix(format, []byte("MZ")) {
        fmt.Println("windows executable")
    }
}
```

在上面的代码中，只要那个格式变量在范围内，且没被垃圾回收，那么整个可执行文件（可能是几兆字节的数据）就会被保留在内存中。
为了解决这个问题，应该复制实际需要的字节。

## 多维切片

在 Go 中，目前还没有这样的东西。也许有一天会有，但目前为止，
你要么需要通过自己计算元素索引来手动将单维切片用作多维切片，
要么使用 "锯齿状 "切片（锯齿状切片是切片的切片）：

```Golang
package main

import "fmt"

func main() {
    x := 2
    y := 3
    s := make([][]int, y)
    for i := range s {
        s[i] = make([]int, x)
    }
    fmt.Println(s)
}
```

输出：

```Golang
[[0 0] [0 0] [0 0]]
```
