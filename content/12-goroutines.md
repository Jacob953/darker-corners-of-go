---
weight: 12
title: "第 12 章 协程"
draft: false
---

# 第 12 章 协程

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 什么是 goroutine

通常情况下，goroutine 可以被认为是轻量级线程。goroutine 的启动非常快速，因为仅需要使用 2kb 内存就可以初始化堆栈（当然，也可以改动）。
goroutine 由 Go 的运行时系统管理（而不是操作系统），因此，在上下文切换的代价很低。
goroutine 就是为并发而生的，即使在多个硬件的线程上运行，它们也可以并行。

> 并发是指同时处理很多事情；并行是指同时做很多事情。
> 
> Rob Pike

goroutine 的效率是非常之高的，如果与 channel 相结合，它们很可能是 Go 的最佳功能。
尽管 goroutine 在 Go 中是无处不在的，但也存在一个极端但很好的例子。
当一个服务器管理大量并发的 websocket 连接时，goroutine 需要分别被单独管理，但更多可能是处于闲置状态的（不占用很多 CPU 或内存）。
为每个连接都创建一个线程，一旦连接数变得数以千计就会出现问题，然而，在使用 goroutine 时，产生数十万的连接也是可能的。

关于 goroutine 是如何工作的，可以 [在这里](https://medium.com/technofunnel/understanding-golang-and-goroutines-72ac3c9a014d) 找到更详细的帖子。

## 运行 goroutine 并不能阻止程序退出

在 Go 中，当主函数退出时，程序也就退出了。此时，任何在后台运行的 goroutine 都会安静地停止。
下面的程序将退出，且不打印任何东西：

```Golang
package main

import (
    "fmt"
    "time"
)

func goroutine1() {
    time.Sleep(time.Second)
    fmt.Println("goroutine1")
}

func goroutine2() {
    time.Sleep(time.Second)
    fmt.Println("goroutine2")
}

func main() {
    go goroutine1()
    go goroutine2()
}
```

为了确保这些 goroutine 可以完成，需要添加一些同步，例如使用通道或 sync.WaitGroup：

```Golang
package main

import (
    "fmt"
    "sync"
    "time"
)

func goroutine1(wg *sync.WaitGroup) {
    time.Sleep(time.Second)
    fmt.Println("goroutine1")
    wg.Done()
}

func goroutine2(wg *sync.WaitGroup) {
    time.Sleep(time.Second)
    fmt.Println("goroutine2")
    wg.Done()
}

func main() {
    wg := &sync.WaitGroup{}
    wg.Add(2)
    go goroutine1(wg)
    go goroutine2(wg)
    wg.Wait()
}
```

输出：（该结果可能会颠倒）

```
goroutine2
goroutine1
```

## goroutine 报警会使整个程序崩溃

在 goroutine 内部发生的报警时，必须用 defer 和 recover() 来处理。否则，整个应用程序将崩溃：

```Golang
package main

import (
    "fmt"
    "time"
)

func goroutine1() {
    panic("something went wrong")
}

func main() {
    go goroutine1()
    time.Sleep(time.Second)
    fmt.Println("will never get here")
}
```


```
panic: something went wrong 

goroutine 18 [running]:
main.goroutine1()
        c:/projects/test/main.go:9 +0x45
created by main.main
        c:/projects/test/main.go:13 +0x45
```