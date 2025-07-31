---
title: 从Parquet到Arrow——列式存储概览
tags:
  - 列式存储
  - Parquet
  - ORC
  - Arrow
categories:
  - 列式存储
date: 2025-07-07 00:49:02
---


数据存储总是需要顺应上层应用的IO模式而发展的。进入大数据时代的数十年里，OLAP分析性系统逐渐替代OLTP事务型系统成为了当下主流的数据系统，对OLAP更为友好的列式存储也随之成为研究热点得以野蛮发展。本文以存储模型的发展出发，简述列式存储设计的关键点，并着重介绍当下主流的开源列式存储格式，为读者提供一张全局视图。

<!-- more -->

# 存储模型的发展

对于人们的日常生活生产来说，二维表是最直观的数据管理方式。以公司职员管理为例子，下面一张简单的二维表就能实现我们对职员管理的增删改查需求。

![图1 二维表](fig1-2D_table.jpg)  

但和我们对数据的直观认知逻辑不同，数据的存储是一维的，也即存储地址是单一维度线性延申的。因此，如何把二维的数据存储到一维的空间中也成为了一个值得探讨的问题。

## NSM模型

NSM（N-ary Storage Model）存储模型​​，即我们常说的​​行存储​​，其核心思想是**将数据按​​条目（行）为单位连续存储​**​。如下图所示，表格中的每一行数据作为一个完整单元，被依次存入NSM Page中，而同一行内不同列的数据则存储在相邻的物理地址上。NSM Page的尾部通常会构建索引结构​​，以实现对每条数据的高效定位。这种存储模型高度契合我们的直觉认知。

![图2 NSM存储模型](fig2-NSM.jpg)  

NSM模型将同一行的数据组织在一起，具有以下显著优点：

- 单行数据的物理连续性使插入和删除操作性能更高
- 对`SELECT * from ...`之类的完整条目查询更加友好，无需进行数据重建

但该模型也存在其固有缺点：

- 当仅需部分列时仍需读取整行数据，造成严重的读放大和缓存低效
- 同一行往往存在不同的数据类型，数据压缩效率低下

传统的数据系统主要面向OLTP场景，它们更注重数据操作的实时性、高并发以及ACID事务保证，例如银行业务、票务业务、订单业务等等。由于OLTP系统常常进行完整数据条目的增加、删除和查询，因此NSM模型天然适合OLTP场景。

## DSM模型

随着大数据时代的到来，OLAP场景需求逐渐显现并迅速增长，这类分析型系统更关注海量数据的快速聚合与分析能力，典型应用包括商业智能、数据仓库、报表统计等。由于OLAP系统通常需要对大型数据集进行复杂的多维度分析，往往只涉及部分列的计算和聚合，这使得NSM行存储模型难以满足海量数据的吞吐需求。

DSM（Decomposition Storage Model）模型与NSM模型相反，其核心思想是**将数据按列而非按行组织存储，每个列单独形成存储单元**。如下图，在NSM模型下，同一列的所有数据值连续存储在相邻物理地址，而同一行的不同列数据则分散存储在不同的Page中，每个列单元维护专属的访问结构和元数据信息。

![图3 DSM存储模型](fig3-DSM.jpg)  

这样的数据组织带来了与NSM模型截然相反的优缺点，其优点在于：

- 当查询仅涉及部分列时可实现"列裁剪"式读取，避免无关列的I/O开销，同时提高缓存效率
- 同一列的数据大多为统一数据类型，进行数据压缩时能得到更高的压缩效率
- 批量处理同列数据时能充分发挥现代CPU的SIMD指令集并行计算能力

反之，其缺点在于：

- 对完整条目的点查询需要从多个列单元收集数据并重建完整行，引入重组开销
- 事务性写入需要更新多个分散的存储区域，引入大量随机小I/O
- 频繁更新的场景还会引发列存储的"压缩抖动"问题

由于OLAP场景多数操作是对部分特定列的海量查询操作，因此DSM模型更契合于OLAP分析型系统。

