---
layout: post
title:  "Go循环和函数练习"
date:   2019-11-01 10:15:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Hello world后的第一个练习
mathjax: true
typora-root-url: ../
---

## 练习：循环和函数

作为练习函数和循环的简单途径，用牛顿法实现开方函数。

在这个例子中，牛顿法是通过选择一个初始点 *z* 然后重复这一过程求 `Sqrt(x)` 的近似值：

![img](/../assets/images/newton.png)

为了做到这个，只需要重复计算 10 次，并且观察不同的值（1，2，3，……）是如何逐步逼近结果的。 然后，修改循环条件，使得当值停止改变（或改变非常小）的时候退出循环。观察迭代次数是否变化。结果与 [http://golang.org/pkg/math/#Sqrt](http://golang.org/pkg/math/#Sqrt)[math.Sqrt] 接近吗？

```go
package main

import (
	"fmt"
	"math"
)

func Sqrt(x float64) float64 {
	z := float64(1)
	for i := 1; i <= 10; i++ {
		z = z - (z*z-x)/(2*z)
	}
	return z
}

func Sqrt2(x float64) (float64, int) {
	z := float64(1)
	y := z - (z*z-x)/(2*z)
	i := 0
	for ; y - z > 0.00000000001 || z -y > 0.00000000001; {
		y = z
		z = z - (z*z-x)/(2*z)
		i ++
	}
	return z, i
}

func Sqrt3(x float64) (float64, int) {
	z := float64(1)
	i := 0
	for ; (z*z-x)/(2*z) > 0.000000000001 || -(z*z-x)/(2*z) > 0.000000000001; {
		z = z - (z*z-x)/(2*z)
		i ++
	}
	return z, i
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt2(2))
	fmt.Println(Sqrt3(2))
	fmt.Println(math.Sqrt(2))
}
```

结果，大概在5次的时候达到最接近值

```
1.414213562373095
1.4142135623730951 5
1.4142135623730951 5
1.4142135623730951
```

