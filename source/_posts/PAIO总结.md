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

PAIO是一个框架，为不同的应用程序实现**可移植**的I/O策略和**优化**，开发人员只需要对**原始的代码库**进行**少量修改**。主要思路是：如果我们能够在请求流经I/O堆栈的不同层时拦截和区分请求，我们就可以在不显著更改层本身的情况下执行复杂的存储策略。PAIO用到了软件定义存储的思想，构建了数据平面stages(mediate和优化跨层的I/O请求)和控制平面(根据不同的存储策略对各个步骤进行协调和微调)。用两个用例展示了PAIO的性能和适用性。

**第一种**方法将基于行业标准lsm的键值存储的第99百分位延迟提高了4倍。**第二个**是确保共享存储环境下动态的每个应用程序带宽保证

## Introduction

以数据为中心的系统已经逐渐成为I/O堆栈的一部分。通常使用存储优化( I/O scheduling, differentiation, and caching)来提高这种系统的性能。但这些优化方式和系统实现紧密耦合，并且由于缺乏全局上下文而可能相互干扰。例如，区分前台和后台I/O以减少尾部延迟等优化方法被广泛应用，然而现在在KVS中实现的方式需要对系统有深刻的理解，并且可移植性很低。类似地，部署在共享基础设施上的应用程序的优化可能由于彼此不了解而发生冲突。

> 基于基础架构的存储优化存在互相干扰、可移植性差、应用程序冲突的问题

本文认为存在一种更好的方式来实现存储优化。PAIO是一种用户级框架，通过采用软件定义存储(SDS)社区的思想，可以构建可移植且普遍适用的存储优化。**关键思想**是通过拦截和处理应用程序执行的I/O，在应用程序之外实现优化，作为数据平步骤。然后，这些优化由逻辑上集中的管理器(控制平面)控制，该管理器具有防止它们之间相互干扰所必需的全局上下文。PAIO不需要对内核进行修改，同时可以达到修改内核存储优化方法的效果。

> 关键思想是在数据平面优化每个应用程序的IO，然后在控制平面获取全局状态。

PAIO的实现面临了很多的挑战。为了在应用程序外部执行复杂的I/O优化，PAIO需要在I/O堆栈中向下传播上下文，从高级api向下传播到以较小粒度执行I/O的较低层。它通过结合上下文传播的思想来实现这一点，使应用程序级的信息能够传播到数据平面步骤，只需进行少量的代码更改，而无需修改现有的api。

> 挑战1是如何把应用程序的信息传播到数据平面

PAIO要求设计新的抽象，允许在用户空间I/O层之间区分和协调I/O请求，同时增强了存储优化的可移植性和易部署性。PAIO主要有四个抽象。`enforcement object`是一个可编程组件，它将单个用户定义的策略(例如速率限制或调度)应用于传入的I/O请求。PAIO使用`context objects`来描述和区分请求，并通过`channels`连接I/O请求、`enforcement object`和`context objects`。为了确保独立存储优化之间的协调(例如，公平性，优先级)，具有全局可见性的控制平面通过使用`rules`对`enforcement object`进行微调

```
用户空间I/O层是什么东西，具体有哪些
```

<img src="/images/PAIO总结/image-20240315163546597.png" alt="image-20240315163546597" style="zoom:67%;" />

> PAIO主要有四个抽象：enforcement object、context objects、channels、rules

使用上述的新特性和抽象，系统设计人员可以开发定制SDS数据平面步骤。为了演示这一点，我们用下面两个用例验证PAIO。**第一**，在RocksDB中添加了一个步骤，然后演示如何通过编排前台和后台任务来防止延迟峰值。结果表明，与基线RocksDB相比，在不同的工作负载和测试场景(例如，不同的存储设备，有和没有I/O带宽限制)下，启用PAIO的RocksDB将第99百分位延迟提高了4，并且与SILK相比实现了相似的尾延迟性能。这就证明了I/O优化可以更简单、更易移植的实现。**第二**，我们将PAIO应用于TensorFlow，并展示了如何在ABCI超级计算机的真实共享存储场景下实现动态的每个应用程序带宽保证。结果显示使用了PAIO的TensorFlow实例带宽目标都被满足了。

```
Software-Defined Storage(SDS)思想主要有什么
```

> PAIO在两个场景下进行了验证

本文贡献如下：

- 一个用户级框架，用于构建可编程和动态适应的数据平面步骤
- 实现了两个目标：(1)减少LSM KVS中的延迟峰值;(2)在共享存储设置下实现每个应用的带宽保证
- 实验结果验证了该方法在综合和真实场景下的性能和适用性

## Motivation and Challenges

当前系统内I/O优化的问题

### 紧耦合的优化

**问题描述**：大多数I/O优化都是单一用途的，因为它们紧密集成在每个系统的核心中。实现这些优化需要对内核有了解，并熟悉内部代码。这就限制了它们的可维护性和跨系统的可移植性。例如，为了减少RocksDB的尾部延迟峰值，SILK提出了一个I/O调度器来控制前台和后台任务之间的干扰。在RocksDB中实现SILK的优化需要修改很多核心模块的代码。并且SILK也不容易实现在别的KVS(LevelDB,pebble)中。