## PAX模型

NSM模型和DSM模型相互对立，各有优劣，分别适应于极端的OLTP和OLAP场景。但现实系统往往需要兼顾两种场景，因此催生了PAX（Partition Attributes Across）模型的诞生。PAX模型的核心思想是，**先对数据条目按行进行分组，划分到不同的存储区域，再将同一分组内的数据按列组织存储**。

如下图所示，PAX模型的存储结构采用两级分区设计。它首先将数据条目按行进行了划分，使同一组的数据条目存储在同一个PAX Page中。进一步地，PAX Page被划分为了多个minipage，分别对应于不同的列。类似于DSM模型，每一列的数据分别被放置在不同的minipage中，使得它们仍然存放在连续地址上。

![图4 PAX存储模型](fig4-PAX.jpg)  

PAX模型是对NSM模型和DSM模型一种折中设计，这使得它有效结合了二者的优势：

- 面对特定列进行扫描查询时，系统只需要顺序访问每一个PAX Page的对应minipage，从而提高缓存效率
- 同一数据条目的各列数据处在同一个PAX Page中，保持了局部性。在面对完整条目的查询时，避免了数据重建过程中的跨页访问
- 数据条目的增加和删除所需进行的数据写入发生在同一个PAX Page，缓解了过程中的随机小I/O问题

# Parquet/ORC——磁盘列式存储

经过前文对NSM、DSM、PAX三种存储模型的简要分析，我们已建立起列式存储的基础认知框架。本章就当前主流的两种开源列式存储格式——Apache Parquet与Apache ORC，通过多维度对比探究其架构设计理念、实现原理及核心优势特性。

先将二者的文件格式示意图摆出来，接下来我们对其中的细节一一探讨。

![图5 Parquet和ORC的文件格式](fig5-parquet_orc_format.jpg)  

## 存储格式

Apache Parquet和Apache ORC的存储格式本质上都基于PAX模型构建，严格来说并非纯列式存储，而是行列混合的存储架构。正因这种同源设计，它们在存储结构上存在诸多相似之处。为了清晰理解，我们明确两个核心概念：PAX模型中划分出的数据条目分组称为​​行组（Row Group，上章图中红蓝色方块）​​，而行组内每列数据对应的物理存储单元称为​​列块（Column Chunk，上章图中红色方块）​​。对应到Parquet和ORC中的概念如下表所示：

<style>
.center-table {
  display: table;
  margin-left: auto;
  margin-right: auto;
}
</style>


<div class="center-table">

|  | **Row Group** | **Column Chunk** |
|:--------:|:--------:|:---------:|
| **Parquet** | Row Group | Column Chunk |
| **ORC** | Stripe | Row Column |

</div>

从图5中，我们可以看出二者的设计在大体上是相似的。Footer中包含了文件级别（例如表结构等）和Row Group级别（例如偏移和Zone Map统计等）的元数据，它作为文件访问的起点，为下一步的数据检索提供了指引。多个Row Group组成了文件的主体数据区，每个Row Group内部遵循PAX模型的设计原理，通过Column Chunk对列数据进行物理组织。对于Parquet而言，Column Chunk还会进一步划分为多个Page，作为压缩单元（后续章节提及）。

尽管在存储格式上如此相似，二者在对如何划分Row Group采取了不同的选择：

- Parquet基于行数来确定Row Group的大小，以此确保单个Row Group有足够多行数据来支持高效的向量查询，但在内存开销上却不可控（因为Column Chunk是I/O单元，更大的Row Group也意味着更大的Column Chunk）

- ORC使用了固定物理存储大小的Row Group以控制内存开销，但当遇到宽表场景（单行含数万列）时，单个Row Group内的有效行数大幅减少，反而降低了批处理效率。

## 过滤器

过滤器是实现高效查询的核心机制，它通过判断目标数据在特定存储单元中的存在性，避免不必要的I/O操作。

