---
title: Go中并发控制
date: 2020-01-15 13:55:29
tags: [golang]
---


# goroutines 
https://seancarpenter.io/posts/concurrency_in_go/

https://www.cnblogs.com/sunsky303/p/11077634.html

```go

package main

import (
	"fmt"
	"time"
)

func doWork(work int) {
	time.Sleep(5 * time.Second)
	fmt.Printf("Work done %d \n", work)
}

func main() {
	go doWork(1)
	go doWork(2)
	doWork(3)
	time.Sleep(10 * time.Second)
}


```
上面的代码输出完全是随机的,3,2,1 的次序 随机的
```
Work done 3 
Work done 2 
Work done 1 
```

# channel 

channel 是 goroutine 用来通信的方式, 同时我们也可以使用channel 来控制grountine 的执行次序
channel 是引用传递的

channel 的类别
-  有缓冲的通道(不会阻塞当前代码的执行) make(chan int , 10)
-  没有缓冲通道的(阻塞当前代码的执行) make(chan int)

- 单项读取通道( <- chan int)
- 写入通道 (chan int <-)

###### example

```go

package main

import (
	"fmt"
	"time"
)

func doWork(ch chan string, work int) {
	time.Sleep(5 * time.Second)
	ch <- fmt.Sprintf("Work done %d \n", work)
}

func main() {
	ch := make(chan string)
	go doWork(ch, 1)
	go doWork(ch, 2)
	x := <-ch
	y := <-ch
	fmt.Println(x)
	fmt.Println(y)
}


```


# Waitgroup


```go

package main

import (
	"fmt"
	"sync"
	"time"
)

func doWork(ch chan string, work int, wait *sync.WaitGroup) {
	ch <- fmt.Sprintf("Work done %d \n", work)
	wait.Done()

}

func main() {
	var ch = make(chan string, 100)
	var wait sync.WaitGroup
	for i := 0; i < 10; i++ {
		wait.Add(1)
		go doWork(ch, i, &wait)
	}
	wait.Wait() // 等待所有的grountine 完成
	fmt.Println("---完成----")
}

```



# select

select 读取 case 是随机的,这个结果是不可预料的

```go

package main

import (
	"fmt"
	"time"
)

func doWork(ch chan string, work int) {
	ch <- fmt.Sprintf("Work done %d \n", work)

}

func main() {
	var ch = make(chan string, 100)
	go doWork(ch, 0)
	tick := time.After(10 * time.Second)

	for {
		select {
		case <-ch:
			fmt.Println("hello world")
		case <-tick:
			fmt.Println("定时器到达")
			break
		}

	}
	fmt.Println("---完成----")
}

```

# Mutex 
- Mutex  : A Mutex must not be copied after first use.(排它锁, 一个获得之后,其他的groutine都在等待状态)
- RWMutex :可以随便读，多个goroutine同时读

# TODO  RWMutex ,Mutex 底层是在呢么实现的, 怎么保证实现锁
同步原语言, 可以保证 异步是 按照预期的执行, 比如数字的加减,
```go

package main

import (
	"fmt"
	"sync"
)

type safeNum struct {
	num   int
	mutex sync.RWMutex
}

func doWork(ch chan string, work int) {
	ch <- fmt.Sprintf("Work done %d \n", work)

}

func increment(numberPtr *safeNum, waitGroupPtr *sync.WaitGroup) {
	numberPtr.mutex.Lock()
	numberPtr.num++
	numberPtr.mutex.Unlock()
	waitGroupPtr.Done()
}
func main() {
	var (
		sf   = safeNum{num: 0}
		wait = sync.WaitGroup{}
	)

	for i := 0; i < 1000; i++ {
		wait.Add(1)
		go increment(&sf, &wait)
	}
	wait.Wait()
	fmt.Println("---完成----", sf.num)
}


```

# atomic

atomic 是直接的CPU指令
```go

package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

var Number int64

func increment(waitGroupPtr *sync.WaitGroup) {
	// numberPtr.mutex.Lock()
	atomic.AddInt64(&Number, 1)
	// numberPtr.mutex.Unlock()
	waitGroupPtr.Done()
}
func main() {
	var (
		wait = sync.WaitGroup{}
	)
	for i := 0; i < 1000; i++ {
		wait.Add(1)
		go increment(&wait)
	}
	wait.Wait()
	fmt.Println("---完成----", Number)
}


```