---
title: Go 语言中的错误处理
date: 2021-10-30
tags: Golang
---

# Go 语言中的错误处理

这篇博客只是想记录一下，个人认为 Go 语言中的错误处理应该是怎样的

## Go 标准库 errors 中的 errors.New()

errors.New 方法返回一个 errorString 的结构体指针，然后这个 errorString 结构体上有一个指针类型的 Error 方法，实现了 error 接口

```Golang
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}

func New(s string) error {
    return &errorString{s}
}
```

这里值得一提的是，errors.New 方法返回了一个 error 接口变量，这个接口变量的接口类型是 error，实际值类型是 errorString 指针，实际值就是个指针。在这种情况下，我如果用 errors.New 的方法创建两个参数一样的变量，这两个变量直接比较肯定是不相等的。（下面的例子把 errors.New 的实现和 errorString 结构体复制过来了，因为 errorString 结构体在 errors 包中是私有的，如果没办法引用到的话不是很好举例）

```Golang 

package main

import (
	"differ-sample/pkg"
)

type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}

func NewError(s string) error {
	return &errorString{s: s}
}

func main() {
	err1 := NewError("A error")
	err2 := NewError("A error")

    // 1: not equal
	pkg.PrintEqualOrNotEqual(err1 == err2)

	{
		err1 := err1.(*errorString)
		err2 := err2.(*errorString)

        // 2: not equal
		pkg.PrintEqualOrNotEqual(err1 == err2)
        // 3: not equal
		pkg.PrintEqualOrNotEqual(*err1 == *err2)
	}
}

// pkg.PrintEqualOrNotEqual 方法的实现如下：
// func PrintEqualOrNotEqual(isEqual bool) {
// 	if isEqual {
// 		fmt.Println("equal")
// 	} else {
// 		fmt.Println("not equal")
// 	}
// }
```

这里为什么会这样呢？其实道理很简单，首先 NewError 方法(也就是 errors.New 方法) 返回了 error 接口变量，err1 和 err2 都是。

... 未完待续