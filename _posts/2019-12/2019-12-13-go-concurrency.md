---
layout: post
title:  "Go并发concurrency"
date:   2019-12-13 16:45:00 +0800
categories: Go
tags: Go-Language
excerpt: Go并发concurrency
mathjax: true
typora-root-url: ../
---

# 并发concurrency

1. goroutine只是官方实现的超级“线程池”而已，每个实例4-5kb的栈内存占用和由于实现机制而大幅减少的创建和销毁开销，是制造Go号称的高并发的根本原因。
2. 并发不是并行
3. 并发主要由切换时间片来实现“同时”运行，在并行则是直接利用多核实现多线程的运行，但Go可以设置使用核数，以发挥多核计算机的能力。
4. Goroutine奉行通过通信来共享内存，而不是共享内存来通信

```go
package main

import (
   "fmt"
   "time"
)

func main() {
   go Go()
   time.Sleep(2 * time.Second)  //主程序等待2s,使得goroutine可以运行
}

func Go() {
   fmt.Println("gogogo!!!")
}
//gogogo
```

# Channel

1. Channel是goroutine沟通的桥梁，大都是阻塞同步的
2. 通过make创建，close关闭
3. Channel是引用类型
4. 可以使用for range 来迭代不断操作channel
5. 可以设置单向（只能读或只能写）或双向通道
6. 可以设置缓存大小，在未被填满前不会发生阻塞

```go
package main

import (
   "fmt"
)

func main() {
   c := make(chan bool)  //bool是存储的数据类型
   go func() {   //匿名函数
      fmt.Println("gogogo")
      c <- true         //把true传入channel
   }()
   <- c    //从channel中取出值
}

func Go() {
   fmt.Println("gogogo!!!")
}
//gogogo
```

使用for range 来迭代不断操作channel，直接channel被close

```go
package main

import (
   "fmt"
)

func main() {
   c := make(chan bool)  //bool是存储的数据类型
   go func() {
      fmt.Println("gogogo")
      c <- true         //把true传入channel
      close(c)  //如果不close，fatal error: all goroutines are asleep - deadlock!

   }()
   for v := range c {  //迭代不断操作channel，直到channel close
      fmt.Println(v)
   }
}
//gogogo
//true
```

如果有缓存，在缓存未被填满前不会发生阻塞

* 有缓存是异步的
* 无缓存是同步阻塞的

```go
package main

import (
   "fmt"
)

func main() {
   c := make(chan bool, 1)  //bool是存储的数据类型，缓存为1
   go func() {
      fmt.Println("gogogo")
      <- c
   }()
   c <- true
}
//这里不会输出gogogo，因为有缓存，当true放到channel里的时候，缓存没满，所以不会发生阻塞，爱读不读，就不会阻塞等待goroutine完成，所以就不会输出gogogo
```

```go
package main

import (
   "fmt"
)

func main() {
   c := make(chan bool)  //bool是存储的数据类型，缓存为1
   go func() {
      fmt.Println("gogogo")
      <- c
   }()
   c <- true
}
//这里当没有缓存的时候，把true放到channel里，就会阻塞，等待有人把它读走，会等待goroutine打印出gogogo，再把channel中内容读走
```

多核的时候，任务的分配是不一定的，index=9的任务有可能先执行，而一旦index=9的任务执行完毕，main函数就退出了，所以每次的输出结果都是不一样的，而且不一定可以打印出所有的结果

```go
package main

import (
   "fmt"
   "runtime"
)

func main() {
   runtime.GOMAXPROCS(runtime.NumCPU())  //多核
   c := make(chan bool)
   for i := 0; i < 10; i++ {
      go Go(c, i)
   }
   <- c
}

func Go(c chan bool, index int)  {
   a := 1
   for i := 0; i < 100000000; i++ {
      a += i
   }
   fmt.Println(index, a)

   if index == 9 {
      c <- true
   }
}
//0 4999999950000001
//2 4999999950000001
//9 4999999950000001
```

两种解方案：1 通过缓存channel