**解决方案**：解耦优化。I/O优化应该从系统的内部逻辑中分离出来，并转移到专用层，从而在不同的场景中变得普遍适用和可移植。

**挑战**：刚性接口。解耦优化是有代价的，因为没有办法使用系统特定优化中存在的粒度和内部应用程序知识。具体来说，传统的I/O堆栈模型需要通过难以扩展的严格接口进行通信，从而丢弃了可用于对不同粒度级别的请求进行分类和区分的信息。例如，图1是一个由应用程序、KVS和POSIX兼容的文件系统组成的I/O堆栈。从KVS提交的POSIX操作来自不同的工作流，包括前台流(a)和后台流，比如flush(b)和compactions(c)。文件系统只能获取请求的大小和类型，所以无法推断该请求的起源。将SILK I/O调度器实现在文件系统和KVS之间移植性较高。然而，它将是无效的，因为它不能区分前台和后台操作。

![image-20240315232733807](/images/PAIO总结/image-20240315232733807.png)

```
flush和compactions操作是什么样的。前台流和后台流举例子
为什么将SILK部署在文件系统之上就会是无效的了
```

> 每一层之间的接口调用都是严格的，上下层之间的信息不互通，所以调度的能力有限

**解决方案**：信息传播。application级别的信息必须在各个层之间传播，以确保解耦优化能够提供与系统特定优化相同级别的控制和性能。

**挑战**：内核层。如果在内核中实现SILK(文件系统、块层)可以增加它适配的KVS个数，但是也有几个缺点。首先，为了将application的信息传播到每一个层，它需要破坏用户到内核层(即POSIX)和内核内部接口，从而降低可移植性和兼容性。并且内核开发更难。最后，这些优化在绕过内核的存储栈中是无效的。

```
POSIX是什么
```

**解决方案**：用户层实现。I/O优化应该只在用户层实现，可以跨不同系统和场景，并且需要简化信息在各层之间的传播

### partial visibility(局部网格可见)

**问题描述**：单独实现的优化会忽略来自其他系统对相同存储资源的竞争。在共享基础设施(云计算、HPC)中，这个问题会导致冲突、I/O竞争以及应用程序和存储后端的性能变化。

**解决方案**：全局控制。优化应该了解周围的环境并协调操作，以确保对I/O工作流和共享资源进行全面控制。

> 就两个问题。
>
> 第一个问题是如何解耦，增加可移植性，适用性。以及解耦之后信息如何传递
>
> 第二个问题是信息如何在不同的应用程序之间共享

## PAIO in a Nutshell

PAIO是一个框架，使系统设计人员能够构建定制的SDS数据平面步骤。在数据平面步骤中使用的PAIO服务于给定user-level层的workflows，支持对请求进行分类和区分，并根据用户定义的存储策略实施不同的存储机制。此类策略的示例可以简单到限制贪婪租户的速率以实现资源公平，也可以复杂到协调具有不同优先级的工作流以确保持续的尾部延迟。PAIO的设计基于五个核心原则。

#### 普遍适用性

为了确保不同I/O层之间的适用性，PAIO的步骤从内部系统逻辑中分离出来。

#### 可编程构件

PAIO遵循解耦设计，将I/O机制与管理I/O的策略分开，并提供抽象，用于构建新的存储优化以应用于请求。

#### 细粒度的I/O控制

PAIO对不同粒度级别的I/O请求进行分类、区分和强制执行。支持在I/O堆栈上应用一组广泛的策略。

#### 步骤协调

为了确保步骤之间能够协调地获取资源，PAIO公开了一个控制接口，使控制平面能够动态地使每个步骤适应新的策略和工作负载变化

#### 低修改性

在I/O层上使用PAIO只需要很少的修改

### PAIO中的抽象

#### Enforcement object

Enforcement object是一种自包含的、单一用途的机制，它对传入的I/O请求应用自定义I/O逻辑。这种机制的例子包括：性能控制和资源管理(令牌桶和缓存)，数据转换(压缩和加密)，数据管理(数据预取、分层)。该抽象为系统设计人员提供了开发新机制(专门用于特定存储策略)的灵活性和可扩展性。

#### Channel

Channel是请求流通过的抽象。每一个channel有一个或者多个Enforcement object(对同一组请求应用不同的机制)

以及将请求映射到要enforced的相应Enforcement object的differentiation rule 

#### Context object

Context object包括描述请求特征的原信息。它包括一组元素(分类器)l例如流ID(例如thread-ID)，请求类型(例如read, open, put, get)，请求大小和请求context.请求context用于描述请求的附加信息，比如确定其起源、背景等。对于每一个请求，PAIO会生成相关的Context object用于在各自的I/O机制上对请求进行分类、区分和强制执行

#### Rule

在PAIO中，一条rule表示控制数据平面步骤状态的操作。rules由控制平面提交，分为三种类型：housekeeping rules管理内部步骤组织，differentiation rules分类和区分I/O请求，enforcement rules 根据工作负载变化调整enforcement object。

###  High-level Architecture

图二展示了PAIO的高层架构。它遵循一种解耦设计，将在外部控制平面实现的策略与在数据平面阶段实现的执行策略的机制分离开来。PAIO的目标是用户级的I/O层。步骤嵌入在层中，拦截所有的I/O请求然后强制执行用户定义的规则。为了达到这个目标，PAIO由四个主要部分组成。

