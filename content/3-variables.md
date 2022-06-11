---
weight: 3
title: "第 3 章 变量"
draft: false
---

# 第 3 章 变量

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 未使用的变量

含有未使用变量的 Go 程序无法编译：

> 存在未使用的变量，表明可能存在一个错误[...]，Go 拒绝编译带有未使用的变量或包导入，以短期的便利保证长期的构建速度和程序的清晰度。
> 
> https://golang.org/doc/faq

该规则的例外是全局变量和函数参数：

```Golang
package main
var unusedGlobal int // 合法的
func f1(unusedArg int) { // 未使用的函数声明也是可以的
    // 错误: 定义但未使用
    a, b := 1,2
    // 这里使用了 b ，但 a 只是分配给了，不算是 "使用"
    a = b 
}   
```

## 短变量声明

短变量声明只在函数中起作用：

```Golang
package main
v1 := 1 // 错误: 在函数体外不能有声明语句

var v2 = 2 // 合法的
func main() {
    v3 := 3 // 合法的
    fmt.Println(v3)
}
```

在设置结构体字段值时，它们也不起作用：

```Golang
package main

type myStruct struct {
    Field int
}

func main() {    
    var s myStruct

    // 错误: 非名称的 s.Field在 := 的左侧。
    s.Field, newVar := 1, 2
    
    var newVar int
    s.Field, newVar = 1, 2 // 这实际上是合法的
}
```

## 变量遮盖

令人遗憾的是，Go 中允许使用变量遮盖。这是你需要经常注意的事情，因为它可能导致难以发现的问题。
发生这种情况往往是图方便，在至少有一个变量是新的情况下，Go 允许使用短变量声明：

```Golang
package main
import "fmt"

func main() {
    v1 := 1
    // 在这里 v1 实际上没有被重新声明，只是被设置了一个新的值
    v1, v2 := 2, 3
    fmt.Println(v1, v2) // 打印 2, 3
}
```

然而，如果该声明是在另一个代码块内，它将声明一个新的变量，有可能导致严重的错误：

```Golang
package main
import "fmt"

func main() {
    v1 := 1
    if v1 == 1 {
        v1, v2 := 2, 3
        fmt.Println(v1, v2) // 打印 2, 3
    }
    fmt.Println(v1) // 打印 1 !
}
```

对于一个更现实的例子，我们假设你有一个返回错误的函数：

```Golang
package main
import (
    "errors"
    "fmt"
)

func func1() error {
   return nil
}

func errFunc1() (int, error) {
   return 1, errors.New("important error")
}

func returnsErr() error {
    err := func1()
    if err == nil {
        v1, err := errFunc1()
        if err != nil {
            fmt.Println(v1, err) // 打印: 1 important error
        }
    }
    return err // 返回 nil!
}
func main() {
    fmt.Println(returnsErr()) // 打印 nil
}
```

解决这个问题的办法很多，其中之一是不要在嵌套的代码块中使用短变量声明：

```Golang
func returnsErr() error {
    err := func1()
    var v1 int
    if err == nil {
        v1, err = errFunc1()
        if err != nil {
            fmt.Println(v1, err) // 打印: 1 important error
        }
    }
    return err // 返回 "important error"
}
```

或者，相对于上面的例子，更好的做法是提前退出：

```Golang
func returnsErr() error {
    err := func1()
    if err != nil {
        return err
    }
    v1, err := errFunc1()
    if err != nil {
        fmt.Println(v1, err) // 打印: 1 important error
        return err
    }
    return nil
}
```

也有一些工具可以帮助避免这个问题。在 go vet 工具中曾有实验性的变量遮盖检测，但它被删除了。
你可以输入如下命令来安装和运行该工具：

```shell
go get -u golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow
go vet -vettool=$(which shadow)
```

打印：

```shell
.\main.go:20:7: declaration of "err" shadows declaration at line 17
```