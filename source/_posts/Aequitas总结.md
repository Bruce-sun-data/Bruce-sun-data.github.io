---
title: Aequitas总结
date: 2024-03-13 16:51:51
tags:
typora-root-url: ..
categories:
    - 论文阅读
---

# Aequitas: Admission Control for Performance-Critical RPCs in Datacenters

## 设计

![image-20240313184932130](/images/Aequitas总结/image-20240313184932130.png)

### Overview

Aequitas在RPC层实现，上层是应用程序，下层是网络层或者传输层。与拥塞控制算法不冲突。并且利用商品交换机中的WFQ。

应用程序发出RPC的时候注释优先级，并将该优先级类映射到请求的QoS类。<u>每当一个RPC完成之后，Aequitas计算它的RNL然后反馈到admission control algorithm。之后对比RNL的SLO目标和RNL的真实值，algorithm会为RPC运行时所在的QoS调整每个destination-host允许的流量</u>。admission control在所有主机之间以完全分布式的方式执行。

Aequitas使用admit probability来确定RPC是否应该在该QoS级别或者降级到较低的QoS级别。降级信息被显式通知回应用程序，作为调整其RPC优先级的提示

## 计算分析 网络验算

RPC网络延迟由带宽和排队延迟决定。本节提供了一个理论特征，该理论指导Aequitas的设计：，通过控制WFQ实现的各自QoS上允许的流量来控制跨优先级类的RPC网络延迟从而提供差异化的slo。

### WFQ 带宽和排队延迟分析

WFQ能够帮助维持RNL SLOs.

![image-20240313185519732](/images/Aequitas总结/image-20240313185519732.png)代表N个QoS，和WFQs的权重一一对应。每个QoS的最低保障提交率是$g_i$.$g_i={\frac{\phi_i}{\sum_j\phi_i}r}$，假设i越小，WFQ的权重越大。如果对QoS类的瞬时需求低于$g_i$，那么该QoS类对应的流不会受到排队延迟干扰。相应地，当其他QoS类的总需求低于其带宽份额时，某个QoS类的带宽份额可能会超过上述速率。

延迟的QoS可以描述为一个关于其自身队列长度的函数，独立于其他QoS级别的队列长度和请求到达情况。当发送方受到漏桶速率限制器约束时，计算最坏情况下的排队延迟是可行的。在此基础上，我们使用网络演算概念来计算QoS类中给定不同利用率水平的延迟界。定义QoS类i的到达速率是$a_i$，所有QoS类的到达速率总和是a.

<img src="/images/Aequitas总结/image-20240313191731202.png" alt="image-20240313191731202" style="zoom:70%;" />我们的分析显示了QoS-mix在过载情况下如何影响WFQ的每个qos延迟界.定义$QoS_i-share$为QoS-mix中的第i个元素。该分析可以为给定的QoS-mix提供延迟边界，但是封闭形式的方程仅限于两个QoS级别。

假定x是QoS-mix中的$QoS_h-share$,其值是$a_h/a$.那么$QoS_l-share$的值就是1-x. $QoS_h:QoS_l$是${\phi}:1$​。如下图所示，将整个发送周期定义为一个时间单位。流量用突发参数ρ表示，是下图中黑色曲线的斜率，ρ大于1。为了稳定，存在一个空闲阶段，使得该周期内的平均负载μ小于1.0。因此，延迟界可以表示为周期的一个分数;根据定义，到达的流量可以在单个时间段内被消耗。我们把它表示为标准化延迟界。其中ρ和μ的值是关于r的标准化

<img src="/images/Aequitas总结/image-20240313192817280.png" alt="image-20240313192817280" style="zoom:67%;" />

<img src="/images/Aequitas总结/image-20240313193156494.png" alt="image-20240313193156494" style="zoom:50%;" />

x的不同子域产生不同的到达曲线，从而产生不同的延迟界表示。例如x在到达某个值之前，$QoS_h$的延迟为0。$QoS_h$排队延迟的公式推算如下。

<img src="/images/Aequitas总结/image-20240313195054253.png" alt="image-20240313195054253" style="zoom:67%;" />

图8绘制了2-QoS场景中每个QoS级别的理论最坏情况延迟。从上述表述中可以得出两个主要结论：QoS-mix会影响每个QoS的排队延迟。在QoSh一定的份额上，可以观察到优先级翻转，即QoSh中的排队延迟高于QoSl

<img src="/images/Aequitas总结/image-20240313195222847.png" alt="image-20240313195222847" style="zoom:67%;" />

















