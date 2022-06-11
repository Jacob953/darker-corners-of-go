# Go 鲜为人知的角落｜第 9 章 循环｜《Darker Corners of Go》独译（已授权）

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## range 迭代器会返回两个值

由于 for-range 的工作方式与其他语言不尽相同，对于 Go 的初学者来说，这很可能是一个坑。
for-range 会返回一个或两个变量，第一个是迭代索引（如果是迭代哈希，则是哈希键），第二个是值。如果只使用一个变量——那它就是索引：

```Golang
package main

import "fmt"

func main() {
    slice := []string{"one", "two", "three"}
    for v := range slice {
        fmt.Println(v) // 0, 1, 2
    }
    for _, v := range slice {
        fmt.Println(v) // one two three
    }
}
```

## for 循环会重复使用迭代器变量

在循环中，每次迭代都会重复使用同一个迭代器变量。如果你读取它的地址，那每次都是同一个地址。
这意味着在每次迭代中，迭代器变量的值都会被复制到同一个内存位置。这使得循环更有效率，但也是 Go 最常见的陷阱之一。
下面是 Go wiki 中的一个例子：

```Golang
package main

import "fmt"

func main() {
    var out []*int
    for i := 0; i < 3; i++ {
        out = append(out, &i)
    }
    fmt.Println("Values:", *out[0], *out[1], *out[2])
    fmt.Println("Addresses:", out[0], out[1], out[2])
}
```

输出：

```Golang
Values: 3 3 3
Addresses: 0xc0000120e0 0xc0000120e0 0xc0000120e0
```

惊讶吧，不过有一个解决方案，是在循环中声明一个新的变量。在代码块中声明的变量不会被重复使用，即使在循环中也是如此：

```Golang
package main

import "fmt"

func main() {
    var out []*int
    for i := 0; i < 3; i++ {
        i := i // 将 i 复制到一个新的变量中
        out = append(out, &i)
    }
    fmt.Println("Values:", *out[0], *out[1], *out[2])
    fmt.Println("Addresses:", out[0], out[1], out[2])
}
```

现在，它就可以像预期的那样运行：

```Golang
Values: 0 1 2
Addresses: 0xc0000120e0 0xc0000120e8 0xc0000120f0
```

如果是 for-range 子句，则会重复使用索引变量和值变量。

这与在一个循环中启动 goroutine 的情况类似：

```Golang
package main

import (
    "fmt"
    "time"
)

func main() {
    for i := 0; i < 3; i++ {
        go func() {
            fmt.Print(i)
        }()
    }
    time.Sleep(time.Second)
}
```

输出：

```Golang
333
```

这些 goroutine 是在这个循环中创建的，但它们需要一点时间来开始运行。
由于它们捕获的是单个 i 变量，因此，Println 会打印 goroutine 执行时的任何值。

在这种情况下，可以像之前的例子那样，在代码块内创建一个新的变量，或者把迭代器变量作为参数传递给 goroutine：

```Golang
package main

import (
    "fmt"
    "time"
)

func main() {
    for i := 0; i < 3; i++ {
        go func(i int) {
            fmt.Print(i)
        }(i)
    }
    time.Sleep(time.Second)
}
```

输出：

```Golang
012
```

这里，goroutine 的参数 i 是一个新变量，它作为创建 goroutine 时的一部分，从迭代器变量中复制过来的。

如果不启动 goroutine，而是在循环中调用一个简单的函数，代码就会像预期的那样工作：

```Golang
for i := 0; i < 3; i++ {
    func() {
        fmt.Print(i)
    }()
}
```

输出：

```Golang
012
```

变量 i 像以前一样被重复使用。然而，直到函数执行完毕，并不是每个函数调用都会让循环继续。在那段时间里，变量 i 会有预期值。

这就变得有点棘手了。请看这个例子，在结构体上调用方法：

