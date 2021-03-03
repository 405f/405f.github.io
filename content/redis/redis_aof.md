---
title: "Redis_aof_rdb"
date: 2021-02-27T11:30:24+08:00
tags: [redis]
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
- append
- write file
- sync

## 命令追加
当AOF 打开的时候,服务器执行的命令,都会以协议的格式写到aof_buf 缓冲区的末尾

## rewrite


## flushAppendOnlyFile 
检查配置的, aof的刷盘机制, 是
```txt

configEnum aof_fsync_enum[] = {
    {"everysec", AOF_FSYNC_EVERYSEC},
    {"always", AOF_FSYNC_ALWAYS},
    {"no", AOF_FSYNC_NO},
    {NULL, 0}
};
```
- always 直接call fsync() 刷盘
- everysec 通过比较server.unixtime > aof_last_fsync   开启后台任务,flush disk


## load aof file
- redis 会启动一个fake client 来读去appendonly.aof, 
- aof content 
```txt
$6
SELECT
$1
0
*3
$3
set
$1
0
$1
0
*3
$3
set
$1
```

1. [loadDataFromDisk](https://github.com/redis/redis/blob/fd052d2a86b1a9ace29abf2868785f0b4621b715/src/server.c#L5719)
2. 检查aof 是否打开,优先加载aof, 负责加载rdb文件[loadAppendOnlyFile](https://github.com/redis/redis/blob/fd052d2a86b1a9ace29abf2868785f0b4621b715/src/aof.c#L739) 

```c
int loadAppendOnlyFile(char *filename) {
     // open file
    server.aof_state = AOF_OFF;

    fakeClient = createFakeClient(); // create 
   /* Read the actual AOF file, in REPL format, command by command. */
    while(1) {
        // read file line by line
        /* Run the command in the context of a fake client */
        fakeClient->cmd = cmd;
        if (fakeClient->flags & CLIENT_MULTI &&
            fakeClient->cmd->proc != execCommand)
        {
            queueMultiCommand(fakeClient);
        } else {
            cmd->proc(fakeClient); // 执行命令
        }
}
```