![image-20240317195418962](/images/PAIO总结/image-20240317195418962.png)

#### Stage interface

应用程序通过stage接口进入stage，该接口在提交到下一个I/O层(文件系统)之前将所有请求路由到PAIO。对每一个请求，都会生成一个具有相应I/O分类器的Context object。

#### Differentiation moudle

区分模块根据请求的Context object对请求进行分类和区分。为了确保用细粒度来区分请求，我们使仅对层本身可访问的应用程序级信息能够传播到PAIO，从而扩大了可以执行的策略集。

#### Enforcement module

该模块负责在请求之间执行真正的I/O策略。它由channel和enforcement object 组成。对于每个请求，模块选择应该处理它的channel和enforcement object 。执行后，请求返回到原始数据路径并提交到下一个I/O层。(文件系统)

#### Control interface

PAIo提供了Control interface，允许控制平面(1)通过创建channels、enforcement objects和differentiation rules,来编排stage的生命周期，(2)通过持续监控和微调stage，确保所有政策得到满足。控制平面提供全局可视化，全面地控制所有的stages。暴露此接口允许由现有控制平面管理stages。

### A Day in the Life of a Request

PAIO如何服务一个工作流的。考虑图三所示的I/O堆栈，它由一个应用程序、RocksDB、一个PAIO阶段和一个posix兼容的文件系统组成。并且包含以下规则：限制RocksDB的flush操作速率为X MiB/s。RocksDB的背景流生成flush和compactions任务，在提交给文件系统之前会被转换成多种POSIX请求。flush被转换成write，conmpactions被转换成read和write。

![image-20240317202819785](/images/PAIO总结/image-20240317202819785.png)

开始阶段，RocksDB初始化PAIO stage，连接到已经部署的控制平面。控制平面提交 housekeeping rules，以创建一个channel和一个enforcement object，对X MiB的请求进行速率限制(白1)。它还提交区分规则(白2)，以确定stage应该处理哪些请求(基于flush的write)。

在执行阶段，RocksDB传播创建了给定操作的context(黑0)，并将所有写操作重定向到PAIO(黑1)。通过上一步可以确保旨在PAIO上强制执行写操作，通过使用 黑0 可以将带flush标记的写操作与其他可以由压缩作业触发的写操作区分开来。接下来stage选择要使用的channel，将请求入队，然后选择服务请求的enforcement object(将请求限制在XMiB/s)。执行请求之后，原写操作提交给文件系统。

控制平面会持续监事和微调数据平面stage。它定期从stage收集为该请求提供服务的吞吐量。根据该测量值，控制平面调整enforcement object来保证flush操作流的速度是X MiB/s，使用新的配置生成enforcement rules。

## I/O Differentiation

PAIO的区分模块提供了在不同粒度级别(即每个工作流、请求类型和请求上下文)对请求进行分类和区分的方法。通过三个步骤区分请求

#### Startup Time

开始的时候，用户定义如何区分请求以及谁应该处理每个请求。首先，它通过指定应该使用哪些I/O分类器来区分请求来定义区分的粒度。例如，为了区分每个workflow，PAIO只考虑Context’s workflow id分类器；而为了根据上下文和类型区分请求，它同时使用*request context*和*request type*分类器。其次，用户为每个通道设置特定的I/O分类器，以确定给定通道接收的请求集。表1提供了样例，channel1只接收来自flow1的流；channel2只处理background tasks生成的read请求；channel3接收来自flow5的压缩写请求。要生成将请求映射到通道的唯一标识符，可以将分类器连接到字符串或散列到固定大小的令牌中。这个流程可以被控制平面设置，或者在stage创建的时候被配置。

![image-20240318092742856](/images/PAIO总结/image-20240318092742856.png)

#### Execution time

第二阶段区分提交到stage的I/O请求，并将它们路由到要执行的各自通道。由两个步骤组成

==Channel selection==

对每一个请求(包含Context object)，PAIO选择必须要服务它的channel。PAIO验证Context的I/O分类器，并将请求映射到要执行的相应channel。这种映射区分过程根据第一阶段的描述完成。

==Enforcement object selection==

每一个channel可以包括多个enforcement objects，与channel选择类似，PAIO需要选择合适的object。对于每个请求，channel验证Context的I/O分类器并将请求映射到相应的enforcement object，然后enforcement object将使用它的I/O机制。

#### Context propagation

一些I/O分类器(比如workflow id, request type and size)可以通过观察原始I/O请求访问。但是应用层的信息只能传递给提交I/O请求的层。operation context是这类信息的一个例子(如图1)，它允许确定给定请求的来源或上下文(它是前台流还是背景流，flush还是compaction或者其他的)。

因此，PAIO支持将附加信息从目标层传播到stage。利用了context propagation的技术，该技术使系统能够沿着执行路径传播context，并应用他们来确保对请求的细粒度控制。为了实现这一点，系统对目标层(该层可以访问信息)的数据路径进行测量，并通过进程的地址空间、共享内存或线程局部变量使其对stage可用。这些信息都在创建Context object时被request context 分类器包含。如果不使用此方法传播context,则需要在找到信息的位置和将信息提交到stage之间更改所有核心模块和函数。

```
目标层是什么？
```

