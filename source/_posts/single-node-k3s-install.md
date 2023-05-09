---
title: 单机部署k3s
tags:
  - K8s
  - K3s
  - Docker
  - 容器
categories:
  - 杂货
date: 2023-05-07 12:24:36
---


最近发现自己对同网段多主机的实验环境需求越来越高了，服务器嘛多了租不起，实验室主机嘛大家共用的不太放得开手脚，虚拟机嘛数量起来有点吃不消，那还是在自己的服务器上部署容器编排平台吧。  

<!-- more -->

单节点没有过多的要求，而且租的服务器没有太多的资源（2核2G），所以就不选择用[K8s](https://kubernetes.io/)了，[K3s](https://k3s.io/)更轻量简单。

文章简单记录一下自己安装部署和配置的流程。 

> **本次安装环境：**  
> \* 腾讯云轻量级服务器2核+2G  
> \* 操作系统Ubuntu Server 22.04 LTS 64bit, 内核Linux 5.15.0-56-generic  
> \* 目标版本：1.25.9  

# 1 Docker

K3s包含并且默认[containerd]作为容器运行时，但是就我的体验来说docker还是更好用一些，K3s也提供了使用Docker作为容器运行时的[安装方法](https://docs.k3s.io/zh/advanced#%E4%BD%BF%E7%94%A8-docker-%E4%BD%9C%E4%B8%BA%E5%AE%B9%E5%99%A8%E8%BF%90%E8%A1%8C%E6%97%B6)。所以在安装K3s前先安装一下Docker Engine，参考[官方文档](https://docs.docker.com/engine/install/)。  

## 1.1 清理旧版本

安装Docker前先卸载清理一下旧版本。不管过去有没有安装过docker，先执行下面的步骤清理一下准没错。  

``` bash
# 1. 删除Docker Engine，CLI，container以及Docker Compose Package
$ sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
# 2. 删除所有镜像，容器和存储卷
$ sudo rm -rf /var/lib/docker
$ sudo rm -rf /var/lib/containerd
```

## 1.2 设置apt仓库

在一台新机器上直接使用`apt`安装docker，一般是找不到的。在安装之前先设置一下apt仓库。  

``` bash
# 1. 更新apt，安装一些包允许apt通过https使用库
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl gnupg

# 2. 添加Docker官方GPG key
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. 设置apt库
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 1.3 安装Docker Engine

完成上面的设置以后，就可以通过`apt`安装Docker Engine了。

``` bash
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 1.4 验证

完成安装了以后，可以验证一下Docker Engine确实成功安装了。  

``` bash
$ sudo docker run hello-world

latest: Pulling from library/hello-world
719385e32844: Pull complete
Digest: sha256:9eabfcf6034695c4f6208296be9090b0a3487e20fb6a5cb056525242621cf73d
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```

出现上面的结果那就说明安装成功了，Docker拉取了hello-world的镜像并且拉起了一个容器。然后可以把容器和镜像给删了。  

``` bash
$ sudo docker ps -a             # 找到hello-world容器的id
$ sudo docker rm {容器id}       # 删除容器
$ sudo docker rmi hello-world   # 删除镜像
```

# 2 K3s

## 2.1 安装

K3s的安装流程很简单，就一条命令。  

``` bash
$ sudo curl -sfL https://get.k3s.io | sh -s - --docker    # 这里的docker就是设置以Docker作为容器运行时
```

国内因为网络原因，更推荐下面的命令。  

``` bash
$ sudo curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - --docker
```

## 2.2 验证

等安装完成以后，小等一会儿，验证一下安装成功。  

``` bash
# 1. 确认集群可用
$ sudo k3s kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
kube-system   local-path-provisioner-6d59f47c7-lncxn   1/1     Running     0          51s
kube-system   metrics-server-7566d596c8-9tnck          1/1     Running     0          51s
kube-system   helm-install-traefik-mbkn9               0/1     Completed   1          51s
kube-system   coredns-8655855d6-rtbnb                  1/1     Running     0          51s
kube-system   svclb-traefik-jbmvl                      2/2     Running     0          43s
kube-system   traefik-758cd5fc85-2wz97                 1/1     Running     0          43s

# 2. 确认docker容器在运行
$ sudo docker ps
CONTAINER ID        IMAGE                     COMMAND                  CREATED              STATUS              PORTS               NAMES
3e4d34729602        897ce3c5fc8f              "entry"                  About a minute ago   Up About a minute                       k8s_lb-port-443_svclb-traefik-jbmvl_kube-system_d46f10c6-073f-4c7e-8d7a-8e7ac18f9cb0_0
bffdc9d7a65f        rancher/klipper-lb        "entry"                  About a minute ago   Up About a minute                       k8s_lb-port-80_svclb-traefik-jbmvl_kube-system_d46f10c6-073f-4c7e-8d7a-8e7ac18f9cb0_0
436b85c5e38d        rancher/library-traefik   "/traefik --configfi…"   About a minute ago   Up About a minute                       k8s_traefik_traefik-758cd5fc85-2wz97_kube-system_07abe831-ffd6-4206-bfa1-7c9ca4fb39e7_0
de8fded06188        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_svclb-traefik-jbmvl_kube-system_d46f10c6-073f-4c7e-8d7a-8e7ac18f9cb0_0
7c6a30aeeb2f        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_traefik-758cd5fc85-2wz97_kube-system_07abe831-ffd6-4206-bfa1-7c9ca4fb39e7_0
ae6c58cab4a7        9d12f9848b99              "local-path-provisio…"   About a minute ago   Up About a minute                       k8s_local-path-provisioner_local-path-provisioner-6d59f47c7-lncxn_kube-system_2dbd22bf-6ad9-4bea-a73d-620c90a6c1c1_0
be1450e1a11e        9dd718864ce6              "/metrics-server"        About a minute ago   Up About a minute                       k8s_metrics-server_metrics-server-7566d596c8-9tnck_kube-system_031e74b5-e9ef-47ef-a88d-fbf3f726cbc6_0
4454d14e4d3f        c4d3d16fe508              "/coredns -conf /etc…"   About a minute ago   Up About a minute                       k8s_coredns_coredns-8655855d6-rtbnb_kube-system_d05725df-4fb1-410a-8e82-2b1c8278a6a1_0
c3675b87f96c        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_coredns-8655855d6-rtbnb_kube-system_d05725df-4fb1-410a-8e82-2b1c8278a6a1_0
4b1fddbe6ca6        rancher/pause:3.1         "/pause"                 About a minute ago   Up About a minute                       k8s_POD_local-path-provisioner-6d59f47c7-lncxn_kube-system_2dbd22bf-6ad9-4bea-a73d-620c90a6c1c1_0
64d3517d4a95        rancher/pause:3.1         "/pause"

# 3. 可以确认一下运行时为docker，看最后一项CONTAINER-RUNTIME有docker字样
$ sudo k3s kubectl get nodes -o wide
```

完成上面的基础验证以后，拉起一个nginx deployment确认可用。复制下面的内容，创建一个yaml配置文件`nginx-deployment.yml`。

``` bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```

根据配置文件创建一个deployment，并且检查。

``` bash
# 创建deployment
$ sudo k3s kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created

# 查看deployment
$ sudo k3s kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           51s

# 查看pod
$ sudo k3s kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-699bfcdcb6-sr8km   1/1     Running   0          2m9s
```

然后就可以删掉了。

``` bash
$ sudo k3s kubectl delete deployment nginx-deployment   # 删除deployment
deployment.apps "nginx-deployment" deleted

$ sudo docker rmi nginx:1.7.9
Untagged: nginx:1.7.9
Untagged: nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
Deleted: sha256:84581e99d807a703c9c03bd1a31cd9621815155ac72a7365fd02311264512656
Deleted: sha256:3b35ce69ff9691cd5c266a827dd889c05666135401756e559e8bbe1ac8e7dfdb
Deleted: sha256:62811140b8ff89fb0ce776cbcc7551b2d8f227065d2af585f1bad07fb07ee307
Deleted: sha256:5e76e0bf4a65fe60ef18c3987da04bb78e297ffd5da6302d6368050821673dab
Deleted: sha256:0df46e0bdc7aef79a9bb22a779e2cc2cc03132327e995fa1ce9d05ecb30707d9
Deleted: sha256:d9e4c3db5e337b04945ced14b6e4b19774425b6a705a9eabf8c5d92d37cb809e
Deleted: sha256:04fa8e2d74d03247eea2e02d13c962341c3832ad86d0e0504eaa9c92d79cf07c
Deleted: sha256:eaf3cdc8309a378b8fc3d8843c6b06048cf78219059ac9ee63633413c37347cb
Deleted: sha256:c1cba39a97737492375aa079d6a8ab5b66e6420a8a6be8b3ada5694fe59d090b
Deleted: sha256:25f33d51678a93ea9a1c83986cfe226739b4b4423fb0dfd7c0d9e71af6a2dc6b
Deleted: sha256:d4ce9897969d842636a8fc2bcb6affc61b6b187a53743f49ef2e365e6a6b5182
Deleted: sha256:7e4ec45a1110399ad13faabf18a0b00c461563da1d8598273b8c2def8a13daf0
Deleted: sha256:86a499023e872dce3634702c42802664f1e25011b8c48abfcdedf0f87ea33751
Deleted: sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef

```

到此，以Docker为容器运行时的K3s就完成安装了。

# Kuboard

## 运行Kuboard服务

[Kuboard](https://kuboard.cn/)是类似于DashBoard的K8s多集群管理界面，但是他比DashBoard好用多了，K3s也是能用的。具体部署流程参考[文档](https://kuboard.cn/install/v3/install-built-in.html#%E9%83%A8%E7%BD%B2%E8%AE%A1%E5%88%92)。  

虽然Kuboard可以运行在K8s或K3s集群内，但是为了结构清晰，还是根据官方的建议以独立运行在集群外的docker里了，创建一个脚本文件`kuboard-install.sh`，内容如下。

``` bash
$ sudo docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 10082:80/tcp \                           # 我这里选择了映射到10082端口
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="{内网IP}:80" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  eipwork/kuboard:v3
```

然后保存运行就可以了。  

``` bash
$ sudo chmod +x ./kuboard-install.sh
$ sudo bash ./kuboard-install.sh
```

## 导入集群

等Kuboard容器运行起来了以后，就可以在浏览器通过`http://{外网ip}:10082`访问界面。  

![](Fig1.jpg)

登录账号`admin`和密码`Kuboard123`。

![](Fig2.jpg)

点击添加集群，选择`.kubeconfig`方式连接。然后将`/etc/rancher/k3s/k3s.yaml`复制到配置文本框里。**注意，内容里有一行`server: https://127.0.0.1:6443`，填到Kuboard以后确认前把`127.0.0.1`修改为你的内网地址**。最后确认就完成集群导入了。

# DockerHub配置

登录到自己在DockerHub的私有仓库。

``` bash
$ docker login -u 用户名 -p 密码
```