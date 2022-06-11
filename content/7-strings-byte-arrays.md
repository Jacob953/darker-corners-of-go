# Go 鲜为人知的角落｜第 7 章 字符串 & 字节数组｜《Darker Corners of Go》独译（已授权）

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## Go 的字符串

在 Go 中，字符串的定义是这样的：

```Golang
type StringHeader struct {
    Data uintptr
    Len  int
}
```

字符串本身是值类型，包含一个指向字节数组的指针和一个固定的长度。与 C 语言不同，Go 字符串中的零字节并不标志着字符串的结束。
字符串里面可以包含任何数据。通常情况下，这些数据被编码为 UTF-8 字符串，但它不一定是这样。

## 字符串不能为 nil

在 Go 中，字符串永远不会为 nil。字符串的默认值是一个空字符串，而不是 nil：

```Golang
package main

import "fmt"

func main() {
    var s string
    fmt.Println(s == "") // 为真
    s = nil // 错误: 不能在赋值中使用 nil 作为字符串类型
}
```

## 字符串是不可变的（某种程度上）

Go 并不希望你修改字符串：

```Golang
package main

func main() {
    str := "darkercorners"
    str[0] = 'D' // 错误: 不能分配给 str[0]
}
```

不可变数据更易于推理，因此产生的问题更少。但缺点是，每次你想从一个字符串中添加或删除某些内容时，都必须分配一个全新的字符串。
如果你真的希望做些更改，可以通过 unsafe 包来修改字符串，但如果你真打算采用这种方式，可能就聪明过头了。

最常见情况是，当许多字符串需要加在一起时，你可能要担心分配的问题。
有一个 strings.Builder 类型用于解决这个问题，它在添加字符串时是批量分配内存，而不是每次都分配内存。

```Golang
package main

import (
    "strconv"
    "strings"
    "testing"
)

func BenchmarkString(b *testing.B) {
    var str string
    for i := 0; i < b.N; i++ {
        str += strconv.Itoa(i)
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    var str strings.Builder
    for i := 0; i < b.N; i++ {
        str.WriteString(strconv.Itoa(i))
    }
}
```

输出：

```Golang
BenchmarkString-8 401053 147346 ns/op 1108686 B/op 2 allocs/op
BenchmarkStringBuilder-8 29307392 44.9 ns/op 52 B/op 0 allocs/op
```

在这个例子中，使用 strings.Builder 比简单的添加字符串（每次分配新的内存）要快3000倍。

在某些情况下，Go 编译器会优化掉这些分配：

1. 当把一个字符串和一个字节片相比较时：str == string(byteSlice)
2. 当 []byte 被用于查找 map[string] 中的条目时：m[string(byteSlice)]
3. 在字符串被转换为字节的 range 子句中：for i, v := range []byte(str) {...}

Go 编译器的新版本可能会增加更多的优化，所以如果性能很重要，最好使用基准测试和分析器。

## 字符串 vs 字节切片

修改字符串的一种方法是首先将其转换为字节切片，然后再转换回字符串。
如下面的例子所示，将一个字符串转换为字节切片，然后再复制整个字符串和字节片。
原字符串并没有改变：

```Golang
package main

import (
    "fmt"
)

func main() {
    str := "darkercorners"

    bytes := []byte(str)
    bytes[0] = 'D'

    str2 := string(bytes)
    bytes[6] = 'C'

    // 打印: darkercorners Darkercorners DarkerCorners
    fmt.Println(str, str2, string(bytes))
}
```

使用 unsafe 包，有可能（但显然是不安全的）直接修改字符串而不分配内存。

> 导入 unsafe 包带来的结果可能是不可移植的，并且不受 Go 1 兼容性准则的保护。
>
> https://golang.org/pkg/unsafe/

```Golang
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    buf := []byte("darkercorners")
    buf[0] = 'D'
    
    // 分配一个字符串，指向与 buf 字节切片相同的数据
    str := *(*string)(unsafe.Pointer(&buf))
    
    // 修改字节切片
    // 现在它指向的是与字符串相同的内存。
    // 这里也对 str 进行了修改
    buf[6] = 'C'
    
    // DarkerCorners DarkerCorners
    fmt.Println(str, string(buf))
}
```

## UTF-8 的那些事儿

Unicode 和 UTF-8 是个棘手的问题。要了解 Unicode 和 UTF-8 的一般工作原理，你可能想阅读 Joel Spolsky 的博客[《每个软件开发人员绝对必须知道的 Unicode 和字符集（没有借口！）》](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)。

做一个简短的回顾：

