# Go 鲜为人知的角落｜第 13 章 接口｜《Darker Corners of Go》独译（已授权）

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## 检查接口变量是否为 nil

在 Go 中，接口当然是最常见的坑之一。但与其他语言不同，Go 的接口不只是一个指向内存位置的指针。

一个接口类型的结构有：

- 静态类型（接口本身的类型）
- 动态类型
- 值

> 如果一个变量为接口类型，当它的动态类型和值都为 nil 时，那它就等于 nil。


```Golang
package main

import (
    "fmt"
)

type ISayHi interface {
    Say()
}

type SayHi struct{}

func (s *SayHi) Say() {
    fmt.Println("Hi!")
}

func main() {
    // 这里，变量 "sayer" 只具有静态类型的 ISayHi。
    // 动态类型和值都是 nil
    var sayer ISayHi
    
    // 果然，Sayer 等于 nil
    fmt.Println(sayer == nil) // true
    
    // 一个具体类型，但值为 nil 的变量
    var sayerImplementation *SayHi
    
    // 接口变量的动态类型现在是 SayHi
    // 接口指向的实际值仍然为 nil
    sayer = sayerImplementation
    
    // sayer 不再等于 nil，因为它的动态类型已经被设置
    // 即使它所指向的值为 nil
    // 这里并不是大多数人所期望的那样
    fmt.Println(sayer == nil) // false
}
```

接口值被设置为一个 nil 结构体时，不能用来做任何事情，那么为什么它不等于nil呢？
与其他语言相比，这便是 Go 的另一个不同之处。在 C# 中，对一个 nil 类调用方法会抛出一个异常，但在 Go 中，无论怎样，这都是允许的。
因此，当接口设置了动态类型时，即使值为 nil，有时也可以使用。所以，你可以说接口并不是真的是 "nil"：

```Golang
package main

import (
    "fmt"
)

type ISayHi interface {
    Say()
}

type SayHi struct{}

func (s *SayHi) Say() {
    // 这个函数并没有访问 s
    // 即使 s 为 nil，这也会正常执行
    fmt.Println("Hi!")
}

func main() {
    var sayer ISayHi
    var sayerImplementation *SayHi
    sayer = sayerImplementation

    // 在 sayer 接口中，SayHi 的值为 nil
    // 在 Go 中，可以对一个 nil 结构体调用方法。
    // 这行可以正常运行，因为 Say 函数并没有访问s
    sayer.Say()
}
```

尽管这可能很奇怪，但没有简单的方法可以用来检查一个接口指向的值是否为 nil。
关于这个问题，有一个长期的讨论，但似乎并没有什么进展。所以在近期，你可以做这些事情：

### 弊端最少的选择 #1：永远不要给接口分配值为 nil 的具体类型

如果你从不给接口变量分配值为 nil 的具体类型（除非是那些被设计为与 nil 接收器一起工作的类型），那么简单的 "==nil" 判断将总是有效。
例如，永远不要这样做：

```Golang
func MyFunc() ISayHi {
    var result *SayHi
    if time.Now().Weekday() == time.Sunday {
      result = &SayHi{}
    }

    // 如果不是 Sunday，则返回一个不等于 nil 的接口
    // 但其具体类型的值为 nil
    // (MyFunc() == nil 将是 false)
    return result
}
```

而应该返回一个实际的 nil：

```Golang
func MyBetterFunc() ISayHi {
    if time.Now().Weekday() != time.Sunday {
        // 如果不是 Sunday
        // MyBetterFunc() == nil 将是 true
        return nil
    }
    return &SayHi{}
}
```

即使这并不理想，但它可能是现有的最好的解决方案，因为那时每个人都必须意识到它的存在，然后，并在代码审计等方面进行监控。
在某种程度上，这是在做计算机可以完成的工作。

### 特殊情况下的选择 #2：反射

如果必要，你可以通过反射来检查一个接口的底层值是否为 nil。
这可能不是一个好主意，如果你的代码总是调用这些函数，程序会变得很慢：

```Golang
func IsInterfaceNil(i interface{}) bool {
    if i == nil {
      return false
    }
    rvalue := reflect.ValueOf(i)
    return rvalue.Kind() == reflect.Ptr && rvalue.IsNil()
}
```

检查 Kind() 的值是否为指针是必要的，因为如果类型为 nil（如简单的 int），IsNil 会报警。

### 请不要做这个选择 #3：在你的结构接口中添加 IsNil

这样做，你可以在不使用反射的情况下检查一个接口是否为 nil：

```Golang
type ISayHi interface {
    Say()
    IsNil() bool
}

type SayHi struct{}

func (s *SayHi) Say() {
    fmt.Println("Hi!")
}

func (s *SayHi) IsNil() bool {
    return s == nil
}
```

### 也许应该考虑 #1 和 #4：断言具体类型

如果你知道接口值应该是什么类型，你可以通过类型转换或类型断言，这样，先得到一个具体类型的值，再来检查它是否为nil：

```Golang
func main() {
    v := MyFunc()
    fmt.Println(v.(*SayHi) == nil)
}
```

如果你真的知道自己在做什么，这也许是好的，但在很多情况下，这就违背了使用接口的初衷。
考虑一下，当 ISayHi 的新实现被添加进来时会发生什么。你是否需要记得去寻找这段代码，并为新结构体添加另一个检查？你会对每个新的实现都这样做吗？
如果这段代码是在处理一个很少发生的事件，且没有对新添加的实现进行检查，而是在代码进入生产后很久才注意到的，那该怎么办？

## 接口是隐性满足的

与许多其他语言不同，你不需要明确说明一个结构实现了一个接口。编译器可以自己解决这个问题。这有很大的意义，而且在实践中非常方便：

```Golang
package main

import (
    "fmt"
)

// 一个接口
type ISayHi interface {
    Say()
}

// 这个结构体实现了 ISayHi，即使不知道存在 ISayHi
type SayHi struct{}
func (s *SayHi) Say() {
    fmt.Println("Hi!")
}

func main() {
    var sayer ISayHi // sayer 是一个 interface
    sayer = &SayHi{} // SayHi 隐式地实现了 ISayHi
    sayer.Say()
}
```

有时，让编译器检查一个结构是否实现了一个接口是很有用的：

```Golang
// 在编译时验证 *SayHi 是否实现了 ISayHi
var _ ISayHi = (*SayHi)(nil)
```

## 对错误的类型进行类型断言

类型断言有单变量和双变量版本。当类型不是被断言的类型时，单变量版本会报警：

```Golang
func main() {
      var sayer ISayHi
      sayer = &SayHi{}

      // t 是一个类型为 *SayHi2 的零值（本例中为 nil）
      // ok 将会是 false
      t, ok := sayer.(*SayHi2)
      if ok {
          t.Say()
      }

      // 警告: 接口转换:
      // main.ISayHi 是 *main.SayHi，而不是 *main.SayHi2
      t2 := sayer.(*SayHi2)
      t2.Say()
}
```