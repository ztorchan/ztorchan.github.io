---
title: >-
  论文阅读 | Design Guidelines for Correct, Efficient, and Scalable Synchronization
  using One-Sided RDMA
tags:
  - RDMA
  - 论文阅读
categories:
  - 论文阅读
math: true
date: 2024-01-30 22:16:00
---


文章由达姆施塔特工业大学的TOBIAS ZIEGLER等人发表于SIGMOD'23，主要描述了如何利用one-sided RDMA来构建正确、高效且具有扩展性的同步机制。

<!-- more -->

*Paper Link*: [https://dl.acm.org/doi/10.1145/3589276](https://dl.acm.org/doi/10.1145/3589276)  
*Open Source*: [https://github.com/DataManagementLab/RDMA_synchronization](https://github.com/DataManagementLab/RDMA_synchronization)  

尽管RDMA已经出现了二十多年，时至今日它仍然是构建高性能分布式系统强有力的工具。尤其是最近几年，学术界对它热情不减反增。这篇文章没有提出什么新颖的系统或者架构，它总结了一套利用one-sided RDMA设计高效正确同步机制的方法和准则。  

作为一个没啥学术天赋的人，比起天马行空的新系统设计，我向来是更喜欢这类“经验总结”类型的文章，它能给我在系统工程实现上提供及时的指导。ATC'16也有一篇[Design Guidelines for High Performance RDMA Systems](https://www.usenix.org/conference/atc16/technical-sessions/presentation/kalia)探讨了如何设计一个高性能的RDMA系统。存算分离、分离式内存等概念火热的今天，更高效的RDMA one-sided操作愈加受到青睐，因此这篇文章也更详细地针对one-sided RDMA进行讨论。  

文章首先指出利用one-sided RDMA来构建正确、高效且具有扩展性的同步机制是非常困难的。现有的文章或多或少都在这方面设计犯了错误。例如有些文章的同步机制的正确性建立在RDMA操作完全遵守地址升序的顺序进行这一假设上。但是这个假设是错误的，RDMA标准中RDMA read操作并没有这一保证，这就可能发生无法检测的错误。文章里还做了相关实验证明了这一观点。  

基于这个动机，文章开始详细介绍在RDMA one-sided场景下各类同步机制的设计。  


## 悲观同步机制

![图1 RDMA实现悲观锁机制](fig1-exclusive_latch.jpg)  

为了避免对远程数据结构的并发修改，利用one-sided RDMA实现锁存器（latch）进而实现单边悲观控制是一个有效的手段。  

现有的悲观锁模式大致可以分为**独占锁**和**读写锁**两种。以读写锁为例，典型的实现是在远程内存设置一个8字节的值：通过RDMA compare-and-swap（RDMA CAS）操作设置其中一个bit（通常是尾部bit）以获取写锁，而其它bit作为读锁计数器，通过RDMA fetch-and-add（RDMA FAA）操作进行添加来获取读锁。 图1就是一个典型的独占锁机制。 

### RDMA atomic操作性能

显而易见，悲观锁的实现高度依赖于RDMA atomics操作，在考虑如何优化悲观锁之前，需要先了解一下它们在各种情形下的性能。  

#### 争用/非争用下RDMA atomic性能

![图2 争用/非争用下RDMA atomic性能对比](fig2-(un)contended_atomic_performance.jpg)  

在某些高并发的工作负载下，不可避免地会出现对同一资源的高度争用，进而引发RDMA atomic对同一把锁的争用修改。那么争用对性能影响几何？图2展示了争用和非争用下RDMA CAS的吞吐率，作为参考也对RDMA READ进行了实验。可以看得出来争用对RDMA atomic操作的影响是非常大的，峰值吞吐率相差了25倍之多，可扩展性也差异巨大。（另外可以看到非争用RDMA CAS和RDMA READ在Worker超出256后也有所下降，这是由于QP数过多，需要维护的状态超出网卡内存导致的）  

#### 步长与RDMA atomic性能

![图3 各个锁间距在不同步长下的RDMA CAS性能](fig3-atomic_under_different_stride.jpg)  

![图4 RNIC内部锁表](fig4-rnic_architecture.jpg)  

除了并发争用以外，一个容易忽视的因素也会影响RDMA atomic的性能——**步长**。先来看看实验结果，可以从图3看到随着锁的步长逐步上升，RDMA CAS的吞吐量在下降。神奇！这是为什么呢？虽然RNIC厂商没有公开其架构，但可以从实验中窥探一二。可能RNIC对RDMA atomic操作的实现是通过内部的一个物理锁表实现的。就像图4(a)展示的那样，锁表结构类似于一个Hash表，通过目标地址的后12 bit来定位表项。对于地址相隔4KB的RDMA atomic操作，即便目标地址实际不同，但由于其后12 bit相同，它们都被分配到同一个锁表表项争用同一把锁，在此产生争用冲突。由此才会出现随着步长增大，吞吐率下降的现象。文章在ConnectX-3、ConnectX-5、ConnectX-6 RNIC都发现了这一现象。  

基于这个推测，要提高非争用下RDMA atomic操作的性能，就得尽可能避免在RNIC内的锁表冲突，也就是避免地址后12 bit相同。两种思路：

- 类似于图4(b.&#8545;)，在每一个锁前都添加一个8 bytes的填充。图4第二个实验验证了这种方法的有效性。  
- 类似于图4(b.&#8546;)，将锁和数据分离，连续的锁的地址必然可以有效避免锁表冲突。  

### 优化悲观锁

有了前面一系列的发现与分析，我们可以开始优化悲观锁了。  

![图5 独占锁优化](fig5-exclusive_latch_optimizations.jpg)  

图1所展示悲观锁机制中，所有的操作都是同步的。每个操作被调用后，都需要轮询CQ来等待操作完成。这当然是正确的，但同时也是低效而高延迟的。图5展示了对其的逐步优化：  

- **Speculative Read（预测读）**。预测读优化将不再等待CAS操作上锁返回，而是在通过CAS尝试上锁后离开进行Read操作，这样可以将Read的延迟隐藏。如果上锁成功，则进行后续的处理；如果上锁失败，则重复这个行为即可。在InfiniBand标准中，RDMA atomic保证在相同QP的后续RDMA操作前完成，因此这个优化不会带来错误。  

- **Write Combing（写合并）**。类似的，写合并优化则是将Write操作与后续的CAS释放锁合并，以此来隐藏等待写操作完成造成的延迟。  

- **Asynchronous Unlatch（异步释放锁）**。上一个优化中仍在有在等待释放锁的CAS，异步释放锁则在发出CAS后直接结束过程。但这也可能引发另一个问题：用于存放Write操作数据的缓冲区不能立刻被重用。因为在下一个操作进行时，上一操作的Write仍可能还没被完成，立刻重用缓冲区可能会导致数据被覆盖。一个典型的解决方法是，为每个QP配备多个写缓冲区，本次操作使用的写缓冲区等到下一个操作Read返回时才可重用。  

- **Write Unlatch（写释放锁）**。由于单次RDMA Write是按地址顺序进行的，我们也可以将独占锁放置在数据的末尾，那么就可以直接通过Write释放锁了。但由于InfiniBand标准并不保证从不同的QP发出的RDMA操作相互之间的可见性和顺序保证，因此这个优化仅适用于通过RDMA CAS操作锁的情况，而不适用于RDMA FAA来操作锁的情况。具体的解释可以参考论文，这里不多赘述。  

### 测试

针对上述的优化方式，文章进行了消融实验。实验中以*CAS-Read-Write-CAS*为一次写流程，*FAA-Read-FAA*为一次读流程。  

![图6 单线程锁优化消融实验](fig6-latch_ablation_study_st.jpg)  

图6展示了单线程下各种优化对读写流程的影响。可以看得出来每个优化都是有效的。对写流程来说，异步释放锁带来了最大的性能提升，但写释放锁优化的作用微乎其微。对读流程来说，由于其操作数本身较少，因此初始性能较写流程更高，但随着优化增加，这个优势也逐渐消失。  

![图7 多线程锁优化消融实验](fig7-latch_ablation_study_mt.jpg)  

图7则展示了多线程下各种优化对写流程的影响（其中*unsync*表示只有一个Read和一个Write，作为参考性能上界）。可以观察到，对于8 workers和32 workers的情况，当所有优化都用上后，其性能与*unsync*版本相当，这表示在这范围内优化带来了良好的扩展性。但是对于128 workers的情况，在使用上异步释放锁的优化后，性能反而下降了一大截。这可能是由于高并发的场景下，下一个写流程来的相当快，上一流程的异步释放锁还没结束，就开始了下一流程的获取锁。这意味着一个worker可能会获取两个锁。这就导致了频繁的冲突。  

## 乐观同步机制

悲观同步机制在争用不强烈的情况下可以实现较高的扩展性。但在有些数据结构下，争用是不可避免的。比如B-树，它的根节点必然会被频繁地访问。即便大部分访问上的是读锁，但RDMA FAA的性能也会在这激烈的争用下大幅降低。在这种情况下，乐观同步机制是相当有必要的。  

![图8 乐观同步机制的实现](fig8-intuition_for_optimistic_synchronization.jpg)  

### 一个直觉上的乐观同步实现

要实现一个乐观同步机制，通常我们的第一反应是：**读者乐观地直接读而后作验证，写者则需获取一个悲观锁避免写写冲突**。读者必须在完成读之后进行检查，确保在读的过程中数据没有被修改。为了实现这种机制，其数据布局一般如图8(a.&#8545;)所示，由一个独占锁、一个版本号和数据部分组成。读流程则如图8(b)所示，读者先通过一个RDMA Read读取整个数据，并检查独占锁是否已被获取。如果没有写者上独占锁，则发起第二次RDMA Read再一次获取版本号，并同第一次读到的版本号比对。若版本号一致，则成功读取。  

### 顺序性

前面谈到的方法看起来非常合理。但是，事实上这是**错误的实现方式**！  

RDMA消息传递的数据传输顺序受到三方面的因素影响：（1）消息顺序；（2）单个消息内的数据包顺序；（3）DMA顺序。其中前两个顺序由InfiniBand和RoCE协议保证了其顺序性，但是最后的DMA顺序性并没有得到保证。也就是说，RDMA Read并不保证其地址顺序性（InfiniBand标准同样没给出承诺），RDMA操作在设计时并没有考虑对同一块内存的并发访问。此时，中间协议就显得尤为重要，例如PCIe。为了进一步理解RDMA Read的执行行为，是时候要理解**PCIe协议**和**缓存一致性协议**了。  

![图9 RDMA的硬件栈](fig9-hw_involved_in_rdma.jpg)  

如图9所示，RDMA Read请求到达远端节点后，RNIC首先将请求发送给PCIe Controller，PCIe Controller从主机内存中获取请求的内存数据。接着数据就会通过PCIe传输给RNIC，并传回给请求者。RDMA请求会在过程中被转换为PCIe事务，由PCIe Root Complex处理。PCIe Root Complex对读请求的处理在cacheline的粒度上是一致的。一旦请求被发起，PCIe协议通过*completions*将数据传输到endpoint。  

当读的数据超出给定值（64 Bytes），这个请求就会通过多个*completion*来完成。对这种情况，PCIe标准是这样描述的：  

{% note info %}  
Memory Read Request serviced with multiple completions, the completions will be returned in address order.
{% endnote %}  

这句描述只保证了*completion*的顺序，但并没有保证数据冲内存取回的顺序。并且事实上，PCI Express® Base Specification Revision 4.0里有作如下描述：  

{% note info %}  
single Memory Read that requests multiple cachelines from host memory is permitted to fetch multiple cachelines concurrently
{% endnote %}  

在这种读顺序性无法保证（幸运的是，PCIe标准保证了RDMA Write的地址顺序性）的情况下，前面所提出来的乐观同步实现就不能保证正确了。比如说，一个读者和写者同时访问了同一块数据：（1）读者先读了第二个cacheline；（2）写者修改数据；（3）写者释放锁并增加版本号；（4）读者读取第一个cacheline；（5）读者发起第二个RDMA Read再次获取版本号。在这种情况下，读者会认为数据没有被修改。由此便发生了错误。  

### 正确的乐观同步机制

有了上述分析，为了保证乐观同步机制的正确执行，我们必须依靠额外的机制。  

![图10 乐观同步机制的正确实现](fig10-correct_optimistic.jpg)  

- **版本号（使用两次RDMA Read）**。这个实现仍然沿用了图8(a.&#8545;)的数据结构，但改变了读流程。如图10(a)，对版本号和数据的读取被拆分成了两次RDMA Read操作。第一次RDMA Read只读取独占锁和版本号，第二次RDMA Read才真正读取数据部分。顺序同步的两次RDMA Read成功避免了先前提到的PCIe乱序读取的问题，若第三次RDMA Read读取到的版本号同第一次读回的相同，则能保证在两次RDMA Read之间数据没有发生改变。但这种方法需要两次RDMA Read，性能低下。  

- **CRC校验和**。这种实现依靠一个校验和来发现数据的不一致，其数据结构如图8(a.&#8546;)所示。读流程中，通过一个RDMA Read读回整个数据并进行校验，如果有写者并发修改目标数据，那么校验就会失败。但是这种方式也有缺陷：（1）CRC校验和的计算开销比较大；（2）存在小概率的巧合，数据被修改但CRC校验仍然无误。  

- **FaRM cacheline版本**。利用[FaRM](https://www.usenix.org/conference/nsdi14/technical-sessions/dragojevi%C4%87)，我们可以进一步提升性能，对应的数据结构如图8(a.&#8547;)所示。FaRM依靠缓存一致的DMA来检测单个RDMA是否一致，它在每个cacheline的开头都会存一个版本号。通过比较每个cacheline的版本号，我们就可以检查在读取过程中是否有并发修改进行。对于写者而言，它首先给数据项上锁，随后读取、修改、增加版本号，最后通过RDMA Write按地址顺序写回数据。  

### 测试

![图11 单线程下乐观读性能](fig11-optimistic_read_st.jpg)  

首先来看单线程下各种乐观同步机制的读性能。图11展示了实验结果，其中“broken”代表的是只进行一次RDMA Read获取数据而不作任何校验的方式。  

出乎意料的，尽管悲观机制需要进行三次消息传递（FAA-Read-FAA），而乐观机制仅需要两次（Read-Read），悲观同步机制比所有乐观读机制都要表现得更好。这可能是由于悲观机制可以充分利用异步释放锁的优化，而乐观机制则仍需要进行同步地校验数据一致。  

然后是“broken”与正确的乐观读性能对比，从图11(b)可以更直观地观察它们之间的差距。FaRM方法下的乐观读性能是最接近“broken”的。但同时可以看出，CRC和FaRM方法都无法随着数据项的增大而保持其性能，因为随着数据项增大，CRC需要计算的字节数随着增大，FaRM需要校验的版本号数目也在增加，这带来额外的开销。而对于第一种版本号的方法，尽管需要进行三次RDMA Read，但其开销不会随着数据项增大而增大，在数据项较大的情况下它展现出了一定的优势。  

![图12 Read-only和Write-only下乐观机制的可扩展性](fig12-read_only_write_only_optimistic_scalability.jpg)  

再来看看各种乐观机制的可扩展性。图12展示了在Read-only和Write-only下各种机制的峰值性能（数据项大小为256 Bytes）。  

对于Read-only的情况，在Worker数量较少的情况下，悲观机制远远好于所有乐观机制，但随着Worker数目逐渐提升，乐观机制的峰值性能反超并领先了一大截。这其中的原因主要是随着Worker数提升，悲观机制的RDMA atomic操作冲突变得更频繁，造成了性能瓶颈。对于乐观机制，可以看到FaRM方法和CRC方法都十分接近“broken”，而版本号方法落后较为明显。  

对于Write-only的情况，尽管读采用了乐观机制，写仍然要保持悲观。但由于此时读者的读流程都是乐观的，因此我们便可以采用RDMA Write而非RDMA CAS来释放锁（Write Unlatch优化）。可以看得出来，在这种情况下Write Unlatch优化表现出了极高的提升，大大提高了悲观写的可扩展性。  

## 结论

总的来说，这篇文章深度分析了现有基于RDMA one-sided操作的同步技术的一些设计误区，并且给出了正确的悲观和乐观同步机制的实现和优化，最后测试了各种实现的性能，总结出各自合适的使用场景。  