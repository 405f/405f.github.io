---
title: debug-k8s
date: 2020-01-03 22:40:01
tags: [k8s]
---



![Alt text](https://lin19999.oss-cn-beijing.aliyuncs.com/flower.svg)

## 查看pod 状态

```bash

kubectl -n  kube-ops describe   pod runner-svzx4ujw-project-1-concurrent-0qd6hq


```

## 删除pod

kubectl -n  kube-ops delete pods runner-svzx4ujw-project-1-concurrent-0qd6hq --grace-period=0 --force


## 查看pod 中的container

kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |\
sort


## 查看pod 中container 的日志

kubectl logs <pod_name> -c <container_name>



