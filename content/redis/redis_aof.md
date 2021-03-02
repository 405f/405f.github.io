---
title: "Redis_aof_rdb"
date: 2021-02-27T11:30:24+08:00
draft: true
---
### run redis

- 
```sh

 docker run --name redis -v ~/test:/data/ -p 6379:6379 -d redis redis-server --appendonly yes



docker run -p 6379:6379 --name redis -v /home/lbq/Documents/dtest11.github.io/content/redis/redis.conf:/usr/local/etc/redis/redis.conf -v /home/lbq/Documents/dtest11.github.io/data:/data -d redis


```
- [redis.conf](https://redis.io/topics/persistence) 添加
```conf
appendonly yes
```

## aof 

