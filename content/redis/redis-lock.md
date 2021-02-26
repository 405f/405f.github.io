---
title: redis-分布式锁
tags: [redis]
---
一个简单的需求， 高并发的时候用户不能重复注册， 因为后台API， 是多个节点部署的， 我们并不是单台机器，那么怎么保证用户频繁的提交注册请求不会导致业务逻辑重复执行，（这里我们不讨论数据库设计的时候是有唯一的主键的，例如手机号码）其实很简单，我们提供一个中心化的服务，在API执行的时候先从这个服务去检查下自己需不需要执行，那么其实这个问题最后就变成了一个锁的问题。由于我们的服务是多个实例，分布部署的，是分布式，那么这个锁也就是分布式锁。

<!--more-->

### 业务需求

用户注册的时候是用手机号作为登录信息来注册，也就是提交到API的数据
```text
{ "username":"nick","mobile":"1881826189"}
```

### 使用 单节点redis 来加锁

#### setnex 的命令

```text
127.0.0.1:6379> set key value [EX seconds] [PX milliseconds] [NX|XX]

EX： 设置指定的到期时间(以秒为单位)。
PX： 设置指定的到期时间(以毫秒为单位)
NX： 仅在键不存在时设置键
XX： 只有在键已存在时才设置

SET mykey myvalue EX 60 NX  :将在键“mykey”不存在时，设置键的值，到期时间为60秒。

```
redis 提供了一个这样的命令， 我们可以使用这个命令来实现一个锁，

####  Example
```go
package lock

// redis 实现分布式锁
import (
	"time"

	"github.com/go-redis/redis/v7"
)

// RedisClientPtr redis connect client
var RedisClientPtr *redis.Client

const ExpireAfter = 60 * time.Second

func init() {
	RedisClientPtr = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})

	_, err := RedisClientPtr.Ping().Result()
	if err != nil {
		panic(err)
	}
}

// RedisLock redis 锁， 最好是每个业务的key 是有唯一性标识的
// 比如注册 register-key: register-1881826,这样子，释放锁的时候不会误删除别人设置的锁
//  Redis `SET key value [expiration] NX` command.
func RedisLock(key string) bool {
	result, err := RedisClientPtr.SetNX(key, true, ExpireAfter).Result()
	// 这种情况下 才算锁定
	if err == redis.Nil && result == true {
		return true
	}
	return false
}

// RedisUnlock redis 释放锁, 这里看自己需不需要关注删除失败的情况，
func RedisUnlock(key string) {
	RedisClientPtr.Del(key)
}


```

### 压测一下单机的性能

测试环境： 16G+8CPU+i7

-. 我们压力测试下并发条件下的性能
```go

package lock

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

func ExampleRedisLock(mobile string) {
	key := "user_register" + mobile
	// try lock
	if !RedisLock(key) {
		//fmt.Println("lock fail")
		return
	}
	// continue exec the things
	//time.Sleep(40 * time.Second)
	RedisUnlock(key)
}



//BenchmarkRedisLockParallel-8   	   80612	     13789 ns/op
// 一秒可以执行    72521.575168613， 7万多次
func BenchmarkRedisLockParallel(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mobile := rand.Int63()
			ExampleRedisLock(fmt.Sprint(mobile))
		}
	})
}


```

可以看到单纯的获取锁，释放锁 整个过程： 1秒可以执行7万次左右


.  不使用并发
```go

//BenchmarkRedisLock-8   	   29391	     38682 ns/op
// 一秒可以执行25851.817382762， 也就是2万多次
func BenchmarkRedisLock(b *testing.B) {
	for n := 0; n < b.N; n++ {
		ExampleRedisLock(fmt.Sprint(b))
	}
}


```
效率是2万次左右


