---
title: "Golang_0315_slice_array"
date: 2021-03-15T23:17:11+08:00
draft: true
---

### slice
```go

type slice struct {
	array unsafe.Pointer // 元素指针
	len   int // 长度 
	cap   int // 容量
}

```


注意，底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。

### 容量的增长
append 


