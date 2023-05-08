---
title: 单机部署k8s
tags:
  - K8s
  - 容器
categories:
  - 杂货
date: 2023-5-7 12:24:36
---

最近发现自己对同网段多主机的实验环境需求越来越高了，服务器嘛租不起，实验室主机嘛大家共用的不太放得开手脚，虚拟机嘛数量起来有点吃不消，那最好的办法就是容器了。按道理说，Docker就足够满足我的需求了，但是学都学了不如把K8s一起学了。这篇文章算个开头，记录一下我在单机上部署K8s的过程，以及遇到的问题和解决方案。  

<!-- more -->

整个流程大多参考[这篇博客](https://www.hyhblog.cn/2021/02/17/deployment-manual-k8s-for-newbee/#18)（感谢分享）和[官方文档](https://kubernetes.io/zh-cn/docs/home/)。  

> **本次安装环境：**  
> \* 腾讯云轻量级服务器2核+2G  
> \* 操作系统Ubuntu Server 22.04 LTS 64bit, 内核Linux 5.15.0-56-generic
> \* 目标版本：1.25.9

# 1 环境检查和准备

## 1.1 安装要求  

在[官方文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)上可以找到安装环境要求，简单列一下：  
- Linux主机
- 2核CPU+2G内存及以上
- 集群中的所有机器的网络彼此均能相互连接（公网和内网都可以）
- 节点之中不可以有重复的主机名、MAC 地址或 product_uuid
- K8s所需端口
- 禁用交换分区（1.22以后的版本支持交换分区了，这个要求可以忽略）

接下来逐条检查。

## 1.2 检查CPU和内存

``` bash
# 查看系统
```
