---
title: go-sync-map
tags: [golang]
---

## sync map

```text

Map is like a Go map[interface{}]interface{} but is safe for concurrent use by multiple goroutines without additional locking or coordination. Loads, stores, and deletes run in amortized constant time.

The Map type is specialized. Most code should use a plain Go map instead, with separate locking or coordination, for better type safety and to make it easier to maintain other invariants along with the map content.

The Map type is optimized for two common use cases: 
(1) when the entry for a given key is only ever written once but read many times, as in caches that only grow
(2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex.
```

我们可以学到的东西

```text

通过 read 和 dirty 两个字段将读写分离，读的数据存在只读字段 read 上，将最新写入的数据则存在 dirty 字段上
读取时会先查询 read，不存在再查询 dirty，写入时则只写入 dirty
读取 read 并不需要加锁，而读或写 dirty 都需要加锁
另外有 misses 字段来统计 read 被穿透的次数（被穿透指需要读 dirty 的情况），超过一定次数则将 dirty 数据同步到 read 上
对于删除数据则直接通过标记来延迟删除

```

作者：张凯强zkqiang
链接：https://juejin.im/post/5e784321518825493c7b7d8d
来源：掘金

## double check

这里设计到设计模式的 Singleton Pattern

```java
class Foo {
        private volatile Helper helper = null;
        public Helper getHelper() {
            if (helper == null) {
                synchronized(this) {
                    if (helper == null)
                        helper = new Helper();
                }
            }
            return helper;
        }
    }

```
上面的例子是我们搜粟double check 经常能看到，我门可以实现一个go版本的

```go

package main

import "sync"

var instance interface{}
var mux sync.Mutex

func GetSingle() interface{} {
	if instance != nil {
		return instance
	}
	// lock
	mux.Lock()
	defer mux.Unlock()
	if instance != nil {
		return instance
	}
	instance = new(int)
	return instance
}

```



### 源码记录

1. define

map


```go
type Map struct {
	mu Mutex

	read atomic.Value // readOnly 支持并发读的一个原子值

	dirty map[interface{}]*entry // 从reade 中读不到，再从这里面读取

	misses int // 记录 从read 中没有读取的次数
}
```

```go

// 一个不能被修改的变量， 保存在Map.read的具体类型
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true= 代表 dirty 中含有 不包含在m 中的内容
}

```
套娃继续看 readeonly 中的entry

```go

type entry struct {
	p unsafe.Pointer // *interface{}
}
```
p 的值有3中

1. nil  代表 这个数据已经被删除了， and  m.dirty == nil
2. expunged  代表 这个数据已经被删除了， and  m.dirty ！= nil
3. 其他情况 这个数据是合法的， 并且这个数据是同时记录到 dirty 和read 中的， 如果这个dirty 不是空的


## 数据读取的实现
```go
/***
关于这个获取value 没啥好说的，首先尝试从read中去读，如果没有度渠道，并且amended是true,那么代表
dirty 还可以再读取以下，
从dirty 中获得数据中之后，要增加下reade的错误机率，最后当错误机率，超过dirty长度，那么直接将dirty的数据
复制到read中，同时将数据drity,miss 恢复到初始值
*
**/
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

```

## 存储

```go
/****
read 中找到这个key ,并且read 对应的enrty.p==nil ，那么存储之后直接返回
否则， 并且read 对应的enrty.p 设置成==nil ，然后存储到dirty

***/

// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		e.storeLocked(&value)
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

```