### redis cluster
在上述的情况的我看到单机的性能已经是很出色的，但是还是有个问题，那就是单点， 整个系统依赖Redis，如果这个节点挂掉，那么整个系统的可用性，会收到很大的影响，如果是主干服务，那么更是不能接受的。为了解决服务的可用性，我们可以部署多台redis,那么即使其中的某些节点挂掉，，整个服务的依然是可用的。
那么当我们的机器是集群的时候，我们该如何获取锁，释放锁，可以总结成一句话 **在大多数机器上执行成功sex nx命令**,这个多数我们是 超过总节点数量的一半 （all=5,success_number >=3,那么就是成功的)

那么我们用代码描述一下整个流程，(我们直接看下redync这个库的实现过程)[https://github.com/go-redsync/redsync](https://github.com/go-redsync/redsync)
```go
// Lock locks m. In case it returns an error on failure, you may retry to acquire the lock by calling this method again.
func (m *Mutex) Lock() error {
	value, err := m.genValueFunc()
	if err != nil {
		return err
	}

	for i := 0; i < m.tries; i++ {
		if i != 0 {
			time.Sleep(m.delayFunc(i))
		}

		start := time.Now()

		n := m.actOnPoolsAsync(func(pool Pool) bool {
			return m.acquire(pool, value)
		})

		now := time.Now()
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.factor)))
		if n >= m.quorum && now.Before(until) {
			m.value = value
			m.until = until
			return nil
		}
		m.actOnPoolsAsync(func(pool Pool) bool {
			return m.release(pool, value)
		})
	}

	return ErrFailed
}


```
上面操作的过程可以详细描述为：
1.  如果不是第一次执行的话。那么先 延迟执行，具体延迟时间 根据函数的返回值确定
2.  获取开始时间
3.  并发的向配置的节点执行set nx 命令，并返回执行成功的节点数量
4.  获取当前时间
5.  计算结束时间 
第4步获得时间+（设置的过期时间 - （过期时间 * factor） 
6.  如果获得成功的机器数量大于1半+ （第5步的时间大于当前时间） == 获取成功， 否则删除预先设置的值


### redlock 使用

[https://github.com/9999-dollor/redsync/blob/master/redsync_test.go#L76](https://github.com/9999-dollor/redsync/blob/master/redsync_test.go#L76)
```go

func TryLock(key string) {
	var conn []*redis.Client
	for _, v := range redis_cluster_address {
		c := redis.NewClient(v)
		if c.Ping().Err() != nil {
			panic(c.Ping().Err())
		}
		conn = append(conn, c)
	}
	rs := New(conn)
	mutex := rs.NewMutex(key)
	err := mutex.Lock()
	if err != nil {
		panic(err)
	}
}

// BenchmarkRedsync-8   	     100	  10542296 ns/op 
// 1 秒94次
func BenchmarkRedsync(b *testing.B) {
	for n := 0; n < b.N; n++ {
		number := rand.Int63()
		TryLock(fmt.Sprint(number))
	}
}

```

### redlock 的问题

#### 锁是具有过期时间的
关于redlock 的问题，[http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) 有一个具体的论述

![redlock](http://martin.kleppmann.com/2016/02/unsafe-lock.png)

RedLock中，为了防止死锁，锁是具有过期时间的。这个过期时间被 Martin 抓住了小辫子

- 如果 Client 1 在持有锁的时候，发生了一次很长时间的 FGC 超过了锁的过期时间。锁就被释放了。
- 这个时候 Client 2 又获得了一把锁，提交数据。
- 这个时候 Client 1 从 FGC 中苏醒过来了，又一次提交数据。

RedLock 只是保证了锁的高可用性，并没有保证锁的正确性。, 目前Rust好像是没有GC


martin.kleppmann 又给出了具体的解决办法， 也就是 没有设置的value是一个单调递增的计数器，每次再设置之前判断，这个计数器是不是小于自己持有的那个，入过不是那么就放弃

#### 过期时间
在任何分布式系统中 时间都是 不可靠的，不能用时间，来做凭据，
如果某个 Redis Master的系统时间发生了错误，造成了它持有的锁提前过期被释放。

### 引用
- [https://github.com/go-redsync/redsync](https://github.com/go-redsync/redsync)
- [https://redis.io/topics/distlock](https://redis.io/topics/distlock)
- [http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)