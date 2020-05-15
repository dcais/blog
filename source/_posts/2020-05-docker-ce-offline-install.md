---
title: docker-ce 离线安装
date: 2020-05-15 15:01:08
tags: 运维 docker
---


## 下载最新的docker-ce安装包

下载最新的repo文件 [http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo](http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo)

## 下载最新版本的 cotainerd.io,docker-ce,docker-ce-cli

下载地址：
官方：[https://download.docker.com/linux/centos/7/x86_64/stable/Packages/](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)
阿里云镜像：[http://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/](http://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/)

## 开始安装

1、添加repo：将下载好的docker-ce.repo文件拷贝到/etc/etc/yum.repos.d/下；

2、按顺序依次安装containerd.io、docker-ce-cli、container-selinux和docker-ce包

## 解决缺少依赖的问题

出现 “>=版本号”：说明你的系统上已经安装了这些包，只是这些包不是最新的，需要升级
  以policycoreutils包为例，可以在[https://pkgs.org/](https://pkgs.org/)上搜索对应的最新的rpm包。

## docker-ce proxy设置

参考 [官网文档](https://docs.docker.com/config/daemon/systemd/)
