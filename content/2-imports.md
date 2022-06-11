---
weight: 2
title: "第 2 章 包导入"
draft: false
---

# 第 2 章 包导入

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 未使用的导入包

带有未使用导入包的 Go 程序是无法进行编译的。这是该语言的特点，因为导入包会降低编译器的速度。
在大型程序中，导入未使用的包会对编译时间产生重大影响。

为了在开发过程中使编译器正常运行，你可以以下方式导入未使用的包：

```Golang
package main
import (
    "fmt"
    "math"
)

// 参照导入未使用的包
var _ = math.Round

func main() {
    fmt.Println("Hello")
}
```

## goimports

更好的解决方案是使用 goimports 工具，它可以删除未引用的导入包。更棒的是，它试图自动找到并添加缺失的包：

```Golang
package main
import "math" 

// 导入而未使用："math"
func main() {
    fmt.Println("Hello") // 未定义：fmt
}
```

**运行 goimports：**

```shell
./goimports main.go            
```

```Golang
package main
import "fmt"

func main() {
    fmt.Println("Hello")
}
```

大多数流行的 IDE 的 Go 插件在保存源文件时，会自动运行 goimports。

## 下划线导入

以下划线的方式导入包只是为了包的副作用。以这种方式导入包，程序会创建包级变量并运行包的 [init 函数](https://medium.com/golangspec/init-functions-in-go-eac191b3860a)：

```Golang
package package1

func package1Function() int {
    fmt.Println("Package 1 side-effect")
    return 1
}

var globalVariable = package1Function()

func init() {
    fmt.Println("Package 1 init side effect")
}

```

**在 package2 中：**

```Golang
package package2
import _ package1
```

这将打印出信息并初始化 globalVariable：

```shell
Package 1 side-effect
Package 1 init side effect
```

多次导入一个包（e.g. 在 main 包及其引用的其他包中），只运行一次 init 函数。

下划线导入在 Go 运行时库中使用。例如，导入 net/http/pprof 会调用它的 init 函数，以暴露可以提供调试信息的 HTTP 端点：

```Golang
import _ "net/http/pprof"
```

## 点导入

点导入允许在没有限定词的情况下，访问导入的包中的标识符：

```Golang
package main
import (
    "fmt"
    . "math"
)
func main() {
    fmt.Println(Sin(3)) // 引用 math.Sin
}
```

> 关于点导入是否应该从语言中删除，有一个公开的辩论。Go 团队不建议在除了在测试包之外的地方使用它们。
> 这使得程序更难阅读，因为不清楚像 Quux 这样的名字是在当前包中还是在导入的包中的顶级标识符。
> 
> https://golang.org/doc/faq

另外，如果你在使用go-lint工具，那么，在测试文件之外使用点导入时，它会显示一个警告，而且你无法轻易关闭它。

Go 团队推荐的一种使用情况是在测试中，由于循环依赖，不能成为被测包的一部分。依赖关系。

```Golang
// foo_test 包测试 foo 包
package foo_test
import (
    "bar/testutil" // 也导入了 "foo"
    . "foo"
)
```

这个测试文件不是 foo 包的一部分，因为它引用了 bar/testutil，而 bar/testutil 又引用了 foo。这将产生一个循环的依赖关系。

在这种情况下，首先要考虑的是，也许有一个更好的方法来结构这些包，以避免循环依赖性。
将 bar/testutil 使用的包从 foo 移到 foo 和 bar/testutil 都可以导入的第三个包中，可能有意义，也可能没有意义，这样就能在 foo 包中正常地编写测试。

如果重构没有意义，并且测试以点导入的方式被引用到独立包中，foo_test 包至少可以假装是 foo 包的一部分。但要注意的是，它不能访问 foo 包中未导出的类型和函数。

可以说，在特定领域的语言中，点导入有一个很好的用例。 例如，Goa 框架将其用于配置。如果没有点导入，它看起来就不是很好：

```Golang
package design
import . "goa.design/goa/v3/dsl"

// API 描述了 API 服务器的全局属性。
var _ = API("calc", func() {
    Title("Calculator Service")
    Description("HTTP service for adding numbers, a goa teaser")
    Server("calc", func() {
        Host("localhost", func() { URI("http://localhost:8088") })
    })
})
```
