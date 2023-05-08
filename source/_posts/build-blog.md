---
title: Hexo博客重搭建
tags:
  - 博客
categories:
  - 杂货
date: 2023-5-5 18:00:35
---

换了个服务器，又要把博客环境重新安装一遍，但是基本忘了怎么做了。为了避免以后出现同一状况，把整个流程记录一下。

<p class="note note-warning"><font face="Song">这不是从头开始搭建Hexo博客，只是把存在github的博客内容和配置在一台新的服务器上拉下来重新搭建环境</font></p>

<!-- more -->

# Node.js安装

## 安装nvm

<p class="note note-warning"><font face="Song">安装之前先把原有的node删了</font></p>

为了以后管理方便，我个人选择而且推荐先安装nvm用于node的版本管理。先去Github的[nvm仓库](https://github.com/nvm-sh/nvm)上看看最新的release版本号，然后通过下面的命令得到安装脚本并且执行：  

``` bash
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/(替换为release版本号)/install.sh | bash
# 比如说我目前就要执行的是
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
```

安装脚本会尝试自动把nvm配置添加到配置文件里，完成安装以后，重启终端或者执行命令重新加载配置  

``` bash
$ source ~/.bashrc  # 根据自己不同的sh来调整，bash的话就是~/.bashrc，一般都是这个
```

然后验证一下nvm安装是否成功  

``` bash
$ nvm -v
0.39.3  # 有版本号输出就是成功
```

## 安装npm

在此之前建议给nvm换源，不然有可能因为网络原因进行不下去，在`~/.bashrc`文件尾部添加下面镜像配置语句  

``` bash
NVM_NODEJS_ORG_MIRROR=https://npm.taobao.org/mirrors/node
```

然后执行下面命令，看到有各种`vxx.xx.x`之类的版本号输出就可以了  

``` bash
$ nvm ls-remote
```

去[Hexo文档](https://hexo.io/zh-cn/docs/index.html)查看目前hexo需要的Node.js版本支持，相应通过下面命令安装并切换使用

``` bash
$ nvm install v18.16.0
$ nvm use v18.16.0
$ nvm list  # 上面的安装成功提示其实很明显了，不过也可以通过下面两个命令来验证一下  
$ npm -v
```

# 安装环境

其实很简单，就一条命令

``` bash
$ npm install --force
```

它会根据记录过的配置，下载好对应的package。  

因为要渲染数学公式，还要额外安装[pandoc](https://github.com/jgm/pandoc)

```
$ sudo apt install pandoc
```

到这里就结束了，可以继续快乐写博客了。