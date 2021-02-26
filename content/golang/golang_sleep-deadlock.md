---
title: asleep-deadlock!
date: 2020-02-09 12:07:24
tags: [golang]
---
写goroutines 偶尔会碰到 
```
fatal error: all goroutines are asleep - deadlock!
```

#### deaklock 的定义
```
A deadlock happens when a group of goroutines are waiting for each other and none of them is able to proceed.
```
1. 首先channel 是用来在groutines中用来传递信息的，那么在单个grountine中讨论channel 是没有意义的


#### channel 的定义
我们声明一个chnnel如如下的方式
```go
messages :=make(chan string)

```
对于channel 我们分为有缓冲的，没有缓冲的， 单向(只读，只写)
```go

message := make(chan string) // 无缓冲
message :=make(chan string, 10) // 有10 个缓冲的通道
message :=make( chan <-int) // 只能写
message := make( <-chan int) //只能读取的channel
```

1. 有缓冲的通道， 我们可以往里面写入数据，不会造成阻塞， 或者死锁

```go


package main

import "fmt"

func main() {
	messages := make(chan string, 10)
	messages <- "hello world"
	messages <- "hello world again"
	messages <- "hello world next time"
	fmt.Println("ending the show")
}

// OUTPUT:
// ending the show
```

2. 单向只能写入的chanel 
这个我们可以看个源码中的例子
[https://github.com/golang/go/blob/master/src/os/signal/signal.go#L105](https://github.com/golang/go/blob/master/src/os/signal/signal.go#L105)

```go

func Notify(c chan<- os.Signal, sig ...os.Signal) {
	if c == nil {
		panic("os/signal: Notify using nil channel")
	}

	watchSignalLoopOnce.Do(func() {
		if watchSignalLoop != nil {
			go watchSignalLoop()
		}
	})

	handlers.Lock()
	defer handlers.Unlock()

	h := handlers.m[c]
	if h == nil {
		if handlers.m == nil {
			handlers.m = make(map[chan<- os.Signal]*handler)
		}
		h = new(handler)
		handlers.m[c] = h
	}

	add := func(n int) {
		if n < 0 {
			return
		}
		if !h.want(n) {
			h.set(n)
			if handlers.ref[n] == 0 {
				enableSignal(n)
			}
			handlers.ref[n]++
		}
	}

	if len(sig) == 0 {
		for n := 0; n < numSig; n++ {
			add(n)
		}
	} else {
		for _, s := range sig {
			add(signum(s))
		}
	}
}


```

3.  单向 只能读取的通道
## TODO 


### groutines 的数据是如何通过channel 传递的

#### 需要重新查找的内容
```go

sg := c.recvq.dequeue(); sg
```


#### channel的具体实现


1.  channel的定义

.  hchan 

```go

type hchan struct {
	qcount   uint           // total data in the queue, 通道中现在拥有的数据数量
	dataqsiz uint           // size of the circular queue //环形队列的数据大小
	buf      unsafe.Pointer // points to an array of dataqsiz elements //指向dataqsiz元素类型大小的数组
	elemsize uint16
	closed   uint32  // 通道是否关闭
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters 接受而阻塞的等待队列
	sendq    waitq  // list of send waiters  发送而阻塞的发送队列
	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex // 对这个结构体加锁，在修改字段的时候
}

```

. waitq

```go

type waitq struct {
	first *sudog
	last  *sudog
}

```

. sudog

```go

type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.
	g *g
	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool
	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // data element (may point to stack)
	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.
	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}
```

### 引用

[go夜读](https://github.com/developer-learning/night-reading-go/issues/450)

https://zhuanlan.zhihu.com/p/59782261


[https://gist.github.com/YumaInaura/8d52e73dac7dc361745bf568c3c4ba37](https://gist.github.com/YumaInaura/8d52e73dac7dc361745bf568c3c4ba37)

