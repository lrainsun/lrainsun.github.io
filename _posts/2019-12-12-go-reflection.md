---
layout: post
title:  "Go反射reflection"
date:   2019-12-12 11:15:00 +0800
categories: Go
tags: Go-Language
excerpt: Go反射reflection
mathjax: true
typora-root-url: ../
---

# 反射reflection

1. 反射可大大提高程序的灵猴性，使得interface{}有更大的发挥余地
2. 反射使用TypeOf和ValueOf函数从接口中获取目标对象信息
3. 反射会将匿名字段作为独立字段（匿名字段本质）
4. 想要利用反射修改对象状态，前提是interface.data是settable,即pointer-interface
5. 通过反射可以“动态”调用方法

```go
package main

import (
   "fmt"
   "reflect"
)

type User struct{
   Id int
   Name string
   Age int
}

func (u User) Hello() {
   fmt.Println("Hello world.")
}

func main() {
   u := User{1, "OK", 12}
   Info(u)  //Info(&u) will return panic: reflect: NumField of non-struct type
}

func Info(o interface{})  {  //传入一个空接口，通过reflect，可以识别出type, method, field
   t := reflect.TypeOf(o)
   fmt.Println("Type:", t.Name())

   v := reflect.ValueOf(o)
   fmt.Println("Fields:")

   for i := 0; i < t.NumField(); i++ {
      f := t.Field(i)
      val := v.Field(i).Interface()
      fmt.Printf("%6s: %v = %v\n", f.Name, f.Type, val)
   }

   for i := 0; i < t.NumMethod(); i++ {
      m := t.Method(i)
      fmt.Printf("%6s: %v\n", m.Name, m.Type)
   }
}
```

只有类型是reflect.Struct类型，才可以像上面这样进行反射

如果传入指针，是不能work的，所以可以先判断一下类型

```go
package main

import (
   "fmt"
   "reflect"
)

type User struct{
   Id int
   Name string
   Age int
}

func (u User) Hello() {
   fmt.Println("Hello world.")
}

func main() {
   u := User{1, "OK", 12}
   Info(&u)
}

func Info(o interface{})  {
   t := reflect.TypeOf(o)
   fmt.Println("Type:", t.Name())

   if k := t.Kind(); k != reflect.Struct {
      fmt.Println("type error")
      return
   }

   v := reflect.ValueOf(o)
   fmt.Println("Fields:")

   for i := 0; i < t.NumField(); i++ {
      f := t.Field(i)
      val := v.Field(i).Interface()
      fmt.Printf("%6s: %v = %v\n", f.Name, f.Type, val)
   }

   for i := 0; i < t.NumMethod(); i++ {
      m := t.Method(i)
      fmt.Printf("%6s: %v\n", m.Name, m.Type)
   }
}
//Type: 
//type error
```

反射匿名或者嵌入字段，通过索引来取得匿名字段的信息，通过Anonymous可以的值是否是匿名字段

```go
package main

import (
   "fmt"
   "reflect"
)

type User struct{
   Id int
   Name string
   Age int
}

type Manager struct {
   User
   title string
}

func main() {
   m := Manager{User: User{1, "OK", 12}, title: "123"}
   t := reflect.TypeOf(m)

   fmt.Printf("%#v\n", t.Field(0)) //reflect.StructField{Name:"User", PkgPath:"", Type:(*reflect.rtype)(0x10be840), Tag:"", Offset:0x0, Index:[]int{0}, Anonymous:true}
   fmt.Printf("%#v\n", t.Field(1)) //reflect.StructField{Name:"title", PkgPath:"main", Type:(*reflect.rtype)(0x10acdc0), Tag:"", Offset:0x20, Index:[]int{1}, Anonymous:false}

   fmt.Printf("%#v\n", t.FieldByIndex([]int{0, 0})) //reflect.StructField{Name:"Id", PkgPath:"", Type:(*reflect.rtype)(0x10ac540), Tag:"", Offset:0x0, Index:[]int{0}, Anonymous:false}
   fmt.Printf("%#v\n", t.FieldByIndex([]int{0, 1})) //reflect.StructField{Name:"Name", PkgPath:"", Type:(*reflect.rtype)(0x10ad000), Tag:"", Offset:0x8, Index:[]int{1}, Anonymous:false}
}
```

修改匿名字段的内容

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	x := 123
	v := reflect.ValueOf(&x)  //当类型是reflect.Ptr的时候，可以进行修改

	if v.Kind() != reflect.Ptr {
		fmt.Println("type error")
	}
	v.Elem().SetInt(999) //我们明确知道是int才可以这样

	fmt.Println(x)
}
//999
```

```go
package main

import (
   "fmt"
   "reflect"
)

type User struct {
   Id int
   Name string
   Age int
}

func main() {
   u := User{1, "OK", 12}
   Set(&u)
   fmt.Println(u)
}

func Set(o interface{})  {
   v := reflect.ValueOf(o)

   if v.Kind() == reflect.Ptr && !v.Elem().CanSet() {
      fmt.Println("cannot set")
      return
   } else {
      v = v.Elem()
   }
   f := v.FieldByName("Name")
   if !f.IsValid() {
      fmt.Println("bad")
      return
   }
   if f.Kind() == reflect.String {
      f.SetString("byebye")
   }
}
//{1 byebye 12}
```

通过反射进行方法的调用

```go
package main

import (
   "fmt"
   "reflect"
)

type User struct {
   Id int
   Name string
   Age int
}

func main() {
   u := User{1, "OK", 12}
   v := reflect.ValueOf(u)
   mv := v.MethodByName("Hello")

   args := []reflect.Value{reflect.ValueOf("rain")}
   mv.Call(args)
}

func (u User) Hello(name string)  {
   fmt.Println("hello", name, ", my name is", u.Name)
}
//hello rain , my name is OK
```