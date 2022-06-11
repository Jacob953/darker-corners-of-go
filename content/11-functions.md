---
weight: 11
title: "第 11 章 函数"
draft: false
---

# 第 11 章 函数

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## defer 语句

defer 似乎没有大坑，但有一些细微差别是值得一提的。

关于这个问题，有一篇来自 Andrew Gerrand 的 [好文章](https://go.dev/blog/defer-panic-and-recover)：

> defer 语句会将其包涵的递延函数推到一个调用列表上，
> 保存的调用列表将在原函数返回后被执行。
> 因此，defer 通常被用来简化执行各种清理动作的函数。

但是，有几点是需要特别注意的：

1. **虽然递延函数在原函数返回时才会被执行，但其参数也会在调用 defer 时被使用**：

```Golang
package main

import (
    "fmt"
)

func main() {
    s := "defer"
    defer fmt.Println(s)
    s = "original"
    fmt.Println(s)
}
```

输出：

```Golang
original
defer
```

2. **一旦原函数返回，递延函数就按后进先出的顺序执行**：

```Golang
package main

import (
    "fmt"
)

func main() {
    defer fmt.Println("one")
    defer fmt.Println("two")
    defer fmt.Println("three")
}
```

输出：

```Golang
three
two
one
```

3. **递延函数可以访问和修改函数中已命名的参数**：

```Golang
package main

import (
    "fmt"
    "time"
)

func timeNow() (t string) {
    defer func() {
      t = "Current time is: " + t
    }()
    return time.Now().Format(time.Stamp)
}

func main() {
    fmt.Println(timeNow())
}
```

输出：

```Golang
Current time is: Feb 13 13:36:44
```

4. **defer 对代码块不起作用，只对整个函数起作用**

与变量声明不同，defer 语句不属于代码块的范围：

```Golang
package main

import (
    "fmt"
)

func main() {
    for i := 0; i < 9; i++ {
        if i%3 == 0 {
            defer func(i int) {
                fmt.Println("defer", i)
            }(i)
          }
    }
    fmt.Println("exiting main")
}
```

输出：

```
exiting main
defer 6
defer 3
defer 0
```

在这个例子中，当 i 为 0、3 和 6 时，递延函数将被添加到调用列表中。但它只有在主函数退出时才会被调用，而不是在 if 语句结束时。

5. **recover() 函数只在递延函数中起作用，在原函数中不会做任何事情**

在 Go 中，如果你想找一个与 try-catch 语句相当的语句，可以肯定这是没有的。想要捕获 panic 的内容，需要在递延函数中使用 recover()：

```Golang
package main

import (
    "fmt"
)

func panickyFunc() {
    panic("panic!")
}

func main() {
    defer func() {
      r := recover()
      if r != nil {
        fmt.Println("recovered", r)
      }
    }()
    panickyFunc()
    fmt.Println("this will never be printed")
}
```

输出：

```
recovered panic!
```
