---
title: RDMA-PMEM远程持久内存
tags:
  - NVM
  - PMEM
  - RDMA
categories:
  - RDMA
  - 持久性内存
date: 2023-10-03 15:44:00
---

谈谈利用RDMA对远程持久内存的构建方式。  

<!-- more -->

持久性内存高吞吐低延迟的性能以及可按字节寻址的特性，配合上RDMA高速网络，总是让人充满遐想，新的大型分布式存储系统仿佛近在咫尺。