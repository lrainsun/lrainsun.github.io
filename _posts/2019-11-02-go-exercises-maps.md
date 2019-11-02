---
layout: post
title:  "Go Map的练习"
date:   2019-11-02 09:47:00 +0800
categories: Go
tags: Go-Exercises
excerpt: Go Map的练习
mathjax: true
typora-root-url: ../

---

## 练习：map

实现 `WordCount`。它应当返回一个含有 `s` 中每个 “词” 个数的 map。函数 `wc.Test` 针对这个函数执行一个测试用例，并输出成功还是失败。

你会发现 [strings.Fields](http://golang.org/pkg/strings/#Fields) 很有帮助。

```go
package main

import (
	"fmt"
	"strings"
)

func WordCount(s string) map[string]int {
	ss := strings.Fields(s)
	m := make(map[string]int)
	for i := 0; i < len(ss); i++ {
		m[ss[i]]++
	}
	return m
}

func main() {
	Test(WordCount)
}



// Test runs a test suite against f.
func Test(f func(string) map[string]int) {
	ok := true
	for _, c := range testCases {
		got := f(c.in)
		if len(c.want) != len(got) {
			ok = false
		} else {
			for k := range c.want {
				if c.want[k] != got[k] {
					ok = false
				}
			}
		}
		if !ok {
			fmt.Printf("FAIL\n f(%q) =\n  %#v\n want:\n  %#v",
				c.in, got, c.want)
			break
		}
		fmt.Printf("PASS\n f(%q) = \n  %#v\n", c.in, got)
	}
}

var testCases = []struct {
	in   string
	want map[string]int
}{
	{"I am learning Go!", map[string]int{
		"I": 1, "am": 1, "learning": 1, "Go!": 1,
	}},
	{"The quick brown fox jumped over the lazy dog.", map[string]int{
		"The": 1, "quick": 1, "brown": 1, "fox": 1, "jumped": 1,
		"over": 1, "the": 1, "lazy": 1, "dog.": 1,
	}},
	{"I ate a donut. Then I ate another donut.", map[string]int{
		"I": 2, "ate": 2, "a": 1, "donut.": 2, "Then": 1, "another": 1,
	}},
	{"A man a plan a canal panama.", map[string]int{
		"A": 1, "man": 1, "a": 2, "plan": 1, "canal": 1, "panama.": 1,
	}},
}
```

