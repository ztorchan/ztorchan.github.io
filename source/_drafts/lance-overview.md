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
---

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

在传统的 Parquet 格式下，宽列的存在使得 Row Group 大小的设置面临两难困境：​​
1. 若​采用较小的Row Group Size以降低单个Row Group的数据量：​一方面，元数据开销必然增加，从而影响性能​；​​另一方面，与宽列关联的窄列数据量会变得极小，可能被放入远小于文件系统最佳读取尺寸的Page中，导致I/O效率低下。​​
​​
2. 若​维持较大的Row Group Size以避免上述问题：一方面，Parquet Writer在写入数据时需要消耗大量内存资源进行缓冲；​另一方面，由于当前主流Parquet Reader如pyarrow）以Row Group为并行单元处理数据，读取过程同样会占用大量内存。​

## 宽表

![图4 宽表](fig4-wide_schemas.png) 

特征工程作为AI工程的核心环节，会从原始数据中提取大量特征（通常多达数千个）​​。这使得许多AI工作负载拥有​​极其宽泛的数据模式（Schema）​​，即数据表包含大量数据列。尽管Parquet等列式存储格式提供了强大的​​列投影功能以最小化数据读取量​​，但读取时​​仍需完整加载所有列的模式元数据​​。这种元数据加载​​在低延迟场景下会引入显著开销​​，​​无法满足性能要求​​；同时，在跨多个文件缓存此类元数据时，​​极易造成内存占用激增，严重制约系统性能​​。

<!-- ## 元数据 -->

# Lance格式

如前所述，Lance格式是支撑LanceDB高效管理和访问多模态数据的关键设计。它​融合了传统数据格式与表格式的特性，承担了这两个层级的任务，对这两个层级分别进行了针对性优化。​​​​本章将深入解析Lance在数据格式层和表格式层的关键优化设计。​

## Data Format

<!-- https://blog.lancedb.com/lance-v2/ -->
<!-- https://blog.lancedb.com/file-readers-in-depth-parallelism-without-row-groups/ -->
<!-- https://blog.lancedb.com/lance-file-2-1-smaller-and-simpler/ -->
<!-- Lance: Efficient Random Access in Columnar Storage through Adaptive Structural Encodings -->
<!-- https://blog.lancedb.com/columnar-file-readers-in-depth-apis-and-fusion/ -->


**（1）大量的点查询**



**（2）**

### 无行组

### 编码


## Table Format



<!-- https://blog.lancedb.com/designing-a-table-format-for-ml-workloads/ -->

# Lance之上的LanceDB

<!-- https://lancedb.github.io/lancedb/ -->
