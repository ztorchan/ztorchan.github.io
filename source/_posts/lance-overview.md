---
title: AI时代的多模态列式存储——LanceDB技术概览
tags:
  - 列式存储
  - Lance
  - LanceDB
  - AI存储
categories:
  - 列式存储
  - 向量数据库
date: 2025-07-21 00:52:47
math: true
---


> **特别说明：本文所讨论的技术设计均基于Lance v2和Lance v2.1**

在[*从Parquet到Arrow——列式存储概览*](https://ztorchan.com/2025/07/07/columnar-storage-overview/)一文中，我们回顾了过去十余年主流的开源列式存储格式。正如前文所述，数据存储的发展始终顺应着上层应用I/O模式的变迁。近年来，随着AI技术的爆发式发展，我们迎来了比大数据时代还“大”的AI时代，对数据I/O的需求也随之发生深刻变革。这使得原有的列式存储方案在某些场景下面临新的挑战。LanceDB应运而生，旨在应对这些挑战。本文将对LanceDB进行技术概览，探讨AI时代数据I/O面临的新问题，以及LanceDB的解决之道。

<!-- more -->

# LanceDB是什么？

<!-- https://mp.weixin.qq.com/s/nerkvBtFo5BM68dacsq17A -->
<!-- https://blog.csdn.net/younger_china/article/details/125905449 -->

LanceDB的愿景是**为AI时代构建统一的数据湖平台，满足多模态数据的管理需求**。随着AI时代的到来，图像、视频和音频等多模态数据的数据量及其访问需求急剧增长。然而，传统数据湖解决方案在高效管理这类数据时面临挑战，迫使相关企业不得不维护多个独立的数据系统，并通过构建复杂的数据交互链路来勉强支持多种模态数据的管理，效率低下。LanceDB正是在此背景下诞生，旨在弥补传统数据湖在多模态数据管理上的不足，为AI场景下的复杂数据提供统一高效的管理平台。

![图1 大数据框架层次](fig1-bigdata_framework_level.jpg)  

LanceDB构建在其开发的新型开源列式存储格式Lance之上，它高效管理多模态数据的能力正得益于Lance格式的针对性设计。如图1所示，Lance格式兼具了数据格式（Data Format）层和表格式（Table Format）层的特性，与传统Parquet/ORC数据格式和MetaStore/Iceberg表格式的单一层级功能有着本质区别。Lance格式不仅保持了高效扫描访问性能，还支持快速随机点查询。同时，它在添加新数据列时无需复制旧数据，实现了真正的Zero-cost Data Evolution特性。这些独特的特性和优势使Lance格式能够充分适应AI时代的数据负载需求，也支撑LanceDB成为当前领先的多模态数据湖平台解决方案。​

# AI工作负载的特点

再次强调，存储设计必须基于上层负载的访问模式进行。在开始后续的技术探讨之前，让我们先分析一下现代AI工作负载的特征及其遇到的问题。​

## 大量的点查询 

​​点查询​​指单次请求仅访问少量数据行的查询操作。当借助二级索引执行此类查询时，目标数据行往往呈​​离散分布​​，不属于同一个Page​。对于AI工作负载来说，无论是训练阶段还是推理阶段，数据搜索都是常用且重要的过程。为此，LanceDB​​从设计之初即定位为一个支持多样化搜索的向量数据库​​，例如语义搜索（Semantic Search）与全文本搜索（Full Text Search, FST）是其支持的核心搜索方式。​​这些搜索过程在底层最终都会转化为点查询操作，可想而知点查询是LanceDB无可避免的主要查询方式​​。

![图2 列式存储点查询（以Parquet为例）](fig2-point_lookups.png) 

然而，传统 Parquet 格式在处理点查询时存在显著瓶颈，主要原因可归结为以下两方面：

1. ​**​海量随机I/O带来的效率困境**​​：​​点查询会触发大量​的随机I/O请求，这种​​访问模式对存储系统极不友好​​。​​尤其对于当前主流的、基于 S3 等云存储构建的数据湖架构而言，其有限的IOPS能力难以高效应对此类海量随机I/O负载。​​

2. ​**​缺乏“可切片性”导致的冗余读取**​​：​​更深层次的原因在于，​​Parquet 的数据编码设计本身不具备“可切片性”​，即它无法单独提取并解码​​目标数据片段。为了成功解码所需数据，系统​​不得不读取目标数据所在的整个Page甚至Column Chunk​​。​​这种必须读取冗余大块数据的机制，在面对仅需少量数据的点查询时，必然会显著增加处理延迟和I/O开销。​

## 宽列

传统的大数据工作负载通常处理的是结构相对简单、尺寸较小的数据列，例如整数、浮点数或长度有限的字符串等基础数据类型。即使是其中最大的字符串，其尺寸通常也处于可控范围。然而，当下以AI工作负载更关注图像、音频、视频等多模态数据。这些工作负载往往将多模态数据的语义嵌入向量（如4KB大小的[CLIP embeddings](https://openai.com/index/clip/)）甚至是原始数据本身作为单独的列进行存储。这导致了“宽列”的形成。​

![图3 宽列](fig3-wide_columns.png) 

在传统的Parquet格式下，宽列的存在使得Row Group大小的设置面临两难困境：​​

1. 若​采用较小的Row Group Size以降低单个Row Group的数据量：​一方面，元数据开销必然增加，从而影响性能​；​​另一方面，与宽列关联的窄列数据量会变得极小，可能被放入远小于文件系统最佳读取尺寸的Page中，导致I/O效率低下。​​
​​
2. 若​维持较大的Row Group Size以避免上述问题：一方面，Parquet Writer在写入数据时需要消耗大量内存资源进行缓冲；​另一方面，由于当前主流Parquet Reader如pyarrow）以Row Group为并行单元处理数据，读取过程同样会占用大量内存。​

## 宽表

![图4 宽表](fig4-wide_schemas.png) 

特征工程作为AI工程的核心环节，会从原始数据中提取大量特征（通常多达数千个）​​。这使得许多AI工作负载拥有​​极其宽泛的数据模式（Schema）​​，即数据表包含大量数据列。尽管Parquet等列式存储格式提供了强大的​​列投影功能以最小化数据读取量​​，但读取时​​仍需完整加载所有列的模式元数据​​。这种元数据加载​​在低延迟场景下会引入显著开销​​，​​无法满足性能要求​​；同时，在跨多个文件缓存此类元数据时，​​极易造成内存占用激增，严重制约系统性能​​。

<!-- ## 元数据 -->

# Lance格式

如前所述，Lance格式是支撑LanceDB高效管理和访问多模态数据的关键设计。它​融合了传统数据格式与表格式的特性，承担了这两个层级的任务，对这两个层级分别进行了针对性优化。​​​​本章将深入解析Lance在数据格式层和表格式层的关键优化设计。​

<!-- https://blog.lancedb.com/lance-v2/ -->
<!-- https://blog.lancedb.com/file-readers-in-depth-parallelism-without-row-groups/ -->
<!-- https://blog.lancedb.com/lance-file-2-1-smaller-and-simpler/ -->
<!-- Lance: Efficient Random Access in Columnar Storage through Adaptive Structural Encodings -->
<!-- https://blog.lancedb.com/columnar-file-readers-in-depth-apis-and-fusion/ -->

## Lance数据格式：摒弃Row Group

传统的列式存储格式（如Parquet和ORC）均遵循PAX存储模型，在数据组织上引入了Row Group和Column Chunk的结构分层，同一Row Group内的每个Column Chunk都包含了相同行数的数据记录。而Lance数据格式的一项核心设计则在于摒弃了Row Group结构，从而获得更高的数据布局灵活性。

![图5 Lance文件数据格式](fig5-lance_data_format.png) 

Lance实际存储数据的是`.lance`文件。如图5所示，一个`.lance`文件主要由三部分组成:

- **Data Pages**：存储实际的列数据。​​每个Page归属于一个特定的列，而每个列可以包含多个数据页。​​​​更进一步地，每个数据页由一系列Buffer构成。​

- **Column Metadata**：当前文件下的每个列均有一个Column Metadata来描述属于该列的元信息，例如每个Page所包含的数据行数及其每个Buffer的offser和size。

- **Footer**：包含描述整个文件的全局元信息，例如文件格式版本、总列数、Column Metadata区域的位置等。

通过摒弃Row Group层级的限制，Lance数据格式实现了极高的数据布局灵活性：单个文件可以仅存储数据表的部分列（而非完整列集）；同一文件内不同列可以包含数量不同的Page；并且每个Page本身也可以包含不同行数的数据记录。这种设计的核心优势在于完美规避了传统格式中因宽列与窄列并存而引发的Row Group大小权衡难题，开发者仅需为每个列配置合适的Buffer大小即可。

此外，摒弃Row Group的设计也带来了其它的优化空间，我们将在后续章节中探讨。

## Lance数据格式：计算并行性与I/O并行性解耦

顺应着摒弃Row Group的设计，Lance进一步将计算和I/O解耦，提高并行能力。

我们先来看看传统的列式存储读取过程，下面是整个流程抽象出的伪代码：

``` C++
parallel for row_group in file:
  # 以Row Group为单位并行计算
  arrays = []
  parallel for column in row_group:
    # 以Column Chunk为单位并行I/O
    array = ArrayBuilder()
    for page in column:
      page = read_page()
      array.extend()
    arrays.append(array)
  batch = RecordBatch(arrays)
  yield batch
```

实际实现可能更复杂（例如Page无法独立解码、预取机制等），但以上伪代码揭示了传统架构的两个关键特征：

1. 数据读取流程中的计算（这里特指解码过程）并行性和I/O并行性是紧密绑定的；

2. 数据读取流程的并行度和上层应用设定的并行程度相关。例如，当上层应用设置了任务并行性度$X$（通常和CPU核心数相关），此时$X$就是对Row Group的访问并行度。同时涉及的数据列总共有$Y$列，那么I/O并行度将为$X \times Y$。

同时，我们还需要认识到两个事实：

1. 硬盘的I/O并行度存在上限​​。当并发I/O请求数量​​低于​​该上限时，硬件性能​​无法被充分利用​​；而当并发请求数量​​超出​​该上限时，​​请求排队将导致平均I/O延迟上升​​。

2. 由于不同列的数据类型各不相同，尺寸不一，存储相同行数的数据所需的Page数量在列与列之间也存在差异。

于是，基于上述特征和事实，我们可以去发现传统列式存储读取数据时存在的问题。在后面的场景中，我们数据读取拆解为两个核心阶段：数据I/O和解码计算两个关键过程，其中解码计算按固定行数分批执行。

![图6 传统列式存储数据读取顺序示例](fig6-traditional_column_storage_read.png) 

图6给出了一个数据读取时I/O顺序的示例。假设此时以$X$为3、$Y$为3的并行度对数据发起访问，由于各列包含的Page数量不同，导致每一轮次发起的I/O请求数量存在差异。如图所示，第一轮I/O因请求数量超过硬盘并行处理能力上限而导致延迟上升；而在第三轮，则因请求数量不足造成硬件性能浪费。​

![图7 数据I/O和解码计算重叠不足导致效率低下](fig7-inefficient_io_decode.png) 

更为关键的是，解码计算的前提是已完成读取的Page包含了各列固定行数批次的数据。即便是最理想的情况，每列至少需要完成一个Page的I/O读取。这意味着在平均情况下，系统必须等$X \times Y$（本例中为9）个I/O操作完成才能启动解码。如前所述，由于并发 I/O 请求过多导致的平均延迟大幅提升，会显著推迟解码计算的开始时间，造成 I/O 与计算的重叠不足，最终严重制约数据读取效率。​

![图8 理想的数据I/O和解码计算重叠](fig8-efficient_io_decode.png) 

![图9 Lance数据读取顺序示例](fig9-lance_read.png) 

​如图8所示，数据读取效率最优化的关键在于实现 I/O 操作与解码计算的最大化重叠。​而要实现这个，只需要按行号顺序访问各列对应的Page（同时控制并发I/O请求数），确保已读取的数据能最大程度地满足即时解码条件，从而实现最优效率。​于是，修改图6的数据读取顺序变为图9所示，以行号作为优先级，顺序读取各列Page即可。

![图10 Lance数据读取过程中的数据I/O和解码计算调度](fig10-lance_read_schedule.png) 

​​基于上述思路，Lance设计了如图10所示的数据读取调度机制：​​

1. ​调度线程（Scheduling Thread）​：根据查询任务和文件元数据，确定目标Page，生成对应的I/O请求并提交至I/O调度器，同时向解码线程发起解码请求。

2. ​​I/O调度器（I/O Scheduler）：​​按行号顺序调度​​所有待读取的Page，并根据​​硬盘性能动态控制并发I/O请求数量​​。

3. ​​解码线程（Decoder Thread）​​：​​接收已完成 I/O 的 Page 数据​​，执行解码计算任务。
   
​​通过这种设计，Lance 实现了访问过程中数据I/O与解码计算的彻底解耦，有效解决了传统列式存储在数据访问过程中的性能瓶颈，显著提升了整体效率。​

## Lance数据格式：随机访问友好的结构编码

前面提到，LanceDB的定位决定了其必然面临海量的点查询需求。这些点查询将会触发大量的随机I/O，对性能相当不友好。这个问题在面临复杂结构时将变得更为严重，例如`List<string>`类型列，`List`和`string`均为变长数据类型，嵌套使得该列的数据分布变得更为复杂，且还可能存在空值情况，这就为点查询引入了严重的寻址开销。

​​点查询的执行效率与数据格式的结构编码设计密切相关。结构编码决定了数据从逻辑表示到物理存储的映射规则。为深入理解这一关键机制，我们首先分析Parquet和Arrow的结构编码方案，对比二者在点查询场景下的数据访问差异（这部分内容在[前一篇文章](https://ztorchan.com/2025/07/07/columnar-storage-overview/)中也曾提及）。​

![图11 Parquet结构编码](fig11-parquet_structral_encode.jpg) 

Parquet为嵌套结构体的每个叶子节点列独立维护一个Column Chunk，每个Column Chunk则进一步划分为多个Page。Page是数据压缩的基本单元，其内部包含：用于编码数据重复性与存在性的Repetition/Definition Level、描述数据项基本属性（如长度）的元信息，以及数据本身。每当对特定行发起访问时，Parquet通过Page Offset Index可快速定位目标行所在的Page，进而加载、解压该Page并读取其内部数据完成访问。​

Parquet的结构编码十分契合于点查询，但也存在几个问题：

- 点查询会引入与Page大小等同的读放大。虽然Parquet并不实际为空值分配空间，且会将Page进行压缩存储，但Page的完整加载还是不可避免地引入了冗余数据读取。

- 面对较大尺寸的数据类型时，单个Page可容纳的数据行数减少，所需Page数量增加，随之直接提升Page Offset Index的内存开销。​

- Page的压缩和解压缩也会引入CPU计算开销。

![图12 Arrow结构编码](fig12-arrow_structral_encode.jpg) 

尽管Arrow的定位在于内存列式存储，但我们仍可以探究其结构编码应用于硬盘会发生什么。Arrow没有对数据进行细粒度的划分，并为空值分配物理空间，其通过直接的有效性位图和偏移量数组标识数据存在性与重复性（而非Parquet的Repetition/Definition Level）。以`List<string>`类型数据列为例（其布局如图12所示），读取单行数据至少触发5次独立的离散I/O操作，且这样的I/O次数会随着嵌套结构深度呈线性增长。​

该结构编码的优势在于消除了读放大问题，但代价是显著增加了I/O操作频次。在内存场景下，此类高频I/O不会成为主要瓶颈；然而当其应用于硬盘或云存储时，高频 I/O 将迅速耗尽底层存储的IOPS能力，形成严重的性能瓶颈。

为了结合上述两种编码结构的优势以更好地应对点查询，Lance为大尺寸数据类型和小尺寸数据类型分别设计了**Full Zip编码**和**Mini-block编码**，确保**面对任意数据类型的查询都能在1次（定长数据类型）至2（变长数据类型）次I/O中完成数据读取**。

![图13 Full Zip编码](fig13-full_zip.jpg) 

Full Zip编码专为大尺寸数据类型优化设计。为规避读放大问题，该编码摒弃了将多个数据项打包成Page的方式，而是按数据项粒度布局存储​。如图13所示，对于叶子列中的每个数据项，Full Zip编码将其 Repetition/Definition Level、数据项属性（如长度）及数据本身​连续存储于连续地址上​，我们可以将这样一组数据称为一个 Buffer。

由于任意数据列的类型和结构固定后，其Repetition/Definition Level的取值范围即被确定，因此Full Zip编码会将Buffer的Repetition/Definition Level通过bit pack编码压缩到1-4字节的固定长度。以`Struct<List<string>>`类型列为例，可以用低位3 bit表示Definition Level和高位1 bit表示Repetition level，共4 bit。图13展示了一个`['AB', 'C'], Null List, Null Struct, [Null], []`数据列基于Zip Full编码的实际存储布局。

上述数据布局设计对于定长数据类型而言已经可以实现在1次I/O内完成目标数据查询，因为此时Buffer是定长的，目标数据的地址偏移量可以直接计算得出。然而，该方案无法直接支持变长数据类型。为此，Full Zip编码还引入了一个Repetition Index数组存储所有数据项的地址偏移量，使得变长数据类型的点查询也仅需2次I/O便可完成。此外，扫描查询也可以通过计算（定长）或查询Repetition Index数组（变长）来获取目标范围来实现。

![图14 Full Zip编码](fig14-mini_block.jpg) 

Mini-block编码则专为小尺寸数据类型优化设计。对于小尺寸数据类型而言，适当的读放大是可以接受的，因此Mini-block 依旧采用分组存储策略，将数据划分为多个分组，我们称之为​​Chunk​​。每个Chunk同样包含了：Repetition/Definition Level、数据项属性（如长度）及数据本身。此外，Chunk头部还额外存储了该Chunk的数据项数量和总大小等统计信息。值得注意的是，这种结构设计与Parquet的Page结构高度相似，但Mini-block通常将Chuck大小控制在了4-8KB内（1-2个扇区大小），也将数据项个数控制在4096个以内，以避免过度读放大。

同样的，Mini-block也为维护了Repetition Index数组以快速确定目标数据在哪个Chunk内，以确保当嵌套结构内存在变长数据类型时依然能在2次I/O内完成对目标数据的读取。每个Chunk的Repetition Index包含N+1个非负整数，其中N是所需的最大随机访问嵌套级别。每个Chunk的Repetition Index由N+1个非负整数构成（N代表所需最大随机访问嵌套深度），例如对于`array[x][y]`访问模式需要维护3个整数：第一个整数记录该Chunk起始前已完成的顶层结构（数据行）数量；第二个整数记录该 Chunk起始前在当前顶层结构内已完成的二级结构数量；第三个整数记录该Chunk起始前在当前二级结构内已完成的三级结构数量。这听着有些抽象，图14给出了一个`List<List<string>>`列的例子。

![图15 Struct Packing编码](fig15-struct_packing.png) 

最后，Lance还给出了一种Struct Packing编码，它允许将部分列合并存储，从而转化为行存储模型。对于那些经常被一起访问的数据列，通过Struct Packing编码合并存储，可以有效降低访问过程中的I/O数据。

## Lance表格式：Zero-cost Data Evolution

在表格式层面，Lance依然借助摒弃Row Group所带来的数据布局灵活性，实现了Zero-cost Data Evolution。

![图16 数据表的演化](fig16-table_growth.png) 

在深入探讨该设计点之前，我们先来解释一下什么是数据表的演化。数据表在完成构建后并非是一成不变的，其结构会随着上层应用的运行而动态变化，发生表的垂直增长和水平增长：垂直增长就是数据行的增加，例如在发生订单交易后，我们需要往交易表里添加一行交易记录；水平增长则是数据列的增加，这在特征工程等场景中更为常见，例如为一张文本分析表新增“情绪倾向”的数据列。

![图17 数据表水平增长后旧列以默认值填充](fig17-horizontal_growth_with_default_value.png) 

问题发生在表的水平增长上。事实上，表结构的水平扩展并非新兴需求——长期运行的系统必然面临新增业务需求，例如电商平台推出下单返现活动时，需在交易表中新增“返现金额”列。由于历史交易记录无需该列数据，为其设置零值默认值是合理方案。在此场景下，理想方案应避免重写数据文件，仅需维护元数据并在查询旧数据时填充默认值。这正是Iceberg等传统表格式采用的标准方案。​

然而，AI工作负载对数据演化的要求远不止于此。其新增数据列的操作更为频繁，且通常旨在为所有历史数据添加特征描述（如特征向量）。在传统表格式下，由于Row Group必须包含完整列集，这迫使系统必须复制并重写所有数据文件。这一机制导致数据表演化开销巨大，在宽列存在的场景下更会进一步放大性能影响。​

![图18 Zero-cost Data Evolution](fig18-zero_cost_data_evolution.png) 

Lance则没有这个包袱了。摒弃Row Group的设计使其摆脱了单个文件必须包含所有数据列的束缚，从而可以采用创新的二维存储布局：将数据按行划分为多个Fragment，每个Fragment内的数据再按列组织为多个独立文件——每个文件包含固定行数的单列或多列数据。当新增数据列时，Lance无需复制或重写任何现有文件，只需单独创建对应的新文件，并可在必要时灵活合并或拆分这些数据文件。（从逻辑结构上看，Fragment和Row Group非常相似）

​![图19 轻量化的数据删除](fig19-zero_cost_data_deletion.png) 

更进一步地，Lance为数据行删除也进行了轻量化设计。它在每个Fragment内专门维护一个Deletion文件​​，用于标记每行数据的删除状态。通过这种设计，Lance在删除数据行时仅需更新元数据（Deletion文件），而无需修改实际的数据文件本身。​

<!-- https://blog.lancedb.com/designing-a-table-format-for-ml-workloads/ -->

# Lance之上的LanceDB

<!-- https://lancedb.github.io/lancedb/ -->

再简单介绍一下LanceDB。

​![图20 LanceDB生态](fig20-lancedb.png) 

LanceDB是基于Lance格式构建的开源向量数据库，其核心代码主要由Rust编写实现。Lance格式的设计在很大程度上解决了多模态数据存储、点查询随机访问和频繁表演化的效率问题，这使得LanceDB非常契合于当下AI工作负载的数据管理需求。同时，LanceDB为主流数据框架和SQL引擎提供广泛接入支持，便于集成现有数据生态系统。这些优势正推动LanceDB成为当下构建多模态数据湖的答案之一。​

​![图21 LanceDB存储底座选择](fig21-lancedb_storage_tradeoffs.png) 

同时，LanceDB也为各种形式的存储底座提供了支持，从高成本低延迟的本地SSD到低成本高延迟的S3云存储。广泛的存储底座选择足以满足各类场景的需要，给了用户充分考量权衡的空间。

​![图22 向量搜索](fig22-lancedb_vector_search.png) 

LanceDB也提供了包括向量搜索（KNN/ANN搜索）和FTS搜索在内的多种数据搜索能力，其中向量搜索作为AI时代核心的数据访问范式，也是是其关键优势所在。​