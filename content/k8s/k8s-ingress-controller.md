---
title: k8s ingress controller 实践
date: 2020-02-27 17:02:29
tags: [k8s]
---


#### k8s 集群的环境是1.7， 三台aliyun ECS

### 首先部署了ingress controller 

使用kubernate-ingress-nginx [https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx),在这个repo 中我们看到这个目录中``` ingress-nginx/deploy/static/ ``` 已经有一部分的yaml 文件， 我们首先部署一个Deployment（nginx-ingress-controller）,[https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml)

 
执行部署命令 ```kubectl create  -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml```, 命令完成之后再次查看下是否成功

```text
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
``` 
输出
```text

NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx   nginx-ingress-controller-7f74f657bd-s6gft   1/1     Running   0          107s

```

### 部署ingress-nginx-controller service

通过上面的2个步骤，只是将ingress-nginx-controller 部署到了k8s 集群中， 并没有暴露出来， 接下来我们使用service 暴露出，


```yaml

### service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---


```

部署service 

```text

 kubectl create -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml

```

检查service 状态

```text

 kubectl get svc -n ingress-nginx

 ```
 输出如下：
 ```text
 NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.100.59.152   <none>        80:31172/TCP,443:32443/TCP   44s


 ```


 ### 部署一个简单的服务测试
 部署一个简单的测试程序， 这里我使用


 ```yaml
 ### nginx.yaml

apiVersion: v1
kind: Service
metadata:
  name: backend-nginx
  namespace: default
spec:
  selector:
    app: backend-nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: backend-nginx-deploy
spec:
  replicas: 5
  selector: 
    matchLabels:
      app: backend-nginx
  template:
    metadata:
      labels:
        app: backend-nginx
    spec:
      containers:
      - name: backend-nginx
        image: nginx
        ports:
        - name: httpd
          containerPort: 80

 ```

 部署一个nginx 服务
 ```text
 kubectl create  -f nginx.yaml
 ```
 查看pod 和service 部署是否正常

 ```text
  kubectl get pod,svc -n default
  ```
  输出

```text

NAME                                        READY   STATUS    RESTARTS   AGE
pod/backend-nginx-deploy-5d77c8b4cd-bjrm8   1/1     Running   0          14m
pod/backend-nginx-deploy-5d77c8b4cd-fssjp   1/1     Running   0          14m
pod/backend-nginx-deploy-5d77c8b4cd-ndvbj   1/1     Running   0          14m
pod/backend-nginx-deploy-5d77c8b4cd-prvvr   1/1     Running   0          14m
pod/backend-nginx-deploy-5d77c8b4cd-rqcht   1/1     Running   0          14m

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/backend-nginx   ClusterIP   10.107.22.40   <none>        80/TCP    14m
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP   27h

```
在mster 节点上根据clusterIP 访问 ``` curl 10.107.22.40:80``` 可以看到 nginx 的欢迎界面

```text

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

### 部署一个ingress 暴露该nginx  服务，使其通过公网可以访问

注意这个 ```myblog.com``` 是一个域名解析， 由于我是在aliyun Ecs 上临时搭建的K8s集群， 没有申请域名， 因此我是将本地的ip 地址 解析到了 myblog.com

具体的添加过程是
```text

### /etc/hosts
### 这个文件中添加 解析记录

47.114.111.226 myblog.com

```
在所有的机器上 都必须要添加 这个解析，包括你本地的测试的时候。


```yaml
### nginx-server-ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-nginx-server
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: myblog.com
    http:
      paths:
      - path: 
        backend:
          serviceName: backend-nginx
          servicePort: 80

```

部署ingress ```kubectl create -f  nginx-server-ingress.yaml ```

查看是否创建成功 ```  kubectl get ing -n default ```
输出： 
```text
NAME                   HOSTS        ADDRESS         PORTS   AGE
ingress-nginx-server   myblog.com   10.100.59.152   80      26s

```

查看绑定关系 ```  kubectl describe ing ingress-nginx-server -n default ```

```text

Name:             ingress-nginx-server
Namespace:        default
Address:          10.100.59.152
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  myblog.com  
                 backend-nginx:80 (10.244.1.23:80,10.244.1.24:80,10.244.2.27:80 + 2 more...)
Annotations:
  kubernetes.io/ingress.class:  nginx
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  75s   nginx-ingress-controller  Ingress default/ingress-nginx-server
  Normal  UPDATE  67s   nginx-ingress-controller  Ingress default/ingress-nginx-server


```

查看服务开放的端口```  kubectl get svc -n ingress-nginx ```
输出如下：
```text

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.100.59.152   <none>        80:31172/TCP,443:32443/TCP   58m


```

此时通过浏览器 访问[http://myblog.com:31172/](http://myblog.com:31172/) 就可以看到这个， 至于这个31172 ,你可以在部署ingress-controller那个yaml 中将它固定死