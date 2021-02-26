---
title:  golang defer 的一些总结
date: 2020-02-02 17:05:44
tags: [golang]
---

### defer 和return 

return  中返回的值 是不受defer 中的逻辑影响的，具体看个例子, return 是先执行的

```go

package main

import "fmt"

func record() int {
	var counter = 10 // counter = 10

	defer func() {
		counter += 10
		fmt.Println("defer中执行counter=", counter) // 这里的counter 不影响return 返回的
	}()
	return counter // counter = 10
}

func main() {
	res := record()
	fmt.Println("最后返回的结果：", res)
}

/****

defer中执行counter= 20
最后返回的结果： 10

*/


```


### defer 中传递参数

```go

package main

import "fmt"

func add(a, b int) int {
	sum := a + b
	fmt.Println("sum=", sum, "a=", a, "b=", b)
	return sum
}
func main() {

	//var whatever [5]struct{}
	a, b := 1, 2

	defer add(a, b)

	defer func() {
		add(a, b)
	}() // a,b 被立即拷贝到函数执行的地方

	defer func(a, b int) {
		fmt.Println("defer func")
		add(a, b)
		fmt.Println("-------")
	}(a, b) // 外面，a,b 操作会影响 最后的传递值， a=11, b= 22

	a += 10
	b += 20

	defer add(a, b)
}

/****
sum= 33 a= 11 b= 22
defer func
sum= 3 a= 1 b= 2
-------
sum= 33 a= 11 b= 22
sum= 3 a= 1 b= 2


*/



```