1. Unicode 是 “一种用于不同语言和文字的国际编码标准，每个字母、数字或符号都被分配了一个独特的数值，适用于不同的平台和程序”。本质上，它是一个“码点”的大表。它包含了所有语言的大部分（但不是全部）字符。该表中，每个码位是一个索引，有时你可以看到用 U+ 符号指定，如 U+0041 表示字母 A。
2. 通常，码位是指一个字符，例如汉字⻯（U+2EEF），但它也可以是一个几何形状或一个字符修饰符（例如德语 ä、ö 和 ü 等字母的音符）。出于某种原因，它甚至可以是一个便便图标（U+1F4A9）。
3. UTF-8 是将 Unicode 大表中的元素编码成计算机可以处理的实际字节的方法之一（也是最常见的一种）。
4. 当用 UTF-8 编码时，单个的 Unicode 代码点可能需要 1 到 4 个字节。
5. 数字和拉丁字母（a-z，A-Z，0-9）的编码为 1 个字节。许多其他语言的字母在 UTF-8 编码中需要 1 个以上的字节。
6. 如果你不知道第 5 条，一旦有人用其他语言使用你的 Go 程序，你的程序可能会崩溃。当然，除非你仔细阅读了本章的其他内容。

## Go 中的字符串编码

Go 中的字符串是一个字节数组。任何字节，字符串本身不在意如何编码，也不必采用 UTF-8 编码。尽管有些库函数甚至是一种语言特性（for-range 循环，下文将介绍）假设它采用 UTF-8 编码。

认为 Go 字符串都是 UTF-8 的情况并不少见。但字符串的字面量给这种混乱带来了很大的影响。虽然字符串本身没有任何特定的编码，但 Go 编译器总是将源代码解释为 UTF-8。

当字符串的字面量被定义后，编辑器会把它同其他的代码一样，保存为 UTF-8 编码的 Unicode 字符串。这就是 Go 解析后会被编译到程序中的内容。无论是编译器还是 Go 的字符串处理代码，都与字符串最终被编码为 UTF-8 无关，这只是文本编辑器将字符串写入磁盘的方式。

```Golang
package main

import (
    "fmt"
)

func main() {
    // 一个含有 Unicode 字符的字符串字面量
    s := "English 한국어"
    // 打印出预期的 Unicode 字符串: English 한국어
    fmt.Println(s)
}
```

为了证明这一点，你可以这样定义一个非UTF-8的字符串：

```Golang
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    s := "\xe2\x28\xa1"
    fmt.Println(utf8.ValidString(s)) // false
    fmt.Println(s) // �(�
}
```

## rune 类型

在Go中，Unicode 码点用 “rune” 类型表示，它是一个 32 位的整数。

## 字符串长度

对字符串调用 len 函数，会返回字符串中的字节数，而不是字符数。

获取字符数可能是相当复杂的。计算字符串中的 rune 可能足够好，也可能不够好：

```Golang
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    s := "한국어" // 3 个韩文字符，用 9 个字节编码
    byteLen := len(s)
    runeLen := utf8.RuneCountInString(s)
    runeLen2 := len([]rune(s)) // 做同 RuneCountInString 一样的事情
    fmt.Println(byteLen, runeLen, runeLen2) // 打印 9 3 3
}
```

不幸的是，有些 Unicode 字符跨越了多个码点，因此也有多个 rune。
正如 [Unicode Standard](http://unicode.org/reports/tr15/) 中所解释的那样，需要做一些可怕的事情，才能计算出 Unicode 字符串中人类所感知的字符数。
Go 库并没有真正提供一个简单的方法来做到这一点。这里提供了一种解决方法：

```Golang
package main

import (
    "fmt"
    "unicode/utf8"
    "golang.org/x/text/unicode/norm"
)

func normlen(s string) int {
    var ia norm.Iter
    ia.InitString(norm.NFKD, s)
    nc := 0
    for !ia.Done() {
        nc = nc + 1
        ia.Next()
    }
    return nc
}

func main() {
    str := "é́́" // 一个特别奇怪的字符串
    fmt.Printf(
        "%d bytes, %d runes, %d actual character",
        len(str),
        utf8.RuneCountInString(str),
        normlen(str))
}
```

输出：

```
7 bytes, 4 runes, 1 actual character
```

## 字符串索引操作符 vs for-range

简而言之，对于字符串索引操作符，返回该字符串的字节数组中索引的字节。
对于 for-range，在一个字符串中对 rune 进行迭代，将字符串解释为 UTF-8 编码的文本：

```Golang
package main

import (
    "fmt"
)

func main() {
    s := "touché"

    // 打印每个字节
    // touchÃ©
    for i := 0; i < len(s); i++ {
        fmt.Print(string(s[i]))
    }
    fmt.Println()

    // 打印每个 rune
    // touché
    for _, r := range s {
        fmt.Print(string(r))
    }
    fmt.Println()
    
    // 将一个字符串转换为 rune 切片，以便通过索引访问
    // touché
    r := []rune(s)
    for i := 0; i < len(r); i++ {
        fmt.Print(string(r[i]))
    }
}
```
