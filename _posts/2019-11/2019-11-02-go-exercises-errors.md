---
layout: post
title:  "Go Error的练习"
date:   2019-11-02 16:20:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Go Error的练习
mathjax: true
typora-root-url: ../
---

## 练习：错误

从之前的练习中复制 `Sqrt` 函数，并修改使其返回 `error` 值。

`Sqrt` 接收到一个负数时，应当返回一个非 nil 的错误值。复数同样也不被支持。

创建一个新类型

```
type ErrNegativeSqrt float64
```

为其实现

```
func (e ErrNegativeSqrt) Error() string
```

使其成为一个 `error`， 该方法就可以让 `ErrNegativeSqrt(-2).Error()` 返回 `"cannot Sqrt negative number: -2"`。

**注意：** 在 `Error` 方法内调用 `fmt.Sprint(e)` 将会让程序陷入死循环。可以通过先转换 `e` 来避免这个问题：`fmt.Sprint(float64(e))`。请思考这是为什么呢？

修改 `Sqrt` 函数，使其接受一个负数时，返回 `ErrNegativeSqrt` 值。

```go
package main

import (
	"fmt"
)

func Sqrt(x float64) (float64, error) {
	z := float64(1)
	if x < 0 {
		return 0, ErrNegativeSqrt(x)
	} else {
		for i := 1; i <= 10; i++ {
			z = z - (z*z-x)/(2*z)
		}
		return z, nil
	}
}

type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
	return fmt.Sprintf("cannot Sqrt negative number: %v", float64(e))
}


func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}
```

## 错误

Go 程序使用 `error` 值来表示错误状态。

与 `fmt.Stringer` 类似，`error` 类型是一个内建接口：

```
type error interface {
    Error() string
}
```

（与 `fmt.Stringer` 类似，`fmt` 包在输出时也会试图匹配 `error`。）

通常函数会返回一个 `error` 值，调用的它的代码应当判断这个错误是否等于 `nil`， 来进行错误处理。

```
i, err := strconv.Atoi("42")
if err != nil {
    fmt.Printf("couldn't convert number: %v\n", err)
}
fmt.Println("Converted integer:", i)
```

`error` 为 nil 时表示成功；非 nil 的 `error` 表示错误。