最常用的过滤器有两种：（1）**​​Zone Map**​​：它存储了指定存储单元的关键统计指标——最大值、最小值和数据条目总数，通过判断目标值是否落在最小-最大区间内，即可高效预判该存储单元是否存在目标数据；（2）**​​Bloom Filter**​​：它是基于概率的位映射过滤器，以极小的存储空间代价存储了数据"指纹"。当查询特定键值时，通过多次哈希计算检测对应比特位状态，若存在未激活位则确认数据必然不存在，反之则有较大概率存在。

Parquet与ORC均支持​​至少两级的Zone Map统计​​：**文件级别**（文件中的每列数据一个Zone Map）和**Row Group级别**（*Row Group中的每个Column Chunk一个Zone Map）。（特别说明：虽然从实现逻辑看，Row Group级别的Zone Map严格对应Column Chunk粒度，但为遵循官方文档表述且配合图5的图示结构讲解，本文仍采用"Row Group级别"的通用描述以避免概念混淆。*）正如图5所示，这两个级别的Zone Map均处于Footer区域，以集中式布局实现高速检索。

进一步地，Parquet和ORC还为更小粒度的存储单元提供了Zone Map支持。Parquet支持构建Page级别的Zone Map，以存储更细粒度的统计信息。在2.9.0版本以前，Parquet将其离散地存储在每个Page的Page Header中，这导致了查询过程中会引发大量的随机小I/O，性能受限；为此，在2.9.0版本之后，Parquet提供了可选的Page Index，将Page级别的Zone Map集中存储在了Page Index区域中，解决了随机小I/O问题。ORC采用可配置行数的动态Zone Map（默认每10000行一个Zone Map），存储在了每个Row Group的Index区域中。

