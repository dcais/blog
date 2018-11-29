---
title: 国内安装minibuke
date: 2018-11-29 21:43:03
tags: [k8s]
---

clone 阿里的 minibuke 仓库

```bash
git clone https://github.com/AliyunContainerService/minikube
```

切换到阿里云分支 aliyun-v0.30.0

```bash
mkdir $GOPATH/src/k8s.io
mv minikube $GOPATH/src/k8s.io/
cd $GOPATH/src/k8s.io/minikube
make
cp out/minikube /usr/local/bin/
chmod a+x/usr/local/bin/minikube
```
