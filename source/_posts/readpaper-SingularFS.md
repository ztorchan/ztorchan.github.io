---
title: >-
  论文阅读 | SingularFS：A Billion-Scale Distributed File System Using a Single
  Metadata Server
tags:
  - 文件系统
  - 元数据
  - 论文阅读
categories:
  - 论文阅读
math: true
date: 2023-11-24 13:58:00
---


这篇文章出自清华大学存储研究组（舒继武团队），探讨了可承载Billion级别分布式文件系统的元数据服务构建。文章发表在了ATC`23。

<!-- more -->

*Paper Link*: [https://www.usenix.org/conference/atc23/presentation/guo](https://www.usenix.org/conference/atc23/presentation/guo)  
*Open Source*: 未开源



文章的标题起得相当直白，利用单个元数据服务器支撑Billion级别的分布式文件系统。大规模的分布式文件系统（DFS）已经是现代数据中心最重要的基础设施之一，但想要用单个元数据服务支撑整个DFS还是相当困难的，现有的DFS都包含多个元数据服务器。而RDMA和持久性内存PM带来的硬件提升给了这个机会，无论是性能还是访问粒度上的提升都给元数据服务带来了很大的上升空间。  

单单把硬件换了肯定是不够的，文章分析了现有的文件系统不足之处，得出了搭建高性能元数据服务的三个挑战：  

![图1 现有文件系统性能分析](fig1-existing_fs_analysis.jpg)  

- **崩溃一致性的保障需要大量开销**。现有文件系统一般使用写前日志（write-ahead-logging）或者日志型文件系统（log-structured-filesystem）来保证崩溃一致性。但它们各自也有缺点：写前日志方式需要两次写入数据，引入了写放大问题；日志型文件系统则是造成了繁重的垃圾回收任务。图1中，Ext4-DAX采用WAL方式保证崩溃一致性，InfiniFS和CephFS则依靠其底层存储的事务达成此目标，它们的吞吐率都比较低。NOVA采用了日志型文件系统，但在协调多节点并发更新时表现也不如人意。  

- **多节点对共享目录并发操作时的锁争用，导致极低的性能和并行度**。在同一共享目录下并发的*create*和*delete*操作需要同时修改共同的父目录dentry和inode，这就带来了严重的锁争用，使得性能大幅度降低（图1 b）。  

- **当前文件系统没有充分利用NUMA架构**。NUMA感知的设计对于元数据服务的存储和处理来说是非常重要的。一方面，过往的工作表明远程PM访问在带宽和小粒度吞吐率上有着明显的下降，因此是否充分利用NUMA的局部性对发挥PM的性能有莫大的影响。另一方面，现有的文件系统无法在保持NUMA局部性的同时扩展到多NUMA节点上，这是由于它们的粗粒度分区策略。  


# SingularFS

针对前面提到的三点挑战，文章提出了针对性设计，并依此实现了SingularFS。

## 总览

 ![图2 SingualrFS架构](fig2-architecture.jpg)  

SingularFS由客户端库和服务端组成。服务端在PM中存储文件系统元信息，客户端通过用户态库提供的类POSIX接口进行访问。服务端和客户端都配备了RDMA网卡进行通信。  

![图3 KV存储中的元数据](fig3-md_in_kvstore.jpg)  

SingularFS采用通用的KV存储作为其存储后端，要求该KV存储能够执行**单点查询**和**前缀匹配（范围查询）**，同时需要保证单目标操作的原子性。  

SingularFS将目录的Inode分为两部分：（1）**时间戳元数据（timestamp metadata）**，包括*atime*、*ctime*、*mtime*，它们的Key为&lt;*inode ID*&gt;；（2）**访问元数据（access metadata）**，包括除时间戳元数据以外的信息，它们的Key为&lt;*parent inode ID/name*&gt;。对于文件的Inode信息，它们的Key也为&lt;*parent inode ID/name*&gt;。  

SingularFS并不直接维护目录项dentry，dentry的维护更新是融合进入Inode KV对的维护更新的，借助底层KV存储的前缀匹配范围查询，可以很容易获取目录项信息。例如客户端发起一次*readdir*调用查询Inode ID为1的目录下的目录项，那么服务端只需要在KV存储中以*1/*为前缀进行范围查询，便能查找到该目录下所有的文件和子目录。  

## 无日志元数据操作（Log-free Metadata Operations）  

![图4 元数据操作分类](fig4-metadata_classification.jpg)  

为了解决崩溃一致性保障带来的大量开销，SingularFS设计了无日志同时保证崩溃一致性的元数据操作。  

SingualrFS首先将元数据操作涉及到的元数据个数，将它们分为了三类，如图4所示。*open*、*close*等操作仅涉及目标Inode的修改，它们为**单点操作（Single-node）**。*mkdir*、*rmdir*等创建和删除操作涉及到目标Inode和父目录Inode的修改，为**两点操作（Double-node）**。而*rename*操作较为特殊，它要同时修改旧Inode、新Inode和它们各自的父目录Inode，因此它单独成一类。它们保证崩溃一致性的方式各自不同。  

- **单点操作**。单点操作由于只修改一个KV对，因此其崩溃一致性可以直接由底层KV存储保证。  

- **两点操作**。两点操作的崩溃一致性保证是SingularFS的设计重点，它通过下面所述的**有序元数据更新实现**。  

- **Rename**。Rename的崩溃一致性仍然由写前日志保证，由于Rename在文件系统实际使用中占比较少，这不会带来太大的性能影响。  

### 有序元数据更新（Ordered Metadata Update）

SingularFS为文件Inode和目录访问信息添加了两个信息：***btime*（创建时间）**和***detime*（删除时间）**。由于两点操作会创建或删除Inode，并修改其父目录的*ctime*，那么可以观察到：  

*有一个目录Inode $d$，对其任意一个子Inode $c$，都应有$d.ctime \geq max(c.btime, c.dtime)$。*  

基于这个观察，SingualrFS有序更新元数据来保证两点操作的崩溃一致性。  

![图5 两点操作的崩溃一致性保证](fig5-crash_consistency_of_double_node.jpg)  

对于**Inode创建**操作，如图5（a）所示，一共分为两个步骤。先创建目标Inode并将其设置其*btime*，随后再将其父目录Inode的*ctime*和*mtime*设置为相同的值。若在这两步之间系统发生了崩溃，重启后可以发现子Inode的*btime*大于父目录Inode的*ctime*和*mtime*，那么便可以完成修复。  

对于**Inode删除**操作，如图5（b）所示，一共分为三个步骤。首先设置目标Inode的*dtime*使其失效，其次设置父目录Inode的*ctime*和*mtime*为相同的值，最后在KV存储中删除目标Inode的KV对。若系统在第一和第二步之间崩溃，依然可以通过比较父目录Inode和子Inode的这几个时间戳发现错误；若在第二和第三步之间发生崩溃，则可以通过检查子Inode的*dtime*是否已被设置以发现应当被删除的Inode，并删除相应的KV对。  

## 分层并发控制（Hierachical Concurrency Control）

分层并发控制的设计是为了解决多节点对共享目录并发操作时的低性能和并行度的问题。当多节点并发地对同一个共享目录下进行两点操作，会产生对父目录的**目录项**和**时间戳**两方面的写冲突。由于在SingularFS中，目录项是跟随子Inode的KV数据对一同变化的，目录项的写冲突问题自然而然就消失了，只剩下对时间戳的写冲突需要解决。  

文章观察到两点操作仅仅只需要修改父目录Inode的*ctime*和*mtime*，一共16B大小。因此利用16B原子CAS操作可以实现无锁更新，基于此SingularFS设计了分层并发控制。  

SingularFS首先将元数据操作分为了三类：  
- ***update***：只更新目标Inode的*ctime*和*mtime*的操作。  
- ***writer***：更新目标Inode其它信息的操作。  
- ***reader***：不更新目标Inode的只读操作。  
例如，*create*/*delete*操作既是目标节点的*writer*，又是其父目录的*updater*。  

![图6 分层并发控制算法](fig6-hierarchical_concurrency_control_algorithm.jpg)  

SingularFS为每一个Inode都添加了一把读写锁，并基于上述分类，设计了如图6的并发控制算法。  

- ***write*与其它操作同步**。*writer*操作时会直接获取读写锁的写锁，以保证操作期间的独占访问；而*updater*和*reader*则会在操作前获取读锁。  

- ***updater*与*updater*同步**。*updater*操作仅仅修改Inode中的*ctime*和*mtime*，将它们放置在连续地址上共16B大小，因此可以使用$cmpxchg16b$做原子修改。具体而言，*updater*首先获取读锁，避免*writer*同时进行修改。接着，获取当前*ctime*和*mtime*的快照。若是当前快照的时间戳不小于传入的目标时间戳，那么说明已经有更近的操作完成了更新，可以直接结束；否则通过$cmpxchg16b$尝试进行CAS原子修改，若是修改失败则重复这个过程。  

- ***updater*与*reader*同步**。可以观察到*ctime*的值是单调递增的，这具备了版本号的性质。基于此，SingularFS采用了OCC的机制达成*updater*和*reader*之间的同步。具体而言，在获取整个Inode数据的前后都获取一次*ctime*，并进行比较，只有当两个*ctime*值相等才视为成功读取。  

## 混合Inode分区（Hybrid Inode Partition）

为了实现对NUMA架构的充分利用，SingularFS设计了混合Inode分区。  

### NUMA间Inode分区

文章观察到文件操作（file operation）占据所有元数据的大部分操作，因此SingularFS主要针对文件操作设计实现NUMA局部性。  

对于单点操作，可以将每个元数据指定给特定的NUMA节点处理，以保持NUMA局部性。但是对于两点操作，例如*create*/*delete*，涉及到的两个Inode可能被分配到不同的NUMA节点，这就导致了需要跨NUMA处理。但是可以观察到，两点操作仅仅是修改了父目录Inode的*ctime*和*atime*，那么就有了解决方案。  

正如最前面所介绍的，SingularFS将目录的Inode分为时间戳元数据和访问元数据两部分，分成两个KV数据对存储。那么，将**父目录时间戳元数据、子目录访问元数据和子文件元数据**共同分在一组，分配给一个NUMA节点进行处理，那么便可以保证两点操作的NUMA局部性。SingularFS将各个元数据按照上述方式分为多个组，使用一致性哈希的方式将各个组分配给NUMA节点，实现了操作的NUMA局部性。（笔者愚见：按照这种方式，如果某个目录下文件和子目录数量极多，反而造成NUMA节点负载不均衡，导致性能下降的风险。）  

但是由于目录元数据被分成两部分存储，对某些需要同时访问修改目录时间戳元数据和目录访问元数据的操作而言，又会引入如何保证崩溃一致性的问题。对此，SingularFS通过确定的操作顺序达成目的：  

- ***mkdir*/*rmdir*操作**。这两个操作涉及到三个KV数据对：目标目录访问元数据、目标目录时间戳元数据、父目录时间戳元数据。SingularFS保证先创建/修改目标目录访问元数据（注意*btime*和*dtime*包含在这里面），这样当崩溃发生时，系统重启后可以轻松根据访问元数据的信息进行恢复。  

- **目录的*set_permission*操作**。这个操作涉及到两个KV数据对：目录的访问元数据和时间戳元数据。为了保证崩溃一致性，SingularFS扩展*btime*的含义至**创建时间和最近*set_permission*时间的最大值**。当执行*set_permission*，SingularFS首先修改访问元数据KV数据对（包括权限和*btime*），再修改时间戳元数据的*ctime*。当崩溃发生，系统重启后也可以根据*btime*和*ctime*的比较判断进行恢复。  

### NUMA内Inode分区

这部分算是对前面设计的一点小优化。 

前面提到，SingularFS并不直接维护目录项，而是通过底层KV存储的范围查找实现对目录项的查询，这就要求了底层KV存储是有序存储的。但是现有的基于B+树的KV存储往往有比较严重的锁争用问题，尤其是在范围查询的遍历过程和更新时的节点分裂过程。那为了尽可能缓解锁争用的问题，SingularFS在每个NUMA节点内创建了多个索引（多颗B+树），将每个元数据通过哈希算法分配到特定的一个索引去存储。这样，当进行单点查询，只需要在目标特定的那个索引上上锁；当进行范围查询，则需要对每个索引都进行查询，而后合并查询结果。  

（笔者愚见：这个优化能起到多大作用得打个问号。）  

另一个是对*rmdir*的优化。*rmdir*操作需要确定目标目录是否为空，对于SingularFS来说就是需要进行一次范围查询，这个开销比较大。文章通过给目录元数据添加一个表示子节点数量的*num_dents*变量来优化这个过程。这个变量的崩溃一致性也是比较好保证的，系统重启时进行一次范围查询便可以对这个值进行校正。同时，为了保证这个变量的并发安全，对它的修改都是采取了原子增减的方式。  


# 实验

老样子，直接看论文吧😋。  

# 总结

这篇文章提出了一个新的元数据服务SingularFS，以KV存储作为底层存储而实现。它以较低的开销实现了的崩溃一致性，并且通过分层控制方式降低了对共享目录的锁争用，提高了并发度。此外还针对NUMA架构做了设计，通过NUMA间和NUMA内的分组充分利用了NUMA架构，进一步提高性能。  

不过这篇文章其实并没有给我一种耳目一新的感受。SingularFS的大部分性能保证和机制实现是依赖于其底层的KV存储的实现方式，这篇文章更多是描述零散细节的修修补补。