```go
package main

import (
   "fmt"
   "runtime"
)

func main() {
   runtime.GOMAXPROCS(runtime.NumCPU())  //多核
   c := make(chan bool, 10)
   for i := 0; i < 10; i++ {
      go Go(c, i)
   }
   for i := 0; i < 10; i ++ {
      <- c
   }
}

func Go(c chan bool, index int)  {
   a := 1
   for i := 0; i < 100000000; i++ {
      a += i
   }
   fmt.Println(index, a)

   c <- true
}
//这样每次都会输出10次打印，只不过顺序不一定
```

2 同步waitgroup，waitgroup可以添加需要完成的任务数，每完成一个任务就可以标记done，任务数就会往下减，等待任务都完成，就退出

```go
package main

import (
   "fmt"
   "runtime"
   "sync"
)

func main() {
   runtime.GOMAXPROCS(runtime.NumCPU())  //多核
   wg := sync.WaitGroup{}
   wg.Add(10)
   for i := 0; i < 10; i++ {
      go Go(&wg, i)
   }
   wg.Wait()
}

func Go(wg *sync.WaitGroup, index int)  {
   a := 1
   for i := 0; i < 100000000; i++ {
      a += i
   }
   fmt.Println(index, a)

   wg.Done()
}
```

# Select

1. 可处理一个或多个channel的发送与接收
2. 同时有多个可用的channel时按随机顺序处理
3. 可用空的select来阻塞main函数
4. 可设置超时

select处理接收

```go
package main

import (
   "fmt"
)

func main() {
   c1, c2 := make(chan int), make(chan string)
   o := make(chan bool, 2)

   go func() {
      for {
         select {   //通过for, select组合来实现不断的信息接收和发送操作
         case v, ok := <-c1:
            if !ok {  //c1关闭后，Ok就是false了，否则一直是true
               o <- true
               break
            }
            fmt.Println("c1", v)

         case v, ok := <-c2:
            if !ok {
               o <- true
               break
            }
            fmt.Println("c2", v)
         }
      }
   }()

   c1 <- 1
   c2 <- "hi"
   c1 <- 3
   c2 <- "hello"

   close(c1)
   //close(c2)  这里，关闭c1 or c2都是可以的，关闭任意一个，都会退出

   <- o
}
//c1 1
//c2 hi
//c1 3
//c2 hello
```

select处理发送

```go
package main

import (
   "fmt"
)

func main() {
   c := make(chan int)

   go func() {
      for v := range c {
         fmt.Println(v)
      }
   }()

   for {
      select {
      case c <- 0:
      case c <- 1:
      }
   }
}
//会随机输出1 0 
```

超时退出

```go
package main

import (
   "fmt"
   "time"
)

func main() {
   c := make(chan bool)
   select {
   case v := <- c:
      fmt.Println(v)
   case <- time.After(3 * time.Second):
      fmt.Println("timeout")  //因为我们没有往c里放入任何数据，所以3s之后会超时退出
   }
}
//timeout
```

创建一个goroutine，与主线程按顺序相互发送信息若干次并打印

```go
package main

import (
   "fmt"
)

var c chan string

func Pingpong()  {
   i := 0
   for {
      fmt.Println(<- c)
      c <- fmt.Sprintf("From Pingpong: Hi, #%d", i)
      i++
   }
}

func main() {
   c = make(chan string)
   go Pingpong()
   for i := 0; i < 10; i++ {
      c <- fmt.Sprintf("From main: Hi, #%d", i)
      fmt.Println(<- c)
   }
}
//From main: Hi, #0
//From Pingpong: Hi, #0
//From main: Hi, #1
//From Pingpong: Hi, #1
//From main: Hi, #2
//From Pingpong: Hi, #2
//From main: Hi, #3
//From Pingpong: Hi, #3
//From main: Hi, #4
//From Pingpong: Hi, #4
//From main: Hi, #5
//From Pingpong: Hi, #5
//From main: Hi, #6
//From Pingpong: Hi, #6
//From main: Hi, #7
//From Pingpong: Hi, #7
//From main: Hi, #8
//From Pingpong: Hi, #8
//From main: Hi, #9
//From Pingpong: Hi, #9
```

