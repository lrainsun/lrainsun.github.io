---
layout: post
title:  "Go Stringers的练习"
date:   2019-11-02 14:30:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Go Stringer的练习
mathjax: true
typora-root-url: ../
---

## 练习：Stringers

让 `IPAddr` 类型实现 `fmt.Stringer` 以便用点分格式输出地址。

例如，`IPAddr{1,`2,`3,`4}` 应当输出 `"1.2.3.4"`。

```go
package main

import "fmt"

type IPAddr [4]byte

// TODO: Add a "String() string" method to IPAddr.
func (ip IPAddr) String() string {
	return fmt.Sprintf("%v.%v.%v.%v", ip[0], ip[1], ip[2], ip[3])
}

func main() {
	addrs := map[string]IPAddr{
		"loopback":  {127, 0, 0, 1},
		"googleDNS": {8, 8, 8, 8},
	}
	for n, a := range addrs {
		fmt.Printf("%v: %v\n", n, a)
	}
}
```

一个普遍存在的接口是 [`fmt`](http://golang.org/pkg/fmt/) 包中定义的 [`Stringer`](http://golang.org/pkg/fmt/#Stringer)。

```go
type Stringer interface {
    String() string
}
```

`Stringer` 是一个可以用字符串描述自己的类型。`fmt`包 （还有许多其他包）使用这个来进行输出。



