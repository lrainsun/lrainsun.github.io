---
layout: post
title:  "Go接口interface"
date:   2019-12-11 11:15:00 +0800
categories: Go
tags: Go-Language
excerpt: Go接口interface
mathjax: true
typora-root-url: ../
---

# 接口interface

1. 接口是一个或多个方法签名的集合
2. 只要某个类型拥有该接口的所有方法签名，即算实现该接口，无需显式声明实现了哪个接口，这称为Structural Typing
3. 接口只有方法声明，没有实现，没有数据字段
4. 接口可以匿名嵌入其它接口，或嵌入到结构中
5. 将对象赋值给接口时，会发生拷贝，而接口内部存储的是指向这个复制品的指针，既无法修改复制品的状态，也无法获取指针
6. 只有当接口存储的类型和对象都为nil时，接口才等于nil
7. 接口调用不会做receiver的自动转换
8. 接口同样支持匿名字段方法
9. 接口也可实现类似OOP中的多态
10. 空接口可以作为任何类型数据的容器

```go
package main

import "fmt"

type USB interface {
	Name() string
	Connect()
}

type PhoneConnecter struct {
	name string
}

func (pc PhoneConnecter) Name() string {
	return pc.name
}

func (pc PhoneConnecter) Connect() {
	fmt.Println("Connect:", pc.name)
}

func Disconnect(usb USB)  {
	fmt.Println("Disconnected.")
}

func main() {
	a := PhoneConnecter{"PhoneConnector"}
	a.Connect()
	Disconnect(a)  //PhoneConnecter实现了USB接口，所以将a作为入参传入也可以
}
//Connect: PhoneConnector
//Disconnected.
```

嵌入接口

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter  //这里是嵌入接口，USB就拥有了Connecter下的方法
}

type Connecter interface {
   Connect()
}

type PhoneConnecter struct {
   name string
}

func (pc PhoneConnecter) Name() string {
   return pc.name
}

func (pc PhoneConnecter) Connect() {
   fmt.Println("Connect:", pc.name)
}

func Disconnect(usb USB)  {
   fmt.Println("Disconnected.")
}

func main() {
   a := PhoneConnecter{"PhoneConnector"}
   a.Connect()
   Disconnect(a)
}
//Connect: PhoneConnector
//Disconnected.
```

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter
}

type Connecter interface {
   Connect()
}

type PhoneConnecter struct {
   name string
}

func (pc PhoneConnecter) Name() string {
   return pc.name
}

func (pc PhoneConnecter) Connect() {
   fmt.Println("Connect:", pc.name)
}

func Disconnect(usb USB)  {
   if pc, ok := usb.(PhoneConnecter); ok {  //类型判断，如果是PhoneConnecter类型，则可以取到name
      fmt.Println("Disconnected: ", pc.name)
      return
   }
   fmt.Println("Unknown device.")
}

func main() {
   a := PhoneConnecter{"PhoneConnector"}
   a.Connect()
   Disconnect(a)
}
//Connect: PhoneConnector
//Disconnected:  PhoneConnector
```

空接口，任何type都实现了空接口；

类型判断：

* 通过类型断言的ok pattern可以判断接口中的数据类型
* 使用type switch则可针对空接口进行比较全面的类型判断

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter
}

type Connecter interface {
   Connect()
}

type PhoneConnecter struct {
   name string
}

func (pc PhoneConnecter) Name() string {
   return pc.name
}

func (pc PhoneConnecter) Connect() {
   fmt.Println("Connect:", pc.name)
}

func Disconnect(usb interface{})  {
   switch v := usb.(type) {
   case PhoneConnecter:
      fmt.Println("Disconnected: ", v.name)
   default:
      fmt.Println("Unknown device.")
   }
}

func main() {
   a := PhoneConnecter{"PhoneConnector"}
   a.Connect()
   Disconnect(a)
}
//Connect: PhoneConnector
//Disconnected:  PhoneConnector
```

接口的相互转换，只能将拥有超集的接口转换为子集的接口

USB可以转换为Connecter，而Connecter不可以转换成USB

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter
}

type Connecter interface {
   Connect()
}

type PhoneConnecter struct {
   name string
}

func (pc PhoneConnecter) Name() string {
   return pc.name
}

func (pc PhoneConnecter) Connect() {
   fmt.Println("Connect:", pc.name)
}


func main() {
   a := PhoneConnecter{"PhoneConnector"}  //PhoneConnecter实现了USB接口
   var b Connecter = Connecter(a)  //USB接口转换成Connecter接口
   b.Connect()
   //b.Name() b.Name undefined (type Connecter has no field or method Name) Connector没有Name方法
}
//Connect: PhoneConnector
```

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter
}

type Connecter interface {
   Connect()
}

type TVConnecter struct {
   name string
}

func (tv TVConnecter) Connect()  {
   fmt.Println("Connected: ", tv.name)
}


func main() {
   a := TVConnecter{"PVConnector"}
   var b USB = USB(a) //cannot convert a (type TVConnecter) to type USB:TVConnecter does not implement USB (missing Name method)
   b.Connect()

}
```

复制品

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter
}

type Connecter interface {
   Connect()
}

type PhoneConnecter struct {
   name string
}

func (pc PhoneConnecter) Name() string {
   return pc.name
}

func (pc PhoneConnecter) Connect() {
   fmt.Println("Connect:", pc.name)
}


func main() {
   a := PhoneConnecter{"PhoneConnector"}
   var b Connecter = Connecter(a)  //当a赋值给接口Connecter时，会发生拷贝，所以接口b里拿到的其实是复制品，无法修改也无法获取原有对象的修改

   a.Connect()
   b.Connect()

   a.name = "pc"
   a.Connect()
   b.Connect()
}
//Connect: PhoneConnector
//Connect: PhoneConnector
//Connect: pc
//Connect: PhoneConnector
```

接口为nil

```go
package main

import "fmt"

type USB interface {
   Name() string
   Connecter
}

type Connecter interface {
   Connect()
}

type PhoneConnecter struct {
   name string
}

func (pc PhoneConnecter) Name() string {
   return pc.name
}

func (pc PhoneConnecter) Connect() {
   fmt.Println("Connect:", pc.name)
}


func main() {
   var a interface{}
   fmt.Println(a == nil) //只有当接口存储的类型和对象都为nil时，接口才等于nil

   var p *int = nil
   a = p
   fmt.Println(a == nil) //这里接口存储的类型不为nil
}
//true
//false
```