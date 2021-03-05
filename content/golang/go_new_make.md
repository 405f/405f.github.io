---
title: "Go_new_make"
date: 2021-03-05T10:16:30+08:00
draft: true
---
### most different
- make 只能在某些特定的数据结构上使用【slice,chan,map】（var users=make(map[string]string）,new 可以在内置的任何数据类型上使用。var a = new(int)
- new 返回的是该对象的引用，（该数据在内存中的地址), make 返回的是这个数据的真实对象
- make / new 对于slice , map, chan 。map 可以指定cap,len , make(type,len,cap) 这一点new是做不到的。

### make 
```golang
make([]T, length, capacity)
make([]int, 50, 100)
```


### new
```golang
new(T)
type S struct { a int; b float64 }
new(S) 
