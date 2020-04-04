---
layout: post
title:  "Go函数闭包的练习"
date:   2019-11-02 10:37:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Go斐波那契数列
mathjax: true
typora-root-url: ../
---

## 练习：斐波纳契闭包

现在来通过函数做些有趣的事情。

实现一个 `fibonacci` 函数，返回一个函数（一个闭包）可以返回连续的斐波纳契数。

> **斐波那契数列**指的是这样一个数列 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233，377，610，987，1597，2584，4181，6765，10946，17711，28657，46368........
>
> 这个数列从第3项开始，每一项都等于前两项之和。

```go
package main

import "fmt"

// fibonacci 函数会返回一个返回 int 的函数。
func fibonacci() func() int {
	r, sum1, sum2 := 0, 0, 0
	return func() int {
		if sum1 == 0 {
			sum1 = 1
			return 1
		} else if sum2 == 0 {
			sum2 = 1
			return 1
		}
		r = sum1 + sum2
		sum1 = sum2
		sum2 = r
		return r
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 20; i++ {
		fmt.Println(f())
	}
}
```

Go 函数可以是闭包的。闭包是一个函数值，它来自函数体的外部的变量引用。函数可以对这个引用值进行访问和赋值；换句话说这个函数被“绑定”在这个变量上。

