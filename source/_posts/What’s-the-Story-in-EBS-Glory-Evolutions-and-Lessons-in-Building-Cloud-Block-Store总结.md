---
title: >-
  What’s the Story in EBS Glory:Evolutions and Lessons in Building Cloud Block
  Store总结
date: 2024-06-06 14:56:24
typora-root-url: ..
categories:
    - 论文阅读
tags:
---

# Evolutions and Lessons in Building Cloud Block Store

## 摘要

架构的进化：EBS1是简便的设计。EBS2更关注高性能和空间效率。EBS3关注于减少网络流量的放大。

四个内容的发展教训和经验：

- 在延迟、吞吐量、IOPS以及容量四个方面实现高弹性
- 通过最小化单个、区域和全局故障事件的爆炸半径来提高可用性
- 在多种硬件卸载方案中识别挑战和主要的性能取舍
- 识别解决方案的利弊，解释为什么看似有的方案在实际中不起作用

## 介绍

EBS体系结构最突出的特点是计算到存储的分解，其中虚拟机(计算端)和磁盘(存储端)不在物理上共存，而是通过数据中心网络相互连接。

==EBS1==有两个值得注意的设计：

- 虚拟磁盘到物理磁盘的原地(立即)更新
- 虚拟磁盘独占管理(每个VD都被一个BlockServer管理)

缺点：直接的虚拟化会导致严重的空间放大和性能瓶颈

==EBS2==改进了两个地方：日志结构的设计以及VD分割

- 首先将盘古分布式文件系统作为了存储后端。并重新设计了VD的写入为顺序添加。对写操作使用三副本，在GC的时候就可以在后端执行数据压缩(data compression)和纠删编码(erasure coding)
- 其次，EBS2以32GB为一个segment分割VD，从而将VD和BlockServers之间的映射从VD级转移到Segment级

优点：空间放大系数从3(3个副本)降为了1.29。提高SSD功率之后，一个VD可以达到1M IOPS和4000MBps的带宽以及100μs的平均延迟。

缺点：流量放大因子增加到4.69，即3(前台复制写入)+ 1(后台GC读取)和0.69(后台EC/压缩写入)

==EBS3==通过两种技术(Fusion Write Engine 和 FPGA-based hardware compression)使用online EC/压缩来减少流量的放大：

- FWE聚合不同segments的写请求来满足EC和compression的大小需求

- EBS3将计算密集型的compression操作卸载到了一个定制的FPGA上来加速

优点：将存储放大因子从1.29降为了0.77(压缩之后)，将流量放大因子从4.69降为了1.59，同时保障了和EBS2一样的性能。

下图展示了只要的技术升级和硬件使用。EBS的演变显示了重心从性能到空间和流量效率的转变。不过，简单地改变高层架构是不够的。从**弹性、可用性、硬件卸载、其他方案**四个方面介绍经验

![image-20240606193251089](/images/What’s-the-Story-in-EBS-Glory-Evolutions-and-Lessons-in-Building-Cloud-Block-Store总结/image-20240606193251089.png)

==elastic(弹性)==  : 提供具有不同性能和不同容量VDs的能力。有两个关键方面：**确定边界**和**细粒度调优**。平均延迟和尾延迟分别由硬件开销以及软件处理决定。因此，我们构建了相应的解决方案，包括EBSX，一种由持久内存支持的一跳架构，用于最小化平均延迟，以及使用专用线程进行I/O以减轻尾部延迟。吞吐量和IOPS影响是差不多的。在前端的BlockClient中，将processing从内核移动到用户层再卸载到FPGA的硬件中。在后端的BlockServer中，通过利用高并行性来实现高效的吞吐量/IOPS控制。对于**空间弹性**来说，EBS提供多种大小以及可调整和快速克隆。

==availability(可用性)== 大规模部署情况下是否存在问题。问题分位三类：单个(一个)、区域(几个)、全局(所有)集群中的中断服务。

