例如图3的I/O栈，为了确定RocksDB后台工作流提交的POSIX操作的来源，系统设计人员测量了RocksDB负责管理flush或compaction作业(黑0)的关键路径，以捕获它们的上下文。接下来信息被传播到stage interface，在该接口中使用所有的I/O分类器(包括请求上下文)创建Context object，并将其提交到stage。

> 这里面是在RocksDB中捕获上下文？

请注意，此步骤是可选的，因为对于不需要强制执行附加信息的策略，可以跳过此步骤

## I/O Enforcement

Enforcemnet 模块为开发用于请求的I/O机制提供构建块。它由几个通道组成，每个通道包含一个或多个enforcement object。

```
为什么一个channel有多个enforcement object？
```

如图3所示，请求被移动到被选择的channel然后放置到SQ(黑3)。对于每个出队的请求，PAIO选择合适的enforcement object(黑4)然后应用其I/O机制(黑5)。相关的I/O机制有令牌桶、缓存、加密方案等等。由于有几种机制可以改变原始请求的状态，例如数据转换(例如，加密、压缩)，在此阶段，enforcement object生成一个Result，该Result封装了请求的更新版本，包括其内容和大小。然后将Result对象返回给stage接口，stage接口对其进行解组、检查并将其路由到原始数据路径(黑6)。这个流程之后，PAIO确保该请求符合具体政策的目标。

#### Optimizations

根据所采用的的策略和机制，PAIO可以仅使用它们的I/O分类器强制执行请求。虽然数据转换直接适用于请求内容，但性能驱动机制(如令牌桶和调度器)只需要强制执行特定的请求元数据(例如，类型、大小、优先级、存储路径)。为了防止增加系统执行的开销，PAIO允许仅在必要时将请求的内容复制到stage的执行路径中.

## PAIO Interfaces and Usage

现在我们详细介绍PAIO如何与I/O层和控制平面交互，如何在用户级层中集成PAIO，以及如何构建enforcement objects.

### Interfaces

#### Stage interfaces

PAIO提供了一个application的可编程接口，用于在I/O层和PAIO的内部机制中建立联系。如表2描述的，它提供两个功能：paio_init初始化一个stage，连接控制平面和stage内部的管理，同时定义workflows应该被如何处理，enforce拦截来自层的请求，并根据相关的Context object将它们路由到Stage。enforce 请求之后，Stage输出Result，然后层恢复原始路径。

![image-20240318143600455](/images/PAIO总结/image-20240318143600455.png)

#### Control interface

Stage和控制平面的交互主要由五个调用支持，如图二所示。stage_info会返回stage相关的信息（包括stage identifier和process identifier (PID)）。基于rule的调用用于管理和调优数据平面阶段。Housekeeping rules (hsk_rule) 管理stage的生命周期(例如，创建channel和and enforcement objects)。differentiation rules (dif_rule) 映射请求到channels和enforcement objects。enforcement rules (enf_rule)根据工作负载和策略变化动态调整给定enforcement object(id)的内部状态(s)。控制平面使用collect调用监视stages，它收集所有工作流的关键性能指标(例如，IOPS，带宽)，并可用于调整数据平面stage.

该接口允许控制平面定义PAIO stage如何处理I/O请求。然而，与数据平面阶段的可靠性相关的问题，以及冲突策略的解决是控制平面的责任[38]，因此与本文是正交的

### Integrating PAIO in User-level Layers

将I/O层移植到使用PAIO阶段可能需要几个步骤。

#### Using PAIO with context propagation

为了将stage集中到层中，需要：

1. 在目标层创建stage，使用paio_init
2. 检测关键数据路径，其中层级信息是可访问的，并在Context object创建时将其传播到Stage。这可能需要创建额外的数据结构
3. 创建将与请求一起提交到Stage的Context object。包括workflow id, request type和size，并且传播信息
4. 对在提交给下一层之前需要在Stage强制执行的I/O操作添加enforce调用。比如，为了enforce给定层的POSIX read 操作，所有read都需要先被提交到PAIO在提交到文件系统
5. 通过检查从enforce返回的Result对象来验证请求是否成功enforced，并恢复执行路径

#### Using PAIO transparently

如果不需要context 传播，PAIO stages可以在I/O层(应用层和文件系统)中透明的使用。PAIO公开了面向层的接口(如POSIX)，并使用LD_PRELOAD将顶层的原始接口调用(如应用程序调用的读写calls)替换为在提交给底层之前首先提交给PAIO的接口调用(如文件系统)。每个支持的调用定义了创建Context object，将请求提交到stage，验证Result和调用原始I/O的逻辑。这使得层可以使用PAIO而无需更改任何代码行。

### Building Enforcement Objects

PAIO使用简单的API创建enforcement objects。如表2所示

- obj_init：创建具有初始状态的enforcement object，包括其类型和初始配置。
- obj_config：提供调优旋钮，以使用新状态更新enforcement object的内部设置。这使控制平面能够动态地使其适应工作负载变化和新策略。
- obj_enf：实现应用于请求的实际I/O逻辑。在应用完它的逻辑之后会返回Result，其中包含请求(r)的更新版本。它还接收一个Context object(ctx)，该对象用于对I/O请求采取不同的操作

