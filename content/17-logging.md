# Go 鲜为人知的角落｜第 17 章 日志｜《Darker Corners of Go》独译（已授权）

> 当前译本仍不稳定，如翻译有问题请及时联系 jacob953@csu.edu.cn。

## log.Fatal 和 log.Panic

在 Go 中，使用日志包进行日志记录时，log.Fatal 和log.Panic 这两个函数有一个陷阱在等着你。
与你期望的日志函数不同，这些函数不仅仅是用不同的日志级别记录一条消息，它们还终止了整个应用程序。
log.Fatal 干净地退出应用程序，log.Panic 调用运行时警告。下面是 Go 日志包中的实际函数：

```Golang
// Fatal 相当于在 Print() 之后调用 os.Exit(1)
func Fatal(v ...interface{}) {
    std.Output(2, fmt.Sprint(v...))
    os.Exit(1)
}

// Panic 相当于在 Print() 之后调用 panic()
func Panic(v ...interface{}) {
    s := fmt.Sprint(v...)
    std.Output(2, s)
    panic(s)
}
```