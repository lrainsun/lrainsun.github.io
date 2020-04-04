---
layout: post
title:  "Go函数func"
date:   2019-12-09 16:15:00 +0800
categories: Go
tags: Go-Language
excerpt: Go函数func
mathjax: true
typora-root-url: ../
---

# func函数

1. Go 函数 == 不支持 == 嵌套、重载和默认参数
2. 但支持以下特性：
   1. 无需声明原型、不定长度变参、多返回值、命名返回值参数、匿名函数、闭包
3. 定义函数使用关键字 func，且左大括号不能另起一行
4. 函数也可以作为一种类型使用

```go
package main

import "fmt"

func main() {
}

//func A(a int ,b string) (int ,string, string) {
//func A(a int ,b string) int {
//func A(a int ,b string) int {
//func A(a int ,b int, c int) int){
//func A(a ,b , c int) int{
//func A() (a ,b , c int){

//func A() (int, int, int){
////如果没有命名返回值,return需要明确
//a, b, c := 1, 2, 3
//return a, b, c
////如果已经命名返回值func A() (a ,b , c int){
////直接return即可
//}

//func B() (a, b, c int) {
//  a, b, c = 1, 2, 3
//  return
//
//}
```

不定长变参

```go
package main

import "fmt"

func main()  {
   A(1, 2)
}

func A(a ...int)  { 
   fmt.Println(a)  //a就变成了slice
}
//[1,2]
```

```go
package main

import "fmt"

func main()  {
   A("HI", 1, 2)
}

func A(b string, a ...int)  { //不定长变参要放在最后
   fmt.Println(b, a)
}
//HI [1 2]
```

```go
package main

import "fmt"

func main()  {
   a, b := 1, 2
   A(a, b)
   fmt.Println(a, b)
}

func A(s ...int)  { //虽然s是slice，但其实是值拷贝
   s[0] = 3
   s[1] = 4
   fmt.Println(s)
}
//[3 4]
//1 2
```

```go
package main

import "fmt"

func main()  {
   s1 := []int{1,2,3,4}
   A(s1)
   fmt.Println(s1)
}

func A(s []int)  {  //函数内修改slice的值，slice本身也被修改了，但并不是传递了指针，而是对slice的内存地址进行了一个拷贝
   s[0] = 5
   s[1] = 6
   s[2] = 7
   s[3] = 8
   fmt.Println(s)
}
//[5 6 7 8]
//[5 6 7 8]
```

```go
package main

import "fmt"

func main()  {
   a := 1
   b := 2
   A(a, &b)
   fmt.Println(a, b)
}

func A(a int, b *int)  {  //a是拷贝值，b是拷贝地址
   a = 3
   *b = 4
   fmt.Println(a, *b)
}
//3 4
//1 4
```

```go
package main

import "fmt"

func main()  {
   a := A  //a是A的复制品，就是A类型，一切皆类型
   a()
}

func A()  {
   fmt.Println("Func A")
}
//Func A
```

匿名函数

```go
package main

import "fmt"

func main()  {
   a := func() {
      fmt.Println("Func A")
   }
   a()
}
//Func A
```

闭包，其实是函数的嵌套，内层的函数可以使用外层函数的所有变量

闭包是由函数及其相关引用环境组合而成的实体(即：闭包=函数+引用环境)

```go
package main

import "fmt"

func main()  {
   f := closure(10)
   fmt.Println(f(1))
   fmt.Println(f(2))
}

func closure(x int) func(int) int {  //closure返回了一个func(int) int的函数
   fmt.Printf("%p\n", &x)
   return func(y int) int {
      fmt.Printf("%p\n", &x)
      return x + y   //这个匿名函数并没有定义x，而是引用了它所在的环境closure中的变量x
   }
}
//0xc000098008
//0xc000098008
//11
//0xc000098008
//12
```

# defer

1. defer的执行方式类似其它语言中的析构函数，在函数体执行结束后按照调用顺序的相反顺序逐个执行
2. 即使函数发生严重错误也会执行
3. 支持匿名函数的调用
4. 常用于资源清理、文件关闭、解锁以及记录时间等操作
5. 通过与匿名函数配合可在 return 之后修改函数计算结果
6. 如果函数体内某个变量作为 defer 时匿名函数的参数，则在定义 defer时即已经获得了拷贝，否则则是引用某个变量的地址
7. Go 没有异常机制，但有 panic/recover 模式来处理错误
8. Panic 可以在任何地方引发，但 recover 只有在 defer 调用的函数中有效

```go
package main

import "fmt"

func main()  {
   fmt.Println("a")
   defer fmt.Println("b")
   defer fmt.Println("c")
}
//a
//c
//b
```

```go
package main

import "fmt"

func main()  {
   for i := 0; i < 3; i++ {
      defer fmt.Println(i)
   }
}
//2
//1
//0
```

```go
package main

import "fmt"

func main()  {
   for i := 0; i < 3; i++ {
      defer func() {
         fmt.Println(i)
      }()  //defer是调用这个函数，其实是个闭包，i是地址的引用，引用了i的地址
   }
}
//3
//3
//3
```

panic

```go
package main

import "fmt"

func main()  {
   A()
   B()
   C()
}

func A()  {
   fmt.Println("Func A")
}

func B() {
  panic("Panic in B")  //panic相当于抛异常，然后后面的c()就不会再执行了
}

func C() {
   fmt.Print("Func C")
}
//Func A
//panic: Panic in B
```

```go
package main

import "fmt"

func main()  {
   A()
   B()
   C()
}

func A()  {
   fmt.Println("Func A")
}

func B() {
   defer func() {
      if err := recover(); err != nil {
         fmt.Println("Recover in B")
      }
   }()  //defer需定义在panic之前
   panic("Panic in B")
}

func C() {
   fmt.Print("Func C")
}
//Func A
//Recover in B
//Func C
```

```go
package main

import "fmt"

func main() {
    var fs = [4]func(){}

    for i := 0; i<4; i++ {
        defer fmt.Println("defer i = ", i) //这里没有匿名函数，i变成了一个参数，值拷贝
        defer func() { fmt.Println("defer_closure i = ", i )}() //同理，i也是引用了地址
        fs[i] = func() {
            fmt.Println("closure i = ", i)  //i没有在函数内定义，用的是外层函数的定义，其实是个引用地址，就是for循环里i的地址，for循环完成后，最后i=4，然后输出i的值(也就是i的地址所存的值，就是4)
        }
    }

    for _, f := range fs {
        f()
    }
}

//closure i =  4
//closure i =  4
//closure i =  4
//closure i =  4
//defer_closure i =  4
//defer i =  3
//defer_closure i =  4
//defer i =  2
//defer_closure i =  4
//defer i =  1
//defer_closure i =  4
//defer i =  0
```

