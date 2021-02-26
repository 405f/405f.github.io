---
title: go-Benchmark
tags: [golang]
---


### 如何写benchmark

我们有函数如下：
```go
package writeT

func Fib(n int) int {
	if n < 2 {
		return n
	}
	return Fib(n-1) + Fib(n-2)
}

```
<!--more-->
benchmark 的代码应该是以xxx_test.go 结尾的一个文件， 我们新建一个文件 write_bench.go_test.go
```go

package writeT

import "testing"

func BenchmarkFib10(b *testing.B) {
	// run the Fib function b.N times
	for n := 0; n < b.N; n++ {
		Fib(10)
	}
}

```

### 运行


- 指定压测的文件
```text
go test -run=XXX.go -bench=BenchmarkFib10
```

- 指定压测的函数
```text
go test  -bench=BenchmarkFib10

```
```text

go test -bench=.

```
这个命令会运行 该文件夹下test的所有Benchmark* 函数

- 指定压力测试的时间
```text
go test -bench=Fib40 -benchtime=20s
```

- 指定测试的CPU 数量
```text
go test -v -bench=.  -cpu=1,2,4,8 
```
这个会输出如下： 针对不同CPU的数量
```text
BenchmarkFib10     	 3517034	       317 ns/op
BenchmarkFib10-2   	 3697018	       321 ns/op
BenchmarkFib10-4   	 3753944	       316 ns/op
BenchmarkFib10-8   	 3694322	       315 ns/op
```


## 查看测试输出的参数

我们运行上面的例子，可以看到输出：
```text
goos: linux
goarch: amd64
pkg: code-save/write_test
BenchmarkFib10-8   	 3578236	       322 ns/op

```
具体解释一下

```text
goos: linux // 平台
goarch: amd64 // 架构
pkg: code-save/write_test // 测试目录
BenchmarkFib10-8   	 3578236	       322 ns/op

```

1. BenchmarkFib10-8 ： 测试函数-执行本次测试的CPU核数 
2. 3578236 ： 该函数执行了这么多次
3. 322 ns/op ：每次循环需要 322 纳秒


##### 引用
[https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go](https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go)