---
title: 论文阅读 | Fisc：A Large-scale Cloud-native-oriented File System
tags:
  - 分布式系统
  - 文件系统
  - 论文阅读
categories:
  - 论文阅读
date: 2023-03-15 13:36:00
---


这篇文章发表于刚刚结束的FAST'23，是阿里云在本次会议中入选的4篇文章之一。文章介绍了他们面向云原生的大规模文件系统Fisc的实现，主要描述了客户端功能的硬件卸载和存储感知的分布式网关带来的性能提高，是一篇工程性较强的文章。

<!-- more -->

*Paper Link*：[https://www.usenix.org/conference/fast23/presentation/li-qiang-fisc](https://www.usenix.org/conference/fast23/presentation/li-qiang-fisc)  

# 1 背景&动机

当下云原生技术已经发展到了一个相对成熟的阶段，普遍的虚拟化方式已经从虚拟机变成了容器，提供给租户的抽象也从资源（CPU和内存）变成了服务（例如数据库和对象存储服务），大量数据分析、机器学习还有事务工作流等等之类的应用已经被部署在了公有云上。但是现有的分布式文件系统（DFS）并不适用于多租户云原生应用，主要的原因有两个：

![Fig1. HDFS客户端在不同带宽下的CPU消耗](Fig1-hdfs_client_consumption.jpg)  

  - 其一，目前**传统DFS的客户端大多是重量级的**，诸如存储协议、网络相关功能和安全相关功能都被放置在了客户端中，导致每个容器内的客户端都要保留许多独占资源，单台服务器只能同时承载少量容器。各个容器内的客户端对于资源的多路复用程度较低，导致资源使用效率低下。Fig.1展示了HDFS客户端在不同带宽下的CPU使用情况，可以看到在200MB/s带宽下的Write就大约消耗了1.1个CPU，考虑到一般模式下每个容器分配2个CPU，有超过50%的CPU资源用于I/O中。
  
  - 其二，**集中式网关无法满足云原生应用对性能、可用性和负载均衡的需求**。主要限制有以下几点：（1）通过网关会有次优的毫秒级延迟；（2）网关无法感知文件语义和存储协议，从而无法进行数据的局部优化和快速故障处理；（3）在不对客户端进行侵入式更改的情况下，无法兼容RDMA等高性能网络堆栈；（4）由于基于网络连接数和基于文件数的负载均衡存在误差；（5）让集中式网关的吞吐量与数千节点的大型文件系统集群相匹配，这会导致高昂的成本。

# 2 Fisc总览
## 2.1 设计原理

Fisc的设计原理可以从三方面概况：

  1. **聚合FS客户端的资源**。从本质上讲，资源聚合本身就是云原生的本质。不同于传统的资源密集型FS客户端，Fisc将存储协议和网络相关的功能卸载到物理域（与之对应的是客户端所处的虚拟域，也即容器）上，例如计算端和存储段的DPU，实现资源聚合和客户端的轻量化。
  
  2. **分布式存储感知网关**。Fisc使用了具备存储感知能力的分布式网关，以此建立了计算端和其对应存储节点的直连“高速路（highway）”。这使得虚拟域和物理域可以通过高性能网络协议进行连接，也可以充分利用存储语义以提高可用性和文件访问请求的局部性，同时保证了各个存储节点的负载均衡。
  
  3. **软硬件协同设计**。Fisc还将新兴的DPU部署在物理服务器上，通过精心的软硬件协同设计，实现从虚拟域用户容器到物理域FS的安全高效的透传。更进一步，Fisc还在DPU中引入高速通道（fast path）来加速I/O处理。

## 2.2 架构

![Fig2. Fisc架构](Fig2-Fisc_architecture.jpg)  

  Fig.2展示了Fisc的整体架构。Fisc由**控制层面**（control plane）和**数据层面**（data plane）组成。  
  
  控制层面提供了开放的APIs给租户创建Fisc FS实例、挂载到虚拟机/容器，以及分配virtio设备加速虚拟域到物理域的透传。数据层面则由接口层、分布式存储感知网关（SaDGW）和持久化层构成。轻量级客户端放置在前端给应用提供FS服务接口；SaDGW在中间层，包含计算端DPU中的Fisc agent和存储节点中的Fisc proxy，以及存储集群中的一组Fisc proxy master。Fisc proxy master负责管理Fisc agent和Fisc proxy；Pangu则在后端作为持久化层，负载处理请求和持久化数据。（阿里云在FAST'23还发表了一篇详细介绍Pangu2.0的文章：[***More Than Capacity: Performance-Oriented Evolution of Pangu in Alibaba***](https://www.usenix.org/conference/fast23/presentation/li-qiang-deployed)）

## 2.3 Fisc工作流
  
  控制层面上，当租户调用API创建一个Fisc实例，Fisc控制层面会将实例映射到后端PanguFS，并把租户信息和挂载点推送给Fisc proxy masters。当租户将挂载点附给虚拟机/容器时，Fisc proxy masters会将挂载点和Fisc proxy的映射推送给其所处物理机的Fisc agent。最后，控制层会将一个virtio-Fisc设备分配给虚拟机/容器。  

  数据层面上，给定一个元数据操作请求，它通过virtio-Fisc设备到达Fisc agent。Fisc agent根据挂载点和Fisc proxy的映射，随机选择一个Fisc proxy。如果是一个文件的打开操作，则将使用其file handle和Fisc proxy位置来构造与打开文件相关联的路由条目。之后，后续对该文件的读写请求将根据路由条目进行路由。

# 3 设计与实现
  
  本节将详细描述Fisc各部分机制的设计与实现。  

## 3.1 轻量级Fisc客户端

### 功能卸载与资源聚合

  典型的重量级FS客户端提供有四类功能：（1）文件接口和相关数据结构；（2）存储协议，例如副本可靠性、数据一致性和故障处理；（3）安全和授权；（4）网络协议。文章发现，在云原生场景下，用户往往只关心第（1）类功能，其余的功能对用户来说是透明的。因此，将其余三类功能从客户端内移出并聚合，卸载到硬件上，可以实现很大程度的资源聚合。  

  - **将网络相关功能卸载到Fisc agent**  
    考虑到在近期，基于DPU的高性能网络栈（Luna/Solar和Nitro SRD）取得了极大成功，Fisc选择将网络相关的功能卸载到了物理域上的Fisc agent。具体而言，Fisc agent扩展了Luna/Solar网络栈，将同一台物理机上的客户端的网络连接聚合。这大大节省了每个客户端预留给网络相关操作的CPU和内存资源。  

  - **将安全相关功能卸载到Fisc agent**  
    Fisc采用了早期检查（early-checking）的设计，当Fisc agent接收到来自Fisc client的检查时，就进行安全检查。不同于在网关上检查的方法，这种设计防止了恶意流量消耗后端存储集群的资源。  

  - **将存储协议功能卸载到Fisc proxy**  
    Fisc将存储协议功能卸载到了位于存储集群的Fisc proxy上，而非Fisc client，主要有三点原因。第一，计算端DPU所持有的资源有限，在将资源花费在网络功能、安全功能和virtio-Fisc设备的裸机虚拟化以后，DPU没有足够的资源继续支持复杂的存储协议了。第二，将存储协议功能卸载到存储集群，有助于将计算端和存储集群之间的存储流量移动到存储集群内的后端网络，从而节省了计算-存储分离体系结构中的稀缺网络资源。第三，这样可以在存储集群中采用面向存储的优化和硬件辅助加速，以提高系统整体性能和降低成本。  

### 简洁性和兼容性

  为了简化Fisc client的开发，Fisc使用了RPC的方式实现了Fisc的APIs。在此之上，Fisc引入类似Protocol Buffers的机制来保持Fisc client在不同版本之间的兼容性。但是直接使用PB（反）序列化会引入额外的开销，浪费DPU的有限资源。因此最终，Fisc将Fisc APIs分类成了数据相关（e.g., read, write）和元数据相关（e.g., create, delete, open, close）两类。数据相关的APIs使用更为频繁，因此为它们精心设计了多个数据结构保持兼容性。而对于元数据相关的APIs，则直接使用PB进行（反）序列化。由此，Fisc在性能和兼容性上达成平衡。

## 3.2 分布式存储感知网关（SaDGW）
  
  Fisc使用了SaDGW用于建立Fisc agent和Fisc proxy之间的直接连接。因此，Fisc得以在这些连接中使用高性能网络栈，并更进一步地利用存储语义建立文件粒度地存储感知路由。

### Agents和Proxies之间的直接高速通路

![Fig3. Fisc的路由过程](Fig3-routing_process_of_Fisc.jpg)  
  
  在DPU的支持下，Fisc得以建立Fisc agents和Fisc proxies之间的直接高速通路，而不再需要网络网关。在高速通路上，Fisc运用了高性能网络栈Luna/Solar，而非TCP/IP网络栈。此外，Fisc还使用了原生数据结构（Raw data structures）来消除Fisc agents和Fisc proxies之间（反）序列化的开销。

  更重要的是，SaDGW引入了**文件粒度的路由表**，通过一种中心控制的机制来管理高速通路。如Fig.3所示，Fisc agent维护了一张文件粒度的路由表（Route table），上面记录了文件handle信息以及对应服务该文件的Fisc proxy地址。当文件初次被打开时，Fisc agent接收到来自Fisc client的文件打开请求，随后随机选择一个Fisc proxy发送该请求。当文件成功被打开后，一条包含文件handle、所选Fisc proxy位置和SLA相关信息的路由条目（entry）将跟随请求响应返回。随后，当I/O请求到达Fisc agent时，它会根据路由表查找相应的Fisc proxy地址并将请求传输过去。由于DPU资源有限，路由表采用LRU实现以控制路由表大小。

### 存储感知的故障处理
  
  对于存储协议的故障处理主要有三方面要考虑：  
  
  - **重试超时（retry timeout）**：Fisc agent重试失败请求的最大次数，和用户设置的请求超时和高速通路质量有关  
  - **重试目的地（retry destination）**：路由表记录指定的Fisc proxy，一旦重试超时发生就会被新的proxy替换。  
  - **高速通路质量（highway quality）**：Fisc proxy响应的平均延迟。  
  
  因此，Fisc扩展了agent上的路由表，增加了三个表项：**重试次数（retry times）、重试超时（retry timeout）、平均延迟（avg-latency）**。以此实现存储感知的故障处理。  

  Fisc利用了多种机制来处理故障：  

  - **重试（retry）**：当Fisc agent检测到一次失败的请求，它会重试多次直到获得成功的响应或是超过了用户设置的重试超时。通常用户会设置一个相对较大的请求超时，因此Fisc agent会按照经验初始化设置一个较小的请求超时来检测失败请求。当失败请求发生，agent会加倍请求超时并重试。这个机制可以处理暂时性的失败，例如网络抖动等等。  
  - **黑名单（Blacklist）**：在检测到连续的请求失败或者请求某一个Fisc proxy有较大的平均延迟时，Fisc agent会将该proxy列入黑名单。一个后台线程会周期性地ping这些proxy，并将成功ping通的proxy移出黑名单。元数据操作请求在选择Fisc proxy时会排除黑名单中的proxy，数据操作请求则有下述重新打开机制。  
  - **重新打开（Reopen）**：如果数据操作请求的目标proxy在黑名单内，Fisc agent会选择一个新的Fisc proxy来重新打开文件，并更新路由表记录。若不在黑名单内，对于失败的请求，Fisc agent会在保证仍有足够剩余时间的情况下重试请求。在余下的时间里，它会选择一个新的proxy重新打开文件，保证在规定时间内完成请求。  

### 位置感知读取
  
  当发起一个读取操作时，请求首先会被发送到Fisc proxy上，然后由它再发送到目标数据所在的Pangu块服务器（chunkserver）上。读响应则会携带数据，沿着相反路径发送回。这会造成读流量的放大，消耗额外带宽，将整个集群吞吐量减半。考虑到每个存储节点上都部署了一个Fisc proxy线程和一个Pangu chunkserver线程，Fisc设计了位置感知读取避免流量放大。

![Fig4. 位置感知读取](Fig4-design_of_locality-aware_read.jpg)  
  
  Fisc在Fisc agent中维护了范围表（range table）。每当Fisc proxy响应Fisc agent的打开或者读取请求时，同时会返回最近可能会被读取的chunk信息（由Pangu的预测机制决定），每次预测的chunk被经验地设置为16。这些信息会被编码组织成范围-地址对（range-location pair），并插入范围表内。范围表的每一条记录对应一个文件，考虑到DPU有限的资源，每条记录最多维护64对范围-地址。对于64MB大小的chunk的情况下，范围表内每个文件的记录最多可以直接定位4GB的数据，这已经满足了绝大多数的文件。此外，每个文件对应范围表记录的索引作为一个只读hint存储在路由表记录内（后续详解）。  

  有了范围表的支持，当一个请求到达Fisc agent，它会查找文件的路由表记录并得到只读hint，进而查找相应范围表记录，得到匹配的范围-地址对。若成功命中，请求会被直接送往目标的存储结点。在某些情况下，读取请求的范围大于命中对的范围，考虑到DPU内有限的CPU资源，Fisc并不将请求划分，避免了分割、组合和针对性的故障处理等复杂操作。当Fisc proxy接受到读取请求，它将调用Pangu客户端完成操作。在此情况下，目标proxy和目标Pangu客户端处于同一个物理节点上，它们便可以使用共享内存完成通信，而非通过网络。这样，读取请求和响应只经过了一个网络，提高了这个存储集群的读吞吐量。  

## 3.3 DPU支持下的软硬件协同设计  

### 基于DPU的Virtio-Fisc设备  

  Virtio-Fisc设备是一个遵从virtio标准的PCIe设备，主要由两部分构成，前端位于虚拟机/容器，后端位于DPU内。Fisc client通过前端将请求放入virtio硬件队列（virtio hardware queue），随后运行在DPU处理器上的Fisc agent将请求从硬件队列取出并处理。Fisc agent将请求发送Fisc proxy，再将接收到的响应放入硬件队列中，由前端Fisc client所处理。Fisc使用了两代Virtio-Fisc设备：  

  1. **基于virtio-block的Virtio-Fisc设备**。该类型的virtio-Fisc设备兼容大多数主要的OS，不必修改就可以被多数虚拟机/容器所使用。前端与标准virtio-block设备相同，使用virtio-block接口，并且实现了一个轻量的通信库，为Fisc client提供了block的读写操作。然而后端有很大不同，硬件队列中的请求是由Fisc agent处理，而非传统的virtio-block软件。  
  2. **定制化设计的Virtio-Fisc设备**。Fisc设计引入了这种设备以消除virtio-block的限制。例如，virtio-block的队列在大多数OS中深度限制在了128，这对Fisc的非阻塞请求是不够的。这种virtio设备更像是一个NIC设备。Fisc更进一步地在RPC层级利用该设备，使得它更适用于如FaaS的云原生服务。它利用virtio队列为RPC请求传递命令并接收响应。  

### 快速路径

![Fig5. 快速路径](Fig5-design_of_fast_path.jpg)  

  Fisc在计算端DPU内的FPGA中维护了路由表的缓存，从而得到了处理文件请求的快速路径。如Fig.5所示，文件handle和网络连接之间的映射被缓存进了FPGA中。当请求携带着文件handle到达定制化virtio-Fisc设备的FPGA时，FPGA解析文件handle并查找缓存的路由表。若命中，该请求会被直接打包成网络包发往目标网络连接；否则，请求会被送入位于CPU内的Fisc agent进行处理。缓存的记录由Fisc agent控制和更新，以减轻FPGA缓存实现的复杂性。为了控制网络传输带宽，每个连接的传输窗口也由Fisc agent在缓存的记录中设置和更新。

### 资源优化
  
  为了尽可能节省DPU上的有限资源，Fisc分别对其CPU、内存和网络做出了优化。

  - **CPU优化**  
    针对CPU的优化主要有两个方面：**批操作（Batch operation）**和**手动PB（反）序列化**。批操作指的是Fisc会将多个请求集中到一个请求当中，以共享Fisc client和Fisc agent之间的virtio协议处理。手动PB（反）序列化是指，Fisc针对特定数据类型使用了定制化的PB（反）序列换方法，这比PB编译器生成的方法更加高效。  

  - **内存优化**  
    Fisc agent主要的内存消耗在于路由表和范围表，为了节省内存，Fisc压缩了表记录所需要的内存。对于存储节点小于一百万的集群，Fisc使用20bit来表示IP地址，而非32bit。  

    更进一步，Fisc把范围表传给并存储在Fisc client，而无需在DPU中存储大量的地址。这样，当Fisc client发起请求时，它可以感知到目标chunk的范围并找到相应地址，将其作为hint伴随请求一起发送。接着Fisc agent首先根据路由表检查文件handle和租户信息，对于通过检查的请求，再根据伴随的hint发送请求。考虑到安全性，范围表在传递给Fisc client时使用了一个索引进行编码，用户无法解析其含义。为了避免应用恶意修改hint，Fisc agent和Fisc proxy都会检查索引，如果检查未通过，Fisc agent会在一段时间内禁用该用户的局部读机制。

  - **网络优化**  
    Fisc仔细处理直接高速通路的连接数量。第一，Fisc使用了共享连接机制（shared-connection mechanism）来减少Fisc agent和Fisc proxy之间的连接数量。第二，Fisc定期解除空闲连接以回收网络资源。第三，相较于同地域的带宽，跨地域的Tbps级带宽显得较窄，因此Fisc agent只连接不同地域的部分Fisc proxy。

## 3.4 大规模部署

### vRPC

![Fig6. vRPC](Fig6-abstracted_vRPC.jpg)  

  如Fig.6所示，Fisc抽象了一个vRPC服务。vRPC和传统的RPC很类似，由Fisc client通过vRPC stub调用，并被Fisc proxy的vRPC server处理。它的具体实现细节对云原生应用开发者来说是透明的，但事实上与传统RPC有区别。首先，在高效的virtio设备的支持下，它提供了从虚拟域到物理域的安全透传。其次，它可以在Fisc agent内被重试，并使用高性能网络栈，这对Fisc client来说仍然是透明地。第三，它为服务提供了一个使用适配器的机会，该适配器可以被集成到Fisc agent中来提高服务可用性。  

### 负载均衡
  
  Fisc引入了两种机制来实现存储集群内上千Fisc proxy的负载均衡。  
  
  - **文件粒度的调度**。传统的负载均衡依赖于基于网络连接的网关调度，侧重于对各proxy的网络连接数量的平衡。然而网络连接的数量与文件的数量之间存在差距，连接数平衡并不意味着每个proxy的文件数量是平衡的。因此，Fisc提出了文件粒度的调度实现负载均衡。Fisc agent根据文件名和其它信息的哈希值将每个文件随机转发到Fisc proxy，实现文件在Fisc proxy之间均匀分布。由于Pangu将文件chunks均匀地分布到数据服务器上，通过位置感知读取，使得Fisc proxy的读请求也均匀平衡。  
  - **中心控制的重调度**。Fisc proxy master会定期收集每个proxy的负载情况，并调度转移部分文件，将它们从高负载的Fisc proxy转移到低负载之处。同时Fisc proxy master还会将负载信息推送给Fisc agent，Fisc agent会相应提高低负载proxy的哈希权重，降低高负载proxy的哈希权重。  

### 端到端的QoS
    
  Fisc支持在线实时应用和离线批处理应用的混合文件访问，分别被赋予高优先级和低优先级。  

  Virtio-Fisc设备、NIC以及网络都采用了基于硬件的QoS机制。Virtio-Fisc设备和NIC充分利用其高/低优先级队列。Fisc通过网络库在IP数据包头设置DSCP值以利用网络交换机的优先级队列。  

  Fisc client、Fisc agent和Fisc proxy则采用了基于软件的QoS机制，使用了混合线程模型，分别对高/低优先级请求使用独占线程。使用混合线程模型是为了避免队头阻塞（head-of-line blocking）问题，Fisc不对大请求进行划分，因此如果高/低优先级请求放入同一队列，可能会出现HOL阻塞问题。另一原因是，缺乏用于英特尔CPU的NIC缓存隔离能力。如果DPDK的轮询线程停止轮询网络包，NIC的缓冲区可能会被低优先级请求的网络包填满。因此，如果要在同一线程处理高/低优先级的网络包，NIC的低优先级队列需要保持轮询，否则会影响高优先级流量。然而，线程中对低优先级请求的不断轮询会导致难以保证高优先级请求的满足。  

# 4 结论

  文章提出了一种面向云原生的大规模文件系统Fisc，采用两层聚合机制实现容器之间对文件客户端资源的多路复用，并采用存储感知的分布式网关提高I/O请求的性能、可用性和负载均衡。Fisc还采用了带有DPU的Virtio-Fisc设备，以实现用户的虚拟域到物理域的高性能安全透传。