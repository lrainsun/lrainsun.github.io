---
layout: post
title:  "Go Interface"
date:   2019-11-25 14:30:00 +0800
categories: Go
tags: Go-Learning
excerpt: Go Interface的理解
mathjax: true
typora-root-url: ../
---

# Interface概述

Interface是Go语言编程中数据类型的关键。

Go不是一种典型的OO语言，它在语法上不支持类和继承的概念。

Go语言引入了一种新类型—Interface，可以实现类似“多态"的效果。

* Go没有类的概念，但数据类型可以定义对应的method(s)，也就是作用于特定数据类型上的函数
* 函数的签名中，需要有一个receiver来表明当前定义的函数会作用在该receiver上（func后，method名前）
* 除Interface以外其他数据类型都可以定义method，但一般我们把method定义在struct上（struct可以看作是不支持继承行为的轻量级的"类"）

# 为什么要定义Interface

## 泛型编程

```go
type InterfaceA interface {
  MethodA() int
  MethodB() bool
}

func TestInferface(i InterfaceA) {
  i.MethodA()
  i.MethodB()
}
```

InterfaceA是一个interface，定义了两个方法，在使用的时候，可以不管实际类型，只要实现这两个方法就可以了

## 隐藏具体实现

这也是interface本身一个重要的作用，隐藏内部的实现，只对外暴露接口

interface的实现可以有很多种方法，但是用户是没有感知的

## providing interception points 

还不理解，大概是 wrapper 或者装饰器

# Interface是一种类型

```go
type I interface {
    Get() int
}
```

**interface 是一种类型**，从它的定义可以看出来用了 type 关键字，更准确的说 interface 是一种**具有一组方法的类型**，这些方法定义了 interface 的行为。

go 允许不带任何方法的 interface ，这种类型的 interface 叫 **empty interface**。

**如果一个类型实现了一个 interface 中所有方法，我们说类型实现了该 interface**，所以所有类型都实现了 empty interface。

让我比较困惑的是，一般对接口的实现都会显示的定义，而在看代码的时候，有时会找不到interface的实现。因为go 没有显式的关键字用来实现 interface，只需要实现 interface 包含的方法即可。

# interface变量存储

如果有多种类型实现了某个interface，这些类型的值都可以直接使用interface的变量存储，也就是说interface变量存储的是实现者的值

在使用 interface 时不需要显式在 struct 上声明要实现哪个 interface ，只需要实现对应 interface 中的方法即可，go 会自动进行 interface 的检查，并在运行时执行从其他类型到 interface 的自动转换，即使实现了多个 interface，go 也会在使用对应 interface 时实现自动转换。

# interface变量存储类型

一个 interface 被多种类型实现时，有时候我们需要区分 interface 的变量究竟存储哪种类型的值。

go 可以使用  `value, ok := element.(T)`：**element 是 interface 类型的变量，T代表要断言的类型，value 是 interface 变量存储的值，ok 是 bool 类型表示是否为该断言的类型 T** 如果element里面确实存储了T类型的数值，那么ok返回true，否则返回false。

```go
if t, ok := i.(*S); ok {
    fmt.Println("s implements I", t)
}
```

