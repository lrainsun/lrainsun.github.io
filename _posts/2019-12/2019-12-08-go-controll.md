---
layout: post
title:  "Go控制语句"
date:   2019-12-08 22:15:00 +0800
categories: Go
tags: Go-Language
excerpt: Go控制语句
mathjax: true
typora-root-url: ../
---

# 指针

Go 虽然保留了指针，但与其它编程语言不同的是，在 Go 当中不支持指针运算以及”->” 运算符，而直接采用”.” 选择符来操作指针目标对象的成员

操作符”&” 取变量地址，使用”*” 通过指针间接访问目标对象默认值为 nil 而非 NULL

```go
package main

import "fmt"
func main() {
    a := 1
    var p *int = &a
    fmt.Println(p)
    fmt.Println(*p)
}
```

# 递增递减语句

 在 Go 当中，++ 与 -- 是作为语句而并不是作为表达式

```go
package main

import "fmt"
func main() {
    //错误做法 a := a++
    a := 1
    a++
    var p *int = &a
    fmt.Println(*p)
}
```

# 判断语句 if

1. 条件表达式没有括号
2. 支持一个初始化表达式（可以是并行方式）
3. 左大括号必须和条件语句或 else 在同一行
4. 支持单行模式
5. 初始化语句中的变量为 block 级别，同时隐藏外部同名变量

```go
package main
import "fmt"
func main() {
    a : = 1
    if a := 1; a> 1 {

    }
    fmt.Println(a)
}
1. if后面没有括号
2. 在if外定义了之后，if中再定义，默认使用if中定义的
```

```go
package main
import "fmt"
func main() {
    a := 10
    if a := 1; a> 0 {
        fmt.Println(a)
    }
    fmt.Println(a)
}
//1
//10
```

# 循环语句 for

1. Go 只有 for 一个循环语句关键字，但支持 3 种形式
2. 初始化和步进表达式可以是多个值
3. 条件语句每次循环都会被重新检查，因此不建议在条件语句中
4. 使用函数，尽量提前计算好条件并以变量或常量代替
5. 左大括号必须和条件语句在同一行

```go
package main
import "fmt"
func main() {
    a :=1
    for{
        a++
        if a>3 {
            break
        }
        fmt.Println(a)
    }
    fmt.Println(over)
}
//2
//3
//over
```

```go
package main
import "fmt"
func main() {
    a :=1
    for a <= 3 {
        a++
        fmt.Println(a)
    }
    fmt.Println(a)
}

//2
//3
//4
//4
```

```go
package main
import "fmt"
func main() {
    a :=1
    for i := 0; i <3; i++ {
        a++
        fmt.Println(a)
    }
    fmt.Println(a)
}
//2
//3
//4
//4
```

# 选择语句 switch

1. 可以使用任何类型或表达式作为条件语句
2. 不需要写 break，一旦条件符合自动终止
3. 如希望继续执行下一个 case，需使用 fallthrough 语句
4. 支持一个初始化表达式（可以是并行方式），右侧需跟分号
5. 左大括号必须和条件语句在同一行

```go
package main

import "fmt"
func main() {
    a := 1
    switch a {
    case 0:
        fmt.Println("a=0")
    case 1:
        fmt.Println("a=1")
    default:
        fmt.Println("None")
    }
}
```

```go
package main
import "fmt"
func main() {
    a := 1
    switch {
    case a >= 0:
        fmt.Println("a>=0")
        //fallthrough
    case a >= 1:
        fmt.Println("a>=1")
    default:
        fmt.Println("None")
    }
}
// a>= 0
加上fallthrough
// a>= 0
// a>= 1
```

```go
package main
import "fmt"
func main() {
    switch a := 1; {
    case a >= 0:
        fmt.Println("a>=0")
    case a >= 1:
        fmt.Println("a>=1")
    default:
        fmt.Println("None")
    }
    fmt.Println("None")
}
//undefined: a 未定义
存在作用域的问题，所以最好放在switch外面
```

# 跳转语句 goto, break, continue

1. 三个语法都可以配合标签使用
2. 标签名区分大小写，若不使用会造成编译错误
3. Break 与 continue 配合标签可用于多层循环的跳出
4. ==Goto 是调整执行位置 ==，与其它 2 个语句配合标签的结果并不相同

```go
package main

import (
    "fmt"
)

func main() {
LABEL1:
    for {
        for i := 0; i< 10; i++ {
            if i > 3 {
                break LABEL1
            }
        }
    }
    fmt.Println("ok")
}
//ok
```

```go
package main

import (
    "fmt"
)

func main() {
//死循环
//LABEL1:
//  for {
//      for i := 0; i< 10; i++ {
//          if i > 3 {
//              goto LABEL1
//          }
//      }
//  }
//  fmt.Println("ok")

    for {
        for i := 0; i< 10; i++ {
            if i > 3 {
                break LABEL1
            }
        }
    }
LABEL1:
    fmt.Println("ok")
}
//报错
//.\temp7.go:33:11: break label not defined: LABEL1
//.\temp7.go:37:1: label LABEL1 defined and not used
```

```go
package main
import (
    "fmt"
)

func main() {
LABEL1:
        for i := 0; i< 10; i++ {
            for {
                continue LABEL1
                fmt.Println(i)
            }
        }
    fmt.Println("ok")
}
//ok
```

将下面的 continue 替换成 goto，程序运行的结果还一样吗？

```go
package main
import "fmt"
func main() {
LABEL1:
        for i := 0; i< 10; i++ {
            for {
                fmt.Println(i)
                continue LABEL1
            }
        }
    fmt.Println("ok")
}
0
1
2
3
4
5
6
7
8
9
ok
//continue改成goto，会一直是0
```