默认情况下，PAIO保持目标系统的操作逻辑(例如排序，错误处理等)，因为提交给PAIO的enforcement object和操作都遵循同步模型。虽然可以开发成异步的，但是是要保证正确性和容错性。

```
这里的同步和异步是什么意思
```

## Implementation

用了9K行C++代码实现。PAIO针对用户级别的层，允许构建新的stage实现和简单的集成，只需要更改少量代码

### Enforcement objects

实施了两种enforcement object。Noop实现了一种传递机制，该机制将请求的内容复制到Result对象，而不需要进行额外的数据处理。Dynamic rate limiter (DRL) 实现了一个令牌桶来控制I/O工作流的速率和突发性。该令牌桶设置了最大token容量(size)和补充桶的周期(fill period)。桶处理请求的速率以令牌/s表示。调用obj_init的时候，桶的这两个变量会被初始化。调用obj_config的时候，rate(r)会根据介于r和fill period之间的函数改变size的大小。对于每一个请求，obj_enf验证context's size分类器然后计算消耗的token数量。token不够的话，请求就等待。为了演示PAIO的I/O机制的可移植性和可维护性，我们在由不同层和目标组成的两个用例上应用了DRL对象。

### I/O cost

我们认为请求的成本是恒定的，例如，读或写请求的每个字节代表一个令牌。尽管成本取决于几个因素(例如，工作负载、类型、缓存命中)，但我们不断校准令牌桶，使其速率收敛于策略目标。实验结果显示该方法在场景中很适用，因为桶的速率在与控制平面的少量交互中收敛。确定I/O成本是后续工作。

### Statistics, communication, and differentiation

PAIO在Channel中实现每个工作流统计计数器，以记录截获请求的带宽、操作数量和收集周期之间的平均吞吐量。控制平面与stages的通信是通过UNIX Domain Sockets实现的。为了创建唯一标识符把请求映射到channels和enforcement objects，使用了MurmurHash3哈希方案，该方案将分类器哈希到固定大小的令牌中。

### Context propagation

为了在每个层中传播信息，使用了共享map，索引是workflow identifier(例如，thread-id)，存储正在提交的请求的Context

### Transparently intercepting I/O calls

PAIO使用LD_PRELOAD来拦截POSIX调用，并将它们路由到Stage或内核。它支持read和write调用，包括不同的变体(例如，pread，pwrite64)。我们发现，支持这组调用足以执行面向数据的策略。我们将其他调用和接口(例如，KVS，对象存储)的支持推迟到未来的工作中。

### Control plane

我们使用3.6K行代码构建了一个简单但功能齐全的控制平面，用于为本文的两个用例执行策略。策略以控制算法的形式实现。为了校准enforcement objects，除了Stage统计数据外，它还从/proc文件系统收集目标层生成的I/O指标。具体来说，它检查读字节和写字节I/O计数器，这些计数器表示块层读/写出去/进来的字节数，并将它们与阶段统计数据进行比较，以收敛到目标性能目标

## Use Cases and Control Algorithms

我们现在给出两个用例，展示了PAIO对不同应用程序和性能目标的适用性

### Tail Latency Control in Key-Value Stores

LSM KVSs(例如RocksDB)使用前台流来参加客户端请求，这些请求以FIFO顺序排队和服务。后台流服务于内部操作，即冲洗和压缩。Flush顺序地写入树的第一层(L0)，只有当有足够的空间时才继续执行。压缩保存在FIFO队列中,等待专用线程池执行。除了低级别压缩(L0 to L1)，这些都可以并行进行。然而，它们的一个常见问题是I/O工作流之间的干扰，从而会导致客户端请求产生延迟峰值。当由于L0 to L1压缩和刷新缓慢或暂停而无法进行刷新时，会出现延迟峰值。

```
RocksDB具体的工作流程是什么样子的？ foreground flows和background flows的区别
```



#### SILK

SILK是一个基于RocksDB的KVS，他通过使用IO调度器来防止这种情况。当客户端负载较低时，为内部操作分配带宽(优先刷新和低级别压缩，因为它们影响客户端延迟)，用低级别压缩抢占高级别压缩。通过下面的控制算法实现，由于这些KVSs被嵌入，KVS I/O带宽被限定在给定的速率(KVSB)。它监控客户端额带宽(Fg)，并将剩余带宽(leftB)分配给内部操作(IB)，IB = KVSB−Fg.SILK使用RocksDB的速率限制器限制IB。Flushes和L0 to L1的压缩具有高优先级，并提供最小的I/O带宽(minB)。高级别压缩优先级较低，并且可以被随时停止。为所有的压缩都共享同一个线程池，所以有可能在某个时候，所有的线程都在处理高级的压缩。因此，SILK会抢占其中一个来执行低级压缩。

SILK改变了很多RocksDB的内部操作。此外，将这些优化移植到其他同样从中受益的KVS，如LevelDB[21]和pebble[47]，需要深入的系统知识和大量的重新实现工作。

#### PAIO

我们发现可以通过编排I/O workflows来实现这些优化，而不是修改RocksDB引擎。因此我们应用SILK的设计原则如下：PAIO数据平面Stage提供I/O机制，用于对后台流进行优先级排序和速率限制，而控制平面重新实现I/O调度算法来编排Stage。

