---
layout: post
title:  "Go Reader的练习"
date:   2019-11-02 16:46:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Go Reader的练习
mathjax: true
typora-root-url: ../
---

## 练习：Reader

实现一个 `Reader` 类型，它不断生成 ASCII 字符 `'A'` 的流。

```go
package main

import (
	"fmt"
	"io"
	"os"
)

type MyReader struct{}

// TODO: Add a Read([]byte) (int, error) method to MyReader.
func (reader MyReader) Read(b []byte) (int, error) {
	i := 0
	for ; i < len(b); i++ {
		b[i] = 'A'
	}
	return i, nil
}

func main() {
	Validate(MyReader{})
}

func Validate(r io.Reader) {
	b := make([]byte, 1024, 2048)
	i, o := 0, 0
	for ; i < 1<<20 && o < 1<<20; i++ { // test 1mb
		n, err := r.Read(b)
		for i, v := range b[:n] {
			if v != 'A' {
				fmt.Fprintf(os.Stderr, "got byte %x at offset %v, want 'A'\n", v, o+i)
				return
			}
		}
		o += n
		if err != nil {
			fmt.Fprintf(os.Stderr, "read error: %v\n", err)
			return
		}
	}
	if o == 0 {
		fmt.Fprintf(os.Stderr, "read zero bytes after %d Read calls\n", i)
		return
	}
	fmt.Println("OK!")
}
```

## Readers

`io` 包指定了 `io.Reader` 接口， 它表示从数据流结尾读取。

Go 标准库包含了这个接口的[许多实现](http://golang.org/search?q=Read#Global)， 包括文件、网络连接、压缩、加密等等。

`io.Reader` 接口有一个 `Read` 方法：

```
func (T) Read(b []byte) (n int, err error)
```

`Read` 用数据填充指定的字节 slice，并且返回填充的字节数和错误信息。 在遇到数据流结尾时，返回 `io.EOF` 错误。

例子代码创建了一个 [`strings.Reader`](http://golang.org/pkg/strings/#Reader)。 并且以每次 8 字节的速度读取它的输出。