ok 是 true 表明 i 存储的是 *S 类型的值，false 则不是，这种区分能力叫 [Type assertions](https://tour.golang.org/methods/15) (类型断言)。

如下面例子就是当obj的类型不是v1.ReplicationController的时候，需要做一些转换处理。

```go
rc, ok := obj.(*v1.ReplicationController)
if !ok {
   // Convert the Obj inside DeletedFinalStateUnknown.
   tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
   if !ok {
   }
}
```

如果需要区分多种类型，可以使用 switch 断言，更简单直接，这种断言方式只能在 switch 语句中使用。

```go
switch t := i.(type) {
case *S:
    fmt.Println("i store *S", t)
case *R:
    fmt.Println("i store *R", t)
}
```

如下面的例子，根据obj的类型决定下面的处理

```go
				switch t := obj.(type) {
				case *v1.Secret:
					return t.Type == bootstrapapi.SecretTypeBootstrapToken && t.Namespace == e.tokenSecretNamespace
				default:
					utilruntime.HandleError(fmt.Errorf("object passed to %T that is not expected: %T", e, obj))
					return false
				}
```

# 实现者的receiver

receiver是value，这个时候没问题:

```go
type Abser interface {
	Abs() float64
}

func getAbs(i Abser) {
	fmt.Println(i.Abs())
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

func main() {
	f := MyFloat(-math.Sqrt2)
	getAbs(f)
	getAbs(&f)
}
```

*func (f MyFloat) Abs() float64* 的receiver是*MyFloat*(value)，如果receiver是value，那么我们无论是传入f(value)，还是&f(pointer)，都是可以正常工作的：

* 如果按value调用那肯定没问题
* 如果是按 pointer 调用，go 会自动进行转换，因为有了指针总是能得到指针指向的值是什么，**go 会把指针进行隐式转换得到 value**

那反过来呢？

```go
type Abser interface {
	Abs() float64
}

func getAbs(i Abser) {
	fmt.Println(i.Abs())
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}

	getAbs(&v)
	getAbs(v)  //wrong
}
```

*func (v \*Vertex) Abs() float64*的reciever是\*Vertex，是pointer，那么getAbs(&v)的调用是没问题的，而getAbs(v)就会提示错误

> cannot use v (type Vertex) as type Abser in argument to getAbs:
> 	Vertex does not implement Abser (Abs method has pointer receiver)

是因为，Go中的函数都是按值传递的。如果receiver是pointer，那么pointer本身就作为 value，而go 就不知道对应的 value（实际是pointer） 的原始值是什么，因为 value 是份拷贝。

# 空interface

interface可以不定义任何方法，那有啥用呢？

空interface在我们需要存储任意类型的数值的时候相当有用，因为它可以存储任意类型的数值。

一个函数把 interface{} 作为参数，那么他可以接受任意类型的值作为参数，如果一个函数返回 interface{}，那么也就可以返回任意类型的值。

# 嵌入interface

如果一个interface1作为interface2的一个嵌入字段，那么interface2隐式的包含了interface1里面的method

```go
type Abser interface {
	Abs() float64
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

func (f MyFloat) Test() float64 {
	return 0
}

type Math interface{
	Abser
	Test() float64
}

func getAbs2(i Math)  {
	fmt.Println(i.Abs())
}

func main() {
	f := MyFloat(-math.Sqrt2)

	getAbs2(f)
}
```

在刚刚的基础上我们新定义了一个interface Math，它包含了Abser里的方法

# reflect

reflect就是能检查程序在运行时的状态。我们一般用到的包是reflect包

使用reflect一般分成三步，下面简要的讲解一下：要去反射是一个类型的值(这些值都实现了空interface)，首先需要把它转化成reflect对象(reflect.Type或者reflect.Value，根据不同的情况调用不同的函数)。这两种获取方式如下：

```go
`t := reflect.TypeOf(i)  ``// 得到类型的元数据，通过t我们能获取类型定义里面的所有元素``v := reflect.ValueOf(i)  ``// 得到实际的值，通过v我们获取存储在里面的值，还可以去改变值`
```

转化为reflect对象之后我们就可以进行一些操作了，也就是将reflect对象转化成相应的值，例如

```go
`tag := t.Elem().Field(0).Tag ``// 获取定义在struct里面的标签``name := v.Elem().Field(0).String() ``// 获取存储在第一个字段里面的值`
```

```go
func getAbs(i Abser) {
   fmt.Println(i.Abs())
   t := reflect.TypeOf(i)
   fmt.Println(t)
   v := reflect.ValueOf(i)
   fmt.Println(v)
}

func main() {
	f := MyFloat(-math.Sqrt2)

	getAbs(f)
}
```

我们得到如下输出

```
1.4142135623730951
main.MyFloat
-1.4142135623730951
```