Stage会拦截所有RocksDB的 workflows。我们将与文件系统交互的每个RocksDB线程视为一个工作流。Channel通过workflow id进行分类。我们使用RocksDB来传播创建给定操作时的context，即刷新(flush)或压缩(例如，compaction_L0_L1)。监控前台流以收集客户端带宽(Fg)。后台流被路由到由DRL对象组成的通道。Fiushed流单独设置通道。由于具有不同优先级(高和低)的压缩可以流经同一通道，因此每个通道包含两个以不同速率配置的DRL对象。通过request context分类器区分enforcement object，并使用第5节中描述的优化来强制执行请求。PAIO还收集刷新(Fl)、低级别压缩(L0)和高级别压缩(LN)的带宽。

控制平面实现了SILK调度算法的控制部分(如算法1所示)。它使用一个反馈控制回路执行以下步骤。1. 收集stage的数据，计算盘的剩余带宽(leftB)并分配到内部操作。并给后台流设置了一个最低带宽。并根据优先级分配leftB。如果两个高优先级的任务都正在执行，它会为它们分配相同份额的leftB，同时确保高级压缩保持流动(minB)，防止低级压缩在队列中阻塞。如果只有一个高优先级的任务正在执行，则将left tb分配给它，将minB分配给其他任务。如果没有高优先级的任务正在执行，则将剩余的tb预留给低优先级的任务。然后，它生成并提交enf_rules，以调整每个enforcement object的比率。对于低优先级的压缩，所有DRL对象均分BLN。由于高优先级压缩是顺序执行的，它将BL0分配给各自的对象。比率BFl分配给负责flushes的对象。

![image-20240318214126023](/images/PAIO总结/image-20240318214126023.png)

#### 集成到RocksDB

在RocksDB中集成PAIO只需要添加85个LoC(表3)

![image-20240318215234069](/images/PAIO总结/image-20240318215234069.png)

1. 初始化PAIO stage并创建额外的结构来标识每个工作流正在执行的任务(10LoC)
2. 检测RocksDB的内部线程池，用于标识运行flush和compaction作业的工作流(17LoC)。为了区分压缩操作的高低优先级，我们检测了创建压缩操作的代码。对于每个job，我们验证其级别并使用工作流将要执行的任务(compaction_L0_L1)更新结构(30LoC)。
3. 根据 workflow id，request type，context和size I/O分类器创建Context.(7LoC)
4. 提交所有读和写calls到Stage(17LoC)
5. Verify the Result of the enforcement(4LoC)

### Per-Application Bandwidth Control

ABCI超级计算机是在人工智能和高性能计算工作负载融合的基础上设计的。上面经常运行tensorFlow框架。为了执行TensorFlow作业，用户可以保留一个完整的节点或它的一小部分(即，作业并发执行)。通过Linux的cgroups将节点划分为资源隔离实例。每个实例对CPU内核、内存空间、GPU和本地存储配额具有独占性。但是本地的磁盘带宽还是共享的，每个作业之间会竞争带宽，导致I/O干扰和性能变化。即使块I/O调度器是公平的，也会为所有实例提供相同的服务级别，从而防止分配不同的优先级。

使用cgroups的块I/O控制器(blkio)允许对每个实例的读写操作进行静态速率限制。但是在ABCI中，一旦速率被设置，就无法动态更改，因为它需要停止作业、调整所有组的速率并重新启动作业，就总体执行时间而言，这是非常昂贵的。所以当一个job停止的时候，就会有带宽浪费。

#### PAIO

为了解决上述问题，我们使用一个PAIO Stage来在每个实例中实现动态速率限制工作流的机制，同时控制平面实现了比例共享算法，确保所有实例都符合各自的策略。

我们的用例侧重于模型训练阶段，其中每个实例运行一个tensorflow作业，该作业使用单个工作流从文件系统读取数据集文件。TensorFlow的read 请求被调度到Stage,其中包含一个带有DRL enforcement object的Channel。通过5中描述的优化来强制执行请求。

![image-20240318222150329](/images/PAIO总结/image-20240318222150329.png)

控制平面使用一个max-min公平算法来保障每个应用的带宽（如算法2），该算法常用于资源公平策略。总体可用磁盘带宽(MaxB)和每个应用程序的带宽需求(demand)由系统管理员或负责管理不同作业实例资源的机制预先定义。该算法也使用了反馈路径。1. 控制平面从每个活动实例的阶段收集统计信息(1)，以及每个TensorFlow作业生成的带宽(在/proc处收集)。2. 计算每个活动实例的rate(3-10)。如果一个实例的需求低于其公平份额，控制平面将分配其需求。如果实例的需求小于其公平份额，则控制平面分配其需求(4-5)，否则分配公平份额(7)。然后将剩余带宽在实例中分配(9-10)。然后，它在Ii和ratei的函数中校准每个实例的速率，生成要提交到每个阶段的最终规则(11)。然后等待下一个周期被调用。

### 集成到tensorFlow

如表3所示，集成到TensorFlow不需要代码改变。使用LD_PRELOAD来拦截TensorFlow的读写请求，然后重定向到PAIO。所有支持的调用都实现了执行请求所需的逻辑。

## Evaluation

我们的评估旨在证明PAIO的性能，以及它在不同情况下执行政策的能力和可行性。结果显示

