---
layout: post
title:  "Go结构struct"
date:   2019-12-10 22:15:00 +0800
categories: Go
tags: Go-Language
excerpt: Go结构struct
mathjax: true
typora-root-url: ../
---

# 结构struct

1. go中的struct与c中的struct非常相似，并且go没有class
2. 使用type \<Name\> struct{}定义结构，名称遵循可见性规则
3. 支持指向自身的指针类型成员
4. 支持匿名结构，可用作成员或定义成员变量
5. 可以使用字面值对结构进行初始化
6. 允许直接通过指针来读写结构成员
7. 相同类型的成员可进行直接拷贝赋值
8. 支持==和!=比较运算符，不支持\< >
9. 支持匿名字段，本质上是定义以某个类型名为名称的字段
10. 嵌入结构作为匿名字段看起来像继承，但不是继承
11. 可以使用匿名字段指针

```go
package main

import "fmt"

type person struct {
	Name string
	Age int
}

func main() {
	a := person{}
	fmt.Println(a)
	a.Name = "Rain"
	a.Age = 18
	fmt.Println(a)
  b := person{
		Name: "Ronnie",
		Age: 10,
	}
	fmt.Println(b)
}
//{ 0}
//{Rain 18}
//{Ronnie 10}
```

```go
package main

import "fmt"

type person struct {
	Name string
	Age int
}

func main() {
	a := person{
		Name: "Rain",
		Age: 18,
	}
	fmt.Println(a)
	A(a)   //值拷贝，所以a的值没有改
	fmt.Println(a)
	B(&a)
	fmt.Println(a)
}

func A(per person)  {
	per.Age = 19
	fmt.Println("A", per)
}

func B(per *person)  {
	per.Age = 20
	fmt.Println("B", per)
}
//{Rain 18}
//A {Rain 19}
//{Rain 18}
//B &{Rain 20}
//{Rain 20}
```

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
}

func main() {
   a := &person{   //a变成指向struct的指针
      Name: "Rain",
      Age: 18,
   }
   a.Name = "Ronnie"  //还是可以直接a.Name
   fmt.Println(a)
   A(a)
   fmt.Println(a)
}

func A(per *person)  {
   per.Age = 19
   fmt.Println("A", per)
}
//&{Ronnie 18}
//A &{Ronnie 19}
//&{Ronnie 19}
```

匿名结构

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
}

func main() {
   a := struct{
      Name string
      Age int
   }{
      Name: "Rain",
      Age: 18,
   }

   fmt.Println(a)
}
//{Rain 18}
```

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
}

func main() {
   a := &struct{
      Name string
      Age int
   }{
      Name: "Rain",
      Age: 18,
   }

   fmt.Println(a)
}
//&{Rain 18}
```

匿名结构嵌套

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
   Contact struct{  //嵌套在struct里的匿名结构
      Phone, City string
   }
}

func main() {
   a := person{
      Name: "Rain",
      Age: 18,
   }
   a.Contact.City = "shanghai"  //Contact是persion的字段名，struct没有名称，所以只能这样初始化
   a.Contact.Phone = "12345678"

   fmt.Println(a)
}
//{Rain 18 {12345678 shanghai}}
```

匿名字段

```go
package main

import "fmt"

type person struct {
   string
   int
}

func main() {
   a := person{"Rain", 18}
   fmt.Println(a)

   //b := person{18, "Rain"}  因为是匿名字段，所以必须严格按照struct定义字段的顺序来初始化，否则会报错
   //fmt.Println(b)
}
//{Rain 18}
```

struct赋值

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
}

func main() {
   a := person{"Rain", 18}
   var b person
   b = a  //可以直接赋值给b
   fmt.Println(b)
}
//{Rain 18}
```

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
}
type person1 struct {
   Name string
   Age int
}

func main() {
   a := person{"Rain", 18}
   b := person1{"Rain", 18}
   fmt.Println(b == a)  //person跟person1是不同的type，所以不可以比较
}
//invalid operation: b == a (mismatched types person1 and person)
```

struct的比较

```go
package main

import "fmt"

type person struct {
   Name string
   Age int
}

func main() {
   a := person{"Rain", 18}
   b := person{"Rain", 18}
   c := person{"Rain", 19}
   fmt.Println(b == a)
   fmt.Println(c == a)
}
//true
//false
```

用嵌入结构实现类似继承的概念，称为组合。嵌入结构默认把所有字段都给了被嵌入的结构

```go
package main

import "fmt"

type human struct {
   Sex int
}

type teacher struct {
   human  //human是human struct类型，这里是没有字段名的，相当于匿名结构，将结构名称默认作为字段名称
   Name string
   Age int
}

type student struct {
   human
   Name string
   Age int
}

func main() {
   a := teacher{Name: "Rain", Age: 18, human: human{Sex: 1}}  
   b := student{Name: "Rain", Age: 18, human: human{Sex: 0}}
 	 a.Name = "Ronnie"
	 a.Age = 20
	 a.human.Sex = 2  
	 fmt.Println(a, b)
	 a.Sex = 3  //a.human.Sex & a.Sex都可以，成功嵌入
	 fmt.Println(a)
}
```

匿名字段与外层结构由同名字段会怎样？

```go
package main

import "fmt"

type A struct {
   B
   Name string
}

type B struct {
   Name string
}


func main() {
   a := A{Name: "A", B: B{Name: "B"}}
   fmt.Println(a.Name, a.B.Name)  //A的Name跟B的Name重名，所以a.Name跟a.B.Name含义不一样了
}
//A B
```

```go
package main

import "fmt"

type A struct {
   B
}

type B struct {
   Name string
}


func main() {
   a := A{B: B{Name: "B"}}  
   fmt.Println(a.Name, a.B.Name)  //A里没有Name，所以a.Name会往下找，找到嵌入的结构B里的Name
//B B
```

```go
package main

import "fmt"

type A struct {
   B
   C
}

type B struct {
   Name string
}

type C struct {
   Name string
}


func main() {
   a := A{B: B{Name: "B"}, C: C{Name: "C"}}
   fmt.Println(a.Name, a.B.Name, a.C.Name)  //当a没有Name字段，往下找的时候，发现B和C里都有Name，就冲突了
}
//ambiguous selector a.Name
```