Bloom Filter是Parquet和ORC额外的可选过滤器。Parquet为每一个Column Chunk构建单独的Bloom Filter（采用了[Split Block Bloom Filter](https://dl.acm.org/doi/10.1145/1498698.1594230)的优化实现），存储在了连续区域。ORC则提供了与Zone Map相同的可配置行数粒度Bloom Filter，同样存储在了每个Row Group的Index区域中。

## 嵌套数据结构

![图6 嵌套数据结构表示](fig6-nested_data_encoding.jpg)  

相较于高度规范化的结构化数据，半结构化数据天生具备更强的灵活性，能够高效容纳各类信息表达，同时提供更高扩展性。为了支持半结构化数据构建，Parquet和ORC分别设计了不同的嵌套数据结构实现方案。

Parquet的嵌套数据结构基于[Dremel](https://dl.acm.org/doi/10.14778/1920841.1920886)。如图6所示，Parquet仅仅将嵌套数据的叶子节点作为独立的数据列进行存储，并为每个数据值绑定两个整型参数：Repetition Level（R）和Definition Level（L）：

- Repetition Level表示当前数据值和前一个数据值在嵌套路径的重复深度（例如图中第一行数据中的tag列，b与其前面的a共享同一个tags，则R为1；而接下来的缺失值和c都与其前面的值属于不同的行，则R为0）
- Definition Level则仅对缺失值有效，表示当前缺失数据最深的有值层级，也即数据缺失发生层级的前一层级（例如图中第二行数据，其name.first为缺失值，但name仍然存在，则D值为1）

这两个参数通过编码重复类型（repeated）的层级变化与可选类型（optional）的空值状态，完整描述了复杂嵌套结构。基于这两个参数序列，Parquet得以通过一个有限状态机实现列式存储数据向原始半结构化数据的重构。

ORC的嵌套数据结构则更为直接。嵌套数据结构上的每一个叶子节点都会成为独立存储的数据列，并为两类特殊数据绑定标识。对于重复类型数据，额外存储整型数据标记列表项数量；对于可选类型数据，则维护布尔值标记其存在状态。

相较而言，由于Parquet的方案只需实际存储叶子节点的列数据，它能够在重构数据时减少读取的列，并且在嵌套结构较为复杂、层次较深的场景下，存储开销较小。但同时，由于Parquet为每个数据列都绑定了Reptition Level和Definition Level，在嵌套结构较简单的情况下，它的存储开销反而更高。

## 数据压缩与编码

压缩（Compression）是降低存储成本的有效手段，其处理过程是完全​​数据类型无关​的​——所有数据均被视为原始字节流，因此无论何种文件格式都可利用压缩算法缩减体积。Parquet和ORC均提供压缩支持：先将数据​​按固定块尺寸（如256KB）分割​​，再对每个数据块​​独立应用压缩算法​​。二者均支持Snappy、LZO等多种算法，供用户根据场景​​在压缩比与（解）压缩速度之间权衡​​。但事实上，在列式存储场景下，压缩机制所带来的效益并不明显。

编码（Encoding）则是一种更轻量级的数据压缩方式，其算法执行与数据类型息息相关，往往能在多数数据类型一致的情况下，达成比通用压缩更优的综合效能（压缩率与处理速度的平衡）。这种特性与列式存储架构天然契合——由于同一列数据具有完全相同的类型且连续存储在物理相邻区域，编码机制能够在此场景中充分发挥其高效性。

Parquet默认会对所有数据列（无论数据类型）应用​​DICT编码​​。该编码在应对大尺寸数据类型（如长文本）及高重复值场景时能实现优异压缩率；随后Parquet会采用Bitpacking或RLE编码对生成的字典数据进行二次压缩，实现压缩效率的进一步优化。

ORC默认情况下只对字符串类型数据应用DICT编码。对于整数类型列和DICT编码所产生的字典，ORC采用了一种贪婪的编码选择机制：ORC根据序列长度情况，按照预设的规则从RLE、Delta Encoding和Bitpacking中选择其中一种进行数据编码。这样的机制有助于达成更高的数据压缩率，但解码速度也相应有所下降，并且可能会造成较多的存储碎片。

# Arrow——内存列式存储

Apache Arrow于2016年2月17日正式成为Apache顶级项目，与聚焦磁盘存储的Parquet和ORC不同，Arrow是专为​​内存计算​而​设计，其愿景在于建立跨异构系统的​​零拷贝数据交换标准​​。本章将主要探究Arrow的设计目标、特性及其内存数据格式。

## 目标与特性优势

![图7 未使用Arrow的异构系统数据传输](fig7-before_arrow.png)  

在大数据生态中，数据工程作为复杂的系统工程，需要整合多源异构的数据输入与多样化的处理工具。当Spark、Pandas等异构计算系统需要访问不同来源、不同格式的数据并进行跨系统、跨语言交互时，由于缺乏统一的数据平台和标准，系统间数据共享必须经历序列化、传输、反序列化的三段式流程。这种繁琐的机制不仅显著增加计算负载，还因数据的拷贝和传输而显得极其低效。

![图8 使用Arrow的异构系统数据共享](fig8-after_arrow.png)  

在这样的背景下，Arrow应运而生，其旨在构建跨平台、跨语言的​​统一内存数据标准​​，通过规范化的内存列式数据结构实现异构系统间​​零拷贝数据共享​​，从根本上消除序列化/反序列化与数据传输的双重开销。与此同时，其硬件协同的设计适配现代CPU特性（如SIMD指令集与缓存预取），显著提升数据计算效能。

基于以上目标，Arrow的设计有着以下几点显著的特性和优势：

- **多语言生态支持​**：Arrow提供多语言原生API（Python、Java、C++等），并与Pandas、Spark等生态深度集成。

- **零拷贝数据交互**：在多语言支持的基础上，Arrow实现了真正的​​语言无关内存标准化​​——在任何编程环境下，其内存数据布局保持完全一致的结构设计。这种统一的物理表示使多语言系统能​​直接共享内存数据​​（如Java程序无缝访问C++构建的Arrow对象），彻底免除序列化/反序列化操作。这在真正意义上实现了零拷贝的数据交互，不仅减少CPU开销，还大幅降低跨语言/跨框架交互的延迟，提升分布式计算效率。

- **硬件高效**：Arrow采用列式内存布局，将同一列数据连续存储，显著提升数据分析效率。列式布局结合现代硬件特性（如CPU缓存、向量化指令），加速数据密集型计算。

- **与磁盘列式存储互补**：Arrow并非与Parquet和ORC这类磁盘列式存储格式互斥，它专注内存计算，与磁盘列式存储形成互补：磁盘列式存储优化磁盘存储压缩率，Arrow优化内存处理效率。二者结合可覆盖数据从存储到计算的全生命周期高性能需求。

## 面向内存的列式数据格式

面向内存的列式数据格式是Arrow的重要组成部分之一，同时也是实现跨系统高效数据共享的基础。本节着重讲解Arrow的内存数据格式，包括定长类型数据格式、变长类型数据格式、列表数据格式、和结构体数据格式。

### 定长类型数据格式

``` C++
type FixedColumn struct {
  data       []byte // 数据
  length     int    // 数据长度
  nullCount  int    // 缺失值个数
  nullBitmap []byte // 缺失值位图
}
```

定长数据类型的数据列可以由上述数据结构管理。`data`是一块连续内存空间，用于存储真正的数据，而`length`则表示了该列数据个数。为了支持可选类型数据，`nullCount`和`nullBitmap`则被引入用于标记缺失值。

例如，一个`int32`类型的数据列`[1, null, 2, 4, 8]`的内存格式如下：


```
* Length: 5, NullCount: 1
* NullBitmap:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00011101                 | 0 (padding)           |

* Data:

  | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63           |
  |-------------|-------------|-------------|-------------|-------------|-----------------------|
  | 1           | unspecified | 2           | 4           | 8           | unspecified (padding) |
```

值得注意的是，即使存在缺失值，Arrow定长类型数据格式也会保留其对应的物理内存空间。这种做法虽会增加内存占用，但其核心优势在于​​访问效率的极致优化​​——无需通过`nullBitMap`执行字段偏移量计算，确保任意字段的随机访问时间复杂度稳定为O(1)。相比内存资源的消耗，Arrow将​​数据访问性能​​置于更高优先级。


### 变长类型数据格式

``` C++
type VarColumn struct {
  data       []byte   // 数据
  offsets    []int64  // 每个数据值的起始偏移
  length     int      // 数据长度
  nullCount  int      // 缺失值个数
  nullBitmap []byte   // 缺失值位图
}
```

相较于定长数据，变长数据的存储额外引入了`offsets`用于表示每个数据值的起始偏移量，以此实现对变长数据的标识。每个数据值的长度则可以通过`offsets[i+1] - offsets[i]`得出。

例如，一个`string`类型的数据列`['joe', null, null, 'mark']`的内存格式如下：

```
* Length: 4, NullCount: 2
* NullBitmap:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001001                 | 0 (padding)           |

* Offsets:

  | Bytes 0-19     | Bytes 20-63           |
  |----------------|-----------------------|
  | 0, 3, 3, 3, 7  | unspecified (padding) |

 * Data:

  | Bytes 0-6      | Bytes 7-63            |
  |----------------|-----------------------|
  | joemark        | unspecified (padding) |
```

### 列表数据格式

``` C++
type List<T> struct {
  data       T        // 数据
  offsets    []int64  // 每个数据值的起始偏移
  length     int      // 数据长度
  nullCount  int      // 缺失值个数
  nullBitmap []byte   // 缺失值位图
}
```

列表`List<T>`表示数据列的每个数据值都是一个`T`类型数据的列表，本质上是一种嵌套数据。与变长类型数据格式相似，唯一不同的是其`data`字段由单纯的真正数据变成了`T`。这是事实上是一次解包，也是嵌套发生所在。

例如，一个`List<int8>`的数据列`[[12, -7, 25], null, [0, -127, 127, 50], []]`的内存格式如下：

```
* Length: 4, NullCount: 1
* NullBitmap:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001101                 | 0 (padding)           |

* Offsets(int32)

  | Bytes 0-3  | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63           |
  |------------|-------------|-------------|-------------|-------------|-----------------------|
  | 0          | 3           | 3           | 7           | 7           | unspecified (padding) |

* Data (int8Array):
  * Length: 7,  NullCount: 0
  * NullBitmap: Not required
  * Data (int8)

    | Bytes 0-6                    | Bytes 7-63            |
    |------------------------------|-----------------------|
    | 12, -7, 25, 0, -127, 127, 50 | unspecified (padding) |
```

可以看到在`List<int8>`的`data`下存放的是`int8`定长类型的完整数据表示，它将原有的`List<int8>`所有数据进行了一次解包，构成了连续的`int8`数组。

进一步地，`List<List<int8>>`类型的`[[[1, 2], [3, 4]], [[5, 6, 7], null, [8]], [[9, 10]]]`的内存格式表示如下：

```
* Length 3
* Nulls count: 0
* Validity bitmap buffer: Not required
* Offsets buffer (int32)

  | Bytes 0-3  | Bytes 4-7  | Bytes 8-11 | Bytes 12-15 | Bytes 16-63           |
  |------------|------------|------------|-------------|-----------------------|
  | 0          |  2         |  5         |  6          | unspecified (padding) |

* Values array (`List<Int8>`)
  * Length: 6, Null count: 1
  * Validity bitmap buffer:

    | Byte 0 (validity bitmap) | Bytes 1-63  |
    |--------------------------|-------------|
    | 00110111                 | 0 (padding) |

  * Offsets buffer (int32)

    | Bytes 0-27           | Bytes 28-63           |
    |----------------------|-----------------------|
    | 0, 2, 4, 7, 7, 8, 10 | unspecified (padding) |

  * Values array (Int8):
    * Length: 10, Null count: 0
    * Validity bitmap buffer: Not required

      | Bytes 0-9                     | Bytes 10-63           |
      |-------------------------------|-----------------------|
      | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 | unspecified (padding) |
```

### 结构体数据格式

有了上述几种数据的基础表示，结构体数据格式只需要按照其具体定义进行相应的嵌套即可。

例如，`Struct<VarBinary, Int32>`的数据`[{'joe', 1}, {null, 2}, null, {'mark', 4}]`，包含了`['joe', null, 'alice', 'mark']`和`[1, 2, null, 4]`两个子列，其内存格式如下：

```
* Length: 4, Null count: 1
* Validity bitmap buffer:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001011                 | 0 (padding)           |

* Children arrays:
  * field-0 array (`VarBinary`):
    * Length: 4, Null count: 1
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001101                 | 0 (padding)           |

    * Offsets buffer:

      | Bytes 0-19     | Bytes 20-63           |
      |----------------|-----------------------|
      | 0, 3, 3, 8, 12 | unspecified (padding) |

     * Value buffer:

      | Bytes 0-11     | Bytes 12-63           |
      |----------------|-----------------------|
      | joealicemark   | unspecified (padding) |

  * field-1 array (int32 array):
    * Length: 4, Null count: 1
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001011                 | 0 (padding)           |

    * Value Buffer:

      | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-63           |
      |-------------|-------------|-------------|-------------|-----------------------|
      | 1           | 2           | unspecified | 4           | unspecified (padding) |
```

# 参考文献

[A Deep Dive into Common Open Formats for Analytical DBMSs (VLDB`23)](https://dl.acm.org/doi/10.14778/3611479.3611507)

[An Empirical Evaluation of Columnar Storage Formats (VLDB`23)](https://dl.acm.org/doi/10.14778/3626292.3626298)

[Apache Parquet Documentation](https://parquet.apache.org/docs/overview/)

[Apache ORC Specification v1](https://orc.apache.org/specification/ORCv1/)

[Apache Arrow Specifications](https://arrow.apache.org/docs/format/index.html)