- 通道数越多，吞吐量越多，延迟越低
- 它可用于在具有不同要求的不同 I/O 层上实施策略
- 通过该数据平面 Stage传播应用层信息，PAIO 在尾部延迟方面比RocksDB高出4倍，同时支持与SILK类似的控制和性能
- 当不需要内部系统知识时，PAIO 可以在不更改应用程序的情况下实施策略。通过具有全局可见性，它可以始终提供每个应用程序的带宽保证，并与静态速率限制方法相比，缩短了总体执行时间。

==实验设置==

<img src="./images/PAIO总结/image-20240319000105869.png" alt="image-20240319000105869" style="zoom:67%;" />

### PAIO Performance and Scalability

我们开发了一个基准测试，用于模拟向 PAIO  Stage提交请求的应用程序。它通过实例的强制调用在闭环中生成和提交多线程请求，在不同数量的客户端（例如工作流）和请求大小下。请求大小和客户端线程数分别介于 0 – 128KiB 和 1 – 128 之间。每个客户端提交100M的请求。PAIO  Stage配置了不同数量的Channel（与客户端线程的数量相匹配），每个通道都包含一个 Noop enforcement object，该对象将请求的缓冲区复制到result object。所有报告的结果都是至少 10 次运行的平均值，标准偏差保持在 5% 以下。

#### IOPS and Bandwidth

<img src="./images/PAIO总结/image-20240319001825981.png" alt="image-20240319001825981" style="zoom:67%;" />

图 4 描述了相对于单个通道的累积 IOPS 比率。0B代表context-only请求，如第5节所示。标有 ∗ 和 + 的结果分别在配置 A 和 B 下进行。

对于配置A，在 0B∗ 请求大小下，单个 PAIO 通道的平均吞吐量为 3.05 MOps/s，延迟为 327 ns。由于工作负载受 CPU 限制，因此性能不会线性扩展，因为客户端线程会争用处理时间。在128 channel，实现了97.4MOps/s的累积吞吐量性能提升了31倍。随着请求大小的增加，PAIO 处理的总字节数也会增加。当配置了128个Channel，可以以384GB/s的速率处理 128KB∗大小的请求。对于单channel来说，PAIO处理 1KB∗ 2.1GBps和128KB* 11.7GBps。对于配置B来说，由于是新的内核版本，所以PAIO的吞吐量更高。

#### Profiling

我们测量了主执行路径中出现的每个 PAIO 操作的执行时间。根据硬件配置，上下文对象创建需要 17 到 19 ns，而通道和强制对象选择需要 85 到 89 ns 才能完成（每个）。配置 0B 和 128KiB 请求大小时，obj_enf持续时间范围在 20 ns 到 8.45 μs 之间。

```
obj_enf的具体操作
```

#### summary

结果表明，PAIO具有较低的开销，因为它是作为用户空间库提供的，不需要昂贵的上下文切换操作。我们预计开销的主要来源将始终取决于对请求应用的 enforcement object 的类型。对于本文用例中使用的 enforcement object（§9.2 – §9.3），我们没有观察到明显的性能下降。

### Tail Latency Control in Key-Value Stores

现在我们将演示PAIO如何在几种工作负载下实现尾部延迟控制。我们比较了RocksDB,Auto-tuned SILK, PAIO

#### System configuration

实验在硬件配置B下使用NVMe SSD。所有系统都按如下方式进行调优。memtable-size的值设置为128MB。使用8个线程服务客户端操作，8个背景线程，1个分给flush，7个分给compactions。内部操作的最低带宽阈值设置为10MBps。为了简化结果，关闭了压缩和提交日志的记录。所有实验均采用db bench基准测试。在SILK测试中，我们将内存使用限制为1GB, I/O带宽限制为200MiB/s。

#### Workloads

我们专注于由突发客户端组成的工作负载，以便更好地模拟生产中的现有服务。客户端请求通过波峰和波谷的组合在一个闭环中发出。初始谷持续300秒，以5kops/s的速度提交操作，并用于执行KVS内部积压。峰值持续100秒，发送速率是20kops/s，后面紧接10秒的5kops谷值。所有数据存储都预加载了100M键值对，使用统一的键分布、8B个键和1024B个值。

我们使用了读写不同比例的三种工作负载。Mixture负载示常用的YCSB工作负载(工作负载 A)，并提供与Nutanix生产工作负载相似的比率。Read-heavy提供了与Facebook类似的操作比率。为了展示一个全面的测试平台，还包含一个write-heavy工作负载。对于每个系统，工作负载在1小时内执行三次，使用统一的密钥分布。只展示前20min的内容。图5-9描述了所有系统和工作负载的吞吐量和第99百分位延迟。理论客户端的负载以红色虚线表示。平均吞吐量显示为虚线。

#### Mixture workload 

![image-20240319094807796](/images/PAIO总结/image-20240319094807796.png)

如图5所示，由于加载阶段的累积积压，所有系统中实现的吞吐量与理论客户机负载不匹配。RocksDB由于持续的flush和低级别的压缩操作，尾部延迟较高。Auto-tuned的吞吐量下降。SILK会周期性经历吞吐量下降。PAIO能提供和SILK类似的平均吞吐量。

#### Read-heavy workload 图6

![image-20240319095452015](/images/PAIO总结/image-20240319095452015.png)

