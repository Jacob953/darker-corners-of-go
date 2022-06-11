---
weight: 10
title: "第 10 章 switch & select 语句"
draft: false
---

# 第 10 章 switch & select 语句

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## case 语句默认为断句

与 C 语言的 case 语句不同，Go 中的 case 语句默认为中断。要使 case 语句通过，请使用 fallthrough 关键字：

```Golang
package main

import (
    "fmt"
    "time"
)

func main() {
    // 这样无法运行，如果是 Saturday 情况，那什么都不会被打印
    switch time.Now().Weekday() {
    case 6: // 这种情况下，不会做任何事情，并跳出 switch
    case 7:
        fmt.Println("weekend")
    }

    switch time.Now().Weekday() {
    case 1:
        break // 这个 break 没有任何作用，因为无论如何都会跳出
    case 2:
        fmt.Println("weekend")
    }

    // fallthrough 关键字将使 Saturday 同样也打印 weekend
    switch time.Now().Weekday() {
    case 6:
        fallthrough
    case 7:
        fmt.Println("weekend")
    }

    // case 也可以有多个值
    switch time.Now().Weekday() {
    case 6, 7:
        fmt.Println("weekend")
    }

    // 条件性中断仍然是可用的
    switch time.Now().Weekday() {
    case 6, 7:
        day := time.Now().Format("01-02")
        if day == "12-25" || day == "12-26" {
            fmt.Println("Christmas weekend")
            break // 不会打印 "weekend"
        }
        // 一个正常的 weekend
        fmt.Println("weekend")
    }
}
```

## 带标签的断点

正如之前在循环的章节中提到的，switch 和 select 也可以利用带标记的 break 来跳出外循环，而不是 switch 或 select 语句本身：

```Golang
package main

import (
    "fmt"
    "strings"
)

func main() {
    s := "The quick brown Waldo fox jumps over the lazy dog"
findWaldoLoop:
    for _, w := range strings.Split(s, " ") {
        switch w {
        case "Waldo":
            fmt.Println("found Waldo!")
            break findWaldoLoop
        default:
            fmt.Println(w, "is not Waldo")
        }
    }
}
```

输出：

```Golang
The is not Waldo
quick is not Waldo
brown is not Waldo
found Waldo!
```
