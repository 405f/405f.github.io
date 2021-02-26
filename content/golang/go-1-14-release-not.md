---
title: go-1.14-release-note
date: 2020-02-28 10:42:55
tags: [golang]
--- 

##  official href 
1.14 Release Note

. [https://blog.golang.org/go1.14](https://blog.golang.org/go1.14)

. [https://golang.org/doc/go1.14](https://golang.org/doc/go1.14)


## 重点变更

. Module support in the go command is now ready for production use. We encourage all users to migrate to go modules for dependency management.

. Embedding interfaces with overlapping method sets

. Improved defer performance

. Goroutines are asynchronously preemptible

. The page allocator is more efficient

. Internal timers are more efficient

<!--more-->
###  模块管理
依赖管理， 废话不说 go mod


## Embedding interfaces with overlapping method sets
允许接口重载
.[https://github.com/golang/proposal/blob/master/design/6977-overlapping-interfaces.md](https://github.com/golang/proposal/blob/master/design/6977-overlapping-interfaces.md)
举个例子
```go

type Rest interface {
	io.ReadCloser
	io.WriteCloser
}

// ReadCloser is the interface that groups the basic Read and Close methods.
type ReadCloser interface {
	Reader
	Closer
}

// WriteCloser is the interface that groups the basic Write and Close methods.
type WriteCloser interface {
	Writer
	Closer
}
```
对于上面的2个方法 都有Closer ，这在1.14 是被允许的， 相当于是方法的重载， 注意重载的方法签名要一致，下面具体写一个demo

```go
package main

import (
	"fmt"
)

type A interface {
	DoX()
	DoY()
}

type B interface {
	DoX()
}

type AB interface {
	A
	B
}

type SA struct {
	Name string
	SB
}

func (SA) DoX() {
	fmt.Println("SA-DOX")
}

func (SA) DoY() {
	fmt.Println("SA-DOY")

}

type SB struct {
	NameSB string
}

func (SB) DoY() {
	fmt.Println("SB-DOY")
}

func main() {
	//var s = SB{
	//	NameSB: "",
	//	SA:     SA{Name: "sa"},
	//}
	var b = SA{
		Name: "",
		SB:   SB{},
	}
	run(b)
}

func run(c AB) {
	c.DoX()
	c.DoY()
}


```
<!--more-->

### runtime 的修改
. [https://golang.org/doc/go1.14#runtime](https://golang.org/doc/go1.14#runtime)
主要是修改defer的性能问