吞吐量方面，所有系统执行相同。在400秒之后，SILK暂停高级别压缩，并实现12毫秒之间的尾部延迟。通过抢占高级压缩并通过与flush相同的线程池为低级压缩提供服务，它可以确保高优先级的任务很少停滞

#### Write-heavy workload 图7

![image-20240319100120781](/images/PAIO总结/image-20240319100120781.png)

写密集型工作负载会产生大量的后台任务积压，导致RocksDB出现高延迟峰值。Auto-tuned限制了所有后台写入，减少了延迟峰值，但在几个时间段内仍然超过5毫秒。SILK暂停高级别压缩，值服务高优先级任务。在PAIO中，由于flush发生得更频繁，控制平面更大幅度地降低了高级压缩的速度，这导致低级压缩在压缩队列中暂时停止，等待执行。PAIO和SILK之间的吞吐量差异是由后者抢占高级压缩来证明的，正如在read-heavy的工作负载中所描述的那样.

#### Mixture workload without rate limiting 图8，9

![image-20240319101127469](/images/PAIO总结/image-20240319101127469.png)

该实验中，KVS可以获得设备的所有带宽。设置KVSB参数值接近设备的限制。Auto-tuned的结果和图5相似。

图8中，由于累积的积压，所有系统的吞吐量性能都很差，平均为7.46 kops/s (RocksDB)， 7.52 kops/s (SILK)和8.88 kops/s (PAIO)。在加载阶段，直到完成累积的积压(0.400秒)，RocksDB经历了长时间的高尾部延迟，峰值为111毫秒。之后，它观察到由于持续刷新和低级别压缩而产生的延迟峰值，其值在15 - 60毫秒之间。SILK和PAIO表现出更持久的延迟性能，在整个观察期内从未超过25毫秒。吞吐量方面，两个系统都观察到由于累积的积压而周期性下降。PAIO恢复得更快。因为PAIO不能抢占压缩，所以它为低优先级的压缩保留了更多的带宽(比SILK多)，从而确保高优先级的任务不必等待执行。因此，PAIO采用主动方法将带宽分配给压缩，而SILK采用被动方法。

```
累积积压为什么会导致吞吐量下降
```

图9显示了在NVMe中的结果。所有系统的吞吐量表现都很好。RocksDB遵循与理论客户端负载相似的性能曲线。原因有两个，首先，它在初始谷期间完成所有累积的积压(以高尾部延迟为代价)，这在剩余执行中得到了积极的反映(即没有观察到明显的性能损失)。其次，由于NVMe设备比SSD设备具有更高的吞吐量性能和并行性，因此RocksDB实现了更持久的性能。在初始谷之后，RocksDB观察到由于频繁刷新和低级别压缩，延迟峰值在7 - 15毫秒之间。与前面的结果类似，两个系统都会经历周期性的吞吐量下降。

#### Summary

我们证明，通过微小的代码更改，PAIO在尾部延迟方面的性能比RocksDB高出最多4倍，并且能够实现与SILK相似的控制和性能，而后者需要对原始代码库进行深入的重构。

### Per-Application Bandwidth Control

现在我们将演示在共享存储场景下，PAIO如何确保每个应用程序的带宽保证。我们的设置是由ABCI超级计算机的需求驱动的

#### System configuration

使用硬件配置A。每个实例使用专用GPU和数据集运行，其内存限制为32GB。磁盘总带宽限制为1GiB/s。在任何时候，一个节点最多执行四个实例，在CPU、GPU和RAM方面具有相同的资源共享。每个实例执行一个TensorFlow作业，分配一个带宽策略，并执行给定数量的训练epoch。实例1到实例4的带宽保证值分别为150、200、300和350 MiB/s，并分别执行6、5、5、4个训练周期。

#### setups

实验在三种设置下进行。Baseline表示ABCI超级计算机支持的当前设置;所有实例在没有带宽保证的情况下执行。Blkio使用Blkio[2]强制带宽限制。在PAIO中，每个实例执行一个动态执行指定带宽目标的PAIO Stage。图10描述了每个设置中所有实例每隔1秒的I/O带宽。实验包括七个阶段，每个阶段标记一个实例开始或完成其执行

![image-20240319111125457](/images/PAIO总结/image-20240319111125457.png)

#### Baseline

实验在52min之内完成。一直均分

#### Blkio

95min之内完成。不能动态地提供可用的磁盘带宽，从而导致更长的执行周期

#### PAIO

56min之内完成。可以共享剩余带宽。

## 相关工作

==SDS systems==

所有系统都针对特定的I/O层，因为它们的设计与它们所应用的软件堆栈的体系结构和特性紧密耦合并受其驱动。相比之下，PAIO从特定的软件堆栈分解，使开发人员能够构建适用于不同用户级层的定制数据平面阶段，而无需对代码进行微小更改。我们通过在两个不同的I/O层(第8节)上集成PAIO来演示这一点。以前的工作也无法执行8.1中展示的策略，因为它们不提供上下文传播，从而抑制了更细粒度的请求区分(即前台任务、高优先级任务和低优先级后台任务)。此外，这些也不适合实现8.2中展示的策略，因为像[31,52,54]这样的解决方案不能在需要裸机访问资源的场景下使用，例如HPC基础设施和裸机云服务器。

==Context propagation==



==Storage QoS==

























































