---
title: rancher-k8s-deploy
date: 2020-01-03 23:01:27
tags: [k8s]
---



<img src="https://lin19999.oss-cn-beijing.aliyuncs.com/rancher.svg" width = "400" height = "400" alt="图片名称" align=center />

## Rancher install

如果之前安装过,需要清理 /opt/rancher 文件夹

```

docker run -d -p 80:80 -p 443:443 \
  --restart=unless-stopped \
  -v /opt/rancher:/var/lib/rancher \
  rancher/rancher:latest

``` 




## 选择 

<img src="https://lin19999.oss-cn-beijing.aliyuncs.com/rancher_dep.png" width = "400" height = "400" alt="图片名称" align=center />