```Golang
package main

import (
    "fmt"
    "time"
)

type myStruct struct {
    v int
}

func (s *myStruct) myMethod() {
    // 打印 myStruct 的值和它的地址
    fmt.Printf("%v, %p\n", s.v, s)
}

func main() {
    byValue := []myStruct{{1}, {2}, {3}}
    byReference := []*myStruct{{1}, {2}, {3}}

    fmt.Println("By value")
    for _, i := range byValue {
        go i.myMethod()
    }
    time.Sleep(time.Millisecond * 100)

    fmt.Println("By reference")
    for _, i := range byReference {
        go i.myMethod()
    }
    time.Sleep(time.Millisecond * 100)
}
```

输出：

```Golang
By value
3, 0xc000012120
3, 0xc000012120
3, 0xc000012120
By reference
1, 0xc0000120e0
3, 0xc0000120f0
2, 0xc0000120e8
```

再次惊讶吧！当 myStruct 采用引用类型时，它的运行起来就像一开始就没有陷阱一样！
这与 goroutine 的创建方式有关，在 goroutine 被创建时，goroutine 的参数会被评估。
方法接收器（myMethod 的 myStruct）实际上是一个参数。

当通过值类型调用时：由于 myMethod 的参数 s 是一个指针，i 的地址被作为参数传给 goroutine。
正如我们所知，迭代器变量是重复使用的，所以每次都是同一个地址。
当迭代器运行时，它将复制一个新的 myStruct 值到 i 变量的同一地址。打印的值是在 goroutine 执行时 i 变量的值。

当通过引用类型调用时：参数已经是一个指针，所以在创建 goroutine 时，它的值被推到新 goroutine 的堆栈中。这恰好是我们想要的地址，这样预期值就被打印出来了。

## 带标签的 break 和 continue

在 Go 中，也许还有一些不太为人所知的特点，比如能够给 for、switch 和 select 语句打上标签，并在这些标签上使用 break 和 continue。
下面是如何跳出外循环：

```Golang
loopi:
    for x := 0; x < 3; x++ {
        for y := 0; y < 3; y++ {
            fmt.Printf(x, y)
            break loopi
        }
    }
```

输出：

```Golang
0 0
```

continue 也可以以类似的方式使用：

```Golang
loopi:
    for x := 0; x < 3; x++ {
        for y := 0; y < 3; y++ {
            fmt.Printf(x, y)
            continue loopi
        }
    }
```

输出：

```Golang
0 0
1 0
2 0
```

标签也可以与 switch 和 select 语句一起使用。在这里，没有标签的 break 只会跳出 select 语句，进入 for 循环：

```Golang
package main

import (
    "fmt"
    "time"
)

func main() {
loop:
    for {
        select {
        case <-time.After(time.Second):
            fmt.Println("timeout reached")
            break loop
        }
    }
    fmt.Println("the end")
}
```

输出：

```Golang
timeout reached
the end
```

如前所述，switch 和 select 语句也可以被标记，所以我们可以把上面的例子转过来：

```Golang
package main

import (
    "fmt"
    "time"
)

func main() {
myswitch:
    switch {
    case true:
        for {
            fmt.Println("switch")
            break myswitch // 在这种情况下，就不必执行 "continue" 了
        }
    }
    fmt.Println("the end")
}
```

在前面的例子中，我们很容易将 “label 语句” 与 goto 所使用的标签混淆。实际上，你可以为 break/continue 和 goto 使用同一个标签，但行为会有所不同。
在下面的代码中，break 会跳出一个有标签的循环，而 goto 会将执行转移到标签的位置（并在下面代码中，导致无限循环）：

```Golang
package main

import (
    "fmt"
    "time"
)

func main() {
loop:
    switch {
    case true:
        for {
            fmt.Println("switch")
            break loop // 跳出 “label 语句”
        }
    }
    fmt.Println("not the end")
    goto loop // 跳入带有 “loop” 标签的语句
}
```

输出：

```
switch
not the end
switch
not the end
...
```
