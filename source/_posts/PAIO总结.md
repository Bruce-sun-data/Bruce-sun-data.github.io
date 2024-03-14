---
title: PAIO总结
date: 2024-03-14 10:52:43
tags:
typora-root-url: ..
categories:
    - 论文阅读
---

# PAIO: General, Portable I/O Optimizations With Minor Application Modifications

论文原地址https://www.usenix.org/conference/fast22/presentation/macedo

## 摘要

PAIO是一个框架，为不同的应用程序实现**可移植**的I/O策略和**优化**，开发人员只需要对**原始的代码库**进行**少量修改**。主要思路是：如果我们能够在请求流经I/O堆栈的不同层时拦截和区分请求，我们就可以在不显著更改层本身的情况下执行复杂的存储策略。PAIO用到了软件定义存储的思想，构建了数据平面stages(mediate和优化跨层的I/O请求)和控制平面(根据不同的存储策略对各个阶段进行协调和微调)。用两个用例展示了PAIO的性能和适用性。

**第一种**方法将基于行业标准lsm的键值存储的第99百分位延迟提高了4倍。**第二个**是确保共享存储环境下动态的每个应用程序带宽保证

## Introduction





















