---
title: grpc-server
date: 2020-02-08 16:40:52
tags: [golang]
---


#### grpc server 的源码分析
grpc 的server 是处理http 链接的入口， client 传递的数据最终都是被传递到了server 端

```go

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```
先创建一个server对象， 并对其添加控制， 如最大接受的消息长度
然后注册protocal buffer 的定义文件

