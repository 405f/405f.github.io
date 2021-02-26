---
title: k8s-fronted-backend
date: 2020-02-27 21:54:39
tags: [k8s]
---

# TODO 
## 微软镜像加速下载
[https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md](https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md)


dockerhub.azk8s.cn/google-samples/hello-go-gke:1.0
gcr.azk8s.cn/<repo-name>/<image-name>:<version>	
gcr.azk8s.cn/google-samples/hello-go-gke:1.0


### 部署后端服务

.   deployment
```yaml


## service/access/hello.yaml 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: backend
spec:
  selector:
    matchLabels:
      app: hello
      tier: backend
      track: stable
  replicas: 2
  template:
    metadata:
      labels:
        app: hello
        tier: backend
        track: stable
    spec:
      containers:
        - name: hello
          image: "nginx"
          ports:
            - name: http
              containerPort: 80

```



. service 

```yaml

## service/access/hello-service.yaml 

apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: backend
spec:
  selector:
    app: hello
    tier: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: http

```