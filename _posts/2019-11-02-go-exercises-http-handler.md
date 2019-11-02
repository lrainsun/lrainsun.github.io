---
layout: post
title:  "Go HTTP Handler的练习"
date:   2019-11-02 17:20:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Go HTTP Handler的练习
mathjax: true
typora-root-url: ../
---

## 练习：HTTP 处理

实现下面的类型，并在其上定义 ServeHTTP 方法。在 web 服务器中注册它们来处理指定的路径。

```
type String string

type Struct struct {
    Greeting string
    Punct    string
    Who      string
}
```

例如，可以使用如下方式注册处理方法：

```
http.Handle("/string", String("I'm a frayed knot."))
http.Handle("/struct", &Struct{"Hello", ":", "Gophers!"})
```

**注意：** 这个例子无法在基于 web 的用户界面下运行。 为了尝试编写 web 服务，你可能需要 [安装 Go](http://golang.org/doc/install/)。

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

type String string

func (s String) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, s)
}

type Struct struct {
	Greeting string
	Punct    string
	Who      string
}

func (s Struct) String() string {
	return fmt.Sprintf("%v %v %v", s.Greeting, s.Punct, s.Who)
}

func (s Struct) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, s)
}

func main() {
	// your http.Handle calls here
	http.Handle("/string", String("I'm a frayed knot."))
	http.Handle("/struct", &Struct{"Hello", ":", "Gophers!"})

	log.Fatal(http.ListenAndServe("localhost:4000", nil))
}
```

运行结果：

![image-20191102171849474](/assets/images/image-20191102171849474.png)

![image-20191102171927438](/assets/images/image-20191102171927438.png)

## Web 服务器

[包 http](http://golang.org/pkg/net/http/) 通过任何实现了 `http.Handler` 的值来响应 HTTP 请求：

```
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}
```

在这个例子中，类型 `Hello` 实现了 `http.Handler`。

访问 http://localhost:4000/ 会看到来自程序的问候。