---
title: Go 语言中的错误处理
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

这里为什么会这样呢？其实道理很简单，首先 NewError 方法(也就是 errors.New 方法) 返回了 error 接口变量，err1 和 err2 都是。那么要比较两个接口变量，要满足其中一个条件：

1. 接口变量的实际类型和实际值是 nil。
2. 接口变量的实际类型是相同的并且是 comparable 的类型。

其中条件 2 非常好理解，实际类型如果都不相同，那肯定不相同。条件 1 可以这么想，如果一个接口变量的实际值和实际类型都是 nil，那么另外一个接口变量要相等只有一种情况，就是实际值和实际类型一样都是 nil，其他情况都不相等（比如实际类型不是 nil，实际值是 nil，参照条件 2 所以两者不相等。实际类型是 nil，实际值不是 nil 的情况不可能发生）。

另外值得注意的是，要使得条件 1 中的接口变量的实际类型和实际值都是 nil，要保证一开始赋值的 nil 是纯净的 nil。举个例子：

```Golang
package main

type Greeter interface {
	SayHello(name string) string
}

type User struct {
}

func (*User) SayHello(name string) string {
	return fmt.Sprintf("Hello %q", name)
}


func main() {
	// 1: Pure nil
	var gt1 Greeter = nil
	var u *User = nil
	// 2: Non-Pure nil
	var gt2 Greeter = u
}
```

在上面的例子中，gt1 的 Greeter 接口变量在一开始赋值的时候用纯净的 nil 字面量直接赋值，所以实际类型和实际值都是 nil。gt2 的 Greeter 接口变量是用了一个指针值是 nil 的 User 指针变量去赋值，所以 gt2 的实际类型是 *User，实际值是 nil。

说到这里，就讲清楚了为什么两个相同构造参数的 error 接口变量不想等。确实，在极端情况下，有可能有两个 error 它们虽然底层的 s 字符串相同，但可能来自不同的包，可不能搞混。

所以 errors.New 方法的正确用法是：在包中定义 error 常量，然后返回错误的时候直接返回 error 常量，而不是每次用到的时候用 errors.New 直接返回一个新的 error。举个例子：

```Golang
package Cert

import (
	"errors"
)

type Cert struct {}

const (
	ErrCertNotFound error = errors.New("can not find the specified cert")
	ErrCertIsInvalid error = errors.New("cert content is invalid")
)

func GetCert(id int64) (Cert, error) {
	// return nil, errors.New("can not find the specified cert") 不要这么干
	return nil, ErrCertNotFound
}

func ParseCert(content string) (Cert, error) {
	// 	return nil, errors.New("cert content is invalid") 不要这么干
	return nil, ErrCertIsInvalid
}
```

未完待续...