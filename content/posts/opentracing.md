---
title: opentracing---分布式链路追踪
date: 2020-03-01 11:59:55
tags: [opentracing]
---


## 分布式链路追踪
追踪程序的调用链，统计调用过程中的一些基本信息，如 SQL 执行时间， http 响应时间


<!--more-->

## opentracing  / jaegertracing
这个2个项目 都是CNCF的成员，已经是大规模在使用中的项目， 我们今天主要看下jaegertracing 是如何使用的

[https://github.com/jaegertracing/jaeger](https://github.com/jaegertracing/jaeger)这个repo下面有个基本的例子，**Hot R.O.D. - Rides on Demand**  具体实现代码位置 [https://github.com/jaegertracing/jaeger/tree/master/examples/hotrod](https://github.com/jaegertracing/jaeger/tree/master/examples/hotrod)


先来直观感受下trace的链路追踪
![jaeger.png](https://lin19999.oss-cn-beijing.aliyuncs.com/jaeger.png)
![1](https://lin19999.oss-cn-beijing.aliyuncs.com/jaeger1.png)


## Jaeger 的组件
Jaeger 的组成部分有

1. Agent : 分服务一起部署的服务，负责收集服务产生的日志。
2. Collector : 收集agent 上面的日志，  同时将这些日志，推送日志的持久化存储中心。
3. Ingester ： 数据存储后端（可以用elasticsearch/Cassandra）(这个不是必须的，如果你的数据量很大，那么你可以先将collector的数据推送到kafa，这样的流处理框架，然后再同个这个组件同步到存储)，我把这个定义为存储端。
4. Query: 一个可视化的 数据搜素展示界面（如果你存储层用了elasticsearch,那么也可以用kibana来查看数据）。


## 实战

### 拉去镜像
 - 我用微软镜像加速拉去


```text

   docker pull dockerhub.azk8s.cn/library/elasticsearch:7.6.0

   docker pull dockerhub.azk8s.cn/jaegertracing/jaeger-agent
   docker tag dockerhub.azk8s.cn/jaegertracing/jaeger-agent:latest jaegertracing/jaeger-agent:latest

   docker pull dockerhub.azk8s.cn/jaegertracing/jaeger-ingester
   docker tag  dockerhub.azk8s.cn/jaegertracing/jaeger-ingester:latest  jaegertracing/jaeger-ingester:latest

   docker pull dockerhub.azk8s.cn/jaegertracing/jaeger-collector

   docker tag dockerhub.azk8s.cn/jaegertracing/jaeger-collector  jaegertracing/jaeger-collector 

   docker pull dockerhub.azk8s.cn/jaegertracing/jaeger-query
   docker tag dockerhub.azk8s.cn/jaegertracing/jaeger-query:latest jaegertracing/jaeger-query

```
### 部署服务 

- 1.首先部署一个存储后端-ElasticSearch

```text
 docker run -d  -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" --name elasticsearch  docker.elastic.co/elasticsearch/elasticsearch:7.6.0

```

通过方位 http://localhost:9200 可以看看部署是否正常
- 2. 部署一个Kibana

```text
 docker pull kibana:7.6.0

docker run -d --link elasticsearch:elasticsearch -p 5601:5601 kibana:7.6.0

//  --link XXXXX:elasticsearch -p 5601:5601 kibana:7.6.0, 这里的XXX 代表上一步生成的name 或者container id.

kibana UI： http://localhost:5601, 通过访问这个可以看下服务是否启动

```
https://zhuanlan.zhihu.com/p/66534611

- 3. 部署agent 


```text


docker run \
  --rm \
  -p5775:5775/udp \
  -p6831:6831/udp \
  -p6832:6832/udp \
  -p5778:5778/tcp \
  --name agent \
  -d \
  jaegertracing/jaeger-agent:latest \
  --reporter.grpc.host-port=192.168.67.1:14250


```

- 4. 部署collector

collector 

```text

docker run -d -e SPAN_STORAGE_TYPE=elasticsearch -e ES_SERVER_URLS=http://47.114.111.226:9200 -p 14250:14250/tcp -p 14269:14269/tcp    --name jaeger-collector  jaegertracing/jaeger-collector

```


- 5. 部署query

query

```text

docker run --name query -d -p 16686:16686 -p 16687:16687 -e SPAN_STORAGE_TYPE=elasticsearch -e ES_SERVER_URLS=http://47.114.111.226:9200 jaegertracing/jaeger-query 

```

查看服务在： http://localhost:16686


组件都部署完成了， 我们接下来写一个服务，来测试下


### write code


```go

// gin-server.go

package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/opentracing/opentracing-go"
	"github.com/opentracing/opentracing-go/ext"
	"github.com/uber/jaeger-client-go"
)

// 初始化 opentracing 的配置，配置agent 的地址
func init() {
	const serviceName = "gin-server"
	// init open tracing
	udpTransport, _ := jaeger.NewUDPTransport("localhost:6831", 1024)
	reporter := jaeger.NewRemoteReporter(udpTransport)
	sampler := jaeger.NewConstSampler(true)
	tracer, _ := jaeger.NewTracer(serviceName, sampler, reporter)
	opentracing.SetGlobalTracer(tracer)
}

func main() {

	router := gin.Default()
	router.Use(OpenTracing())
	router.GET("/", HealthGET)

	if err := router.Run(":8090"); err != nil {
		panic(err)
	}

}

// HealthGET 一个简单的服务，访问这个服务的时候会产生日志
func HealthGET(c *gin.Context) {
	client := http.Client{}
	req, _ := http.NewRequest("GET", "http://example.com/", nil)

	span := opentracing.SpanFromContext(c.Request.Context())
	fmt.Println((span.Context()))
	opentracing.GlobalTracer().Inject(span.Context(), opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(req.Header))

	client.Do(req)
}

// OpenTracing 一个中间件，
// 注入一个span ,追踪耗时
func OpenTracing() gin.HandlerFunc {
	return func(c *gin.Context) {
		wireCtx, _ := opentracing.GlobalTracer().Extract(
			opentracing.HTTPHeaders,
			opentracing.HTTPHeadersCarrier(c.Request.Header))

		serverSpan := opentracing.StartSpan(c.Request.URL.Path,
			ext.RPCServerOption(wireCtx))
		defer serverSpan.Finish()
		c.Request = c.Request.WithContext(opentracing.ContextWithSpan(c.Request.Context(), serverSpan))
		c.Next()
	}
}



```


### run code

```go

go run gin-server.go 

```
运行之后你通过kiabna 会看到2个index

```text

jaeger-service-2020-03-02
jaeger-span-2020-03-02

``` 
创建索引你可以看到哦数据，
同时通过jaeger UI:[http://localhost:16686](http://localhost:16686) 也可以看到一个简单的例子

![1212](https://lin19999.oss-cn-beijing.aliyuncs.com/jaaa.png)


### 接下来的 

上面只是一个简单的http 服务，接下来，我会写一个涉及到数据(mysql)和存储（redis) 等各种应用，贴近实际的一个案例


## 引用
[https://opentracing.io/](https://opentracing.io/)
[https://www.jaegertracing.io/docs/1.17/](https://www.jaegertracing.io/docs/1.17/)
[https://github.com/9999-dollor/code_save/blob/master/gin_server/gin-server.go](https://github.com/9999-dollor/code_save/blob/master/gin_server/gin-server.go)
[Go微服务全链路跟踪详解](https://zhuanlan.zhihu.com/p/79419529)
[OpenTracing入门与Jaeger实现](https://zhuanlan.zhihu.com/p/34318538)
