---
layout: post
title: Data Protection Basic - 1
description: 数据保护系列文章。介绍什么是数据保护。数据保护的主要手段之一，备份的基本概念。
categories:
- [Chinese, Software, Hardware]
tags:
- [backup, data protection, file system, virtualization, storage, cloud]
---

>本文首发于<http://oliveryang.net>. 转载时请包含原文或者作者网站链接。

## 数据保护的那点儿事儿

数据保护就是保护数据使其免于数据损坏(Data Corruption)和数据丢失(Data Loss)的过程。
常见的数据保护方式主要有以下两大类，

- **备份(Backup)**

  备份是指为了应对数据丢失(data loss)而将计算机数据进行拷贝和归档的过程。
  根据数据保存时间和目的，广义上的备份又可以细分为备份(Backup)和归档(Archive)。
  归档存储系统，因为其数据访问热度，有时又被称为冷存储(Cold Storage)。

- **灾难恢复(Disaster Recovery)**

  灾难恢复为重要的IT基础设施和系统提供了在自然或人为灾害之后能够恢复业务连续性的技术策略和手段。
  灾难恢复是IT基础设施业务连续性
  ([Business Continuity](https://en.wikipedia.org/wiki/Business_continuity))
  方案中重要的一环。

本篇文章主要关注数据保护技术中备份的基本概念。

### 1. 备份需求

下面介绍的概念直接决定了用户如何选择潜在的数据保护方案，

- **RPO(Recovery Point Objective)**
  即目标恢复点。RPO关系到最大可容忍的数据丢失量。

- **RTO(Recovery Time Objective)**
  即目标恢复时间。RTO关系到最大可容忍的业务中断时间。此外，RTO也直接决定了恢复的性能要求。

- **Backup window**
  备份窗口指备份软件执行备份所需的时间窗口。由于备份或多或少的会对被保护的应用造成一定程度的干扰，
  备份窗口的大小直接反映了备份对业务的干扰程度。备份窗口的限制也直接决定了对备份的性能要求。

备份恢复方案的选择，不但和用户的RPO/RTO/Backup window有关，还可能和以下因素有关，

- **Data Retention Period**, 即备份数据的存放时间。
- 数据中心基础架构
- 用于灾备的预算

### 2. 备份方法

为满足用户业务不同的备份需求，一个备份方案可能会同时采用下面一种或者几种备份方法，

- **Full Backup**
  完全备份。即把源端所有的数据拷贝到备份存储上。

- **Incremental Backup**
  增量备份。即只备份上次完全备份或增量备份之后变更的数据。

- **Differential Backup**
  差异备份。即只备份上次完全备份后变更过的数据。和增量备份的差别是，增量备份可以基于上次的增量备份做。
  但差异备份必须基于上次的完全备份做。

- **Synthetic Backup**
  合成备份。即一次完全备份之后，一直做增量备份。一旦增量备份做完，就会利用原来的增量备份和完全备份去合成出一个完全备份。
  一般合成是由备份软件或者支持合成备份的备份存储来完成的。因此，这种备份方式时刻都会有一个最新的合成后的完全备份可以用于数据恢复。
  同时，由于备份历史和回滚数据的存在，可以计算出反向的增量，用户也可以恢复到任何一个备份的历史版本。
  这种方式也被叫做reverse delta备份。苹果电脑的时间机器(Time Machine)，就是此方法应用上的一个例子。

- **CDP(Continuous Data Protection)**
  连续数据保护。这种备份发生在Block层。通常会利用在block层的IO分路器(IO Spliter)，
  把主机端下发的每一个IO都复制下来。并且因为有回滚日志，数据也可以恢复到某个历史版本。
  CDP的RPO是零，因此相当于备份为**每IO**粒度，IO复制是实时同步的。

- **Near CDP or CRR(Continuous Remote Replication)**
  近似CDP或者连续远程复制。很多备份软件或存储产品也实现了基于Block层的实时异步复制。
  RPO此时并不为零，但是也维持在一个较小的时间粒度。

根据备份的实现原理，备份方法还可以分为以下几种，

- **Image Level Backup**
   镜像级别的备份可以发生在物理机，也可以是虚拟机。近些年新出现的虚拟机备份产品完全是基于虚拟机镜像级别的备份方案。

- **File Level Backup**
   文件级别的备份。传统备份大都基于文件系统之上来做文件级别的备份。通常这些备份产品都需要在主机里安装一个代理软件
   (Backup Agent)来负责拷贝文件。新的虚拟机备份产品已经完全抛弃了这种备份方式，完全基于镜像备份。

- **Snapshot**
   快照可以被广泛的用于存储和虚拟机镜像的备份。新的虚拟机备份产品大量的利用了虚拟机镜像，实现了Near CDP的分钟级别的RPO。

不同的RPO和RTO的需求可能导致不同的数据保护方案的选择。下面的表格是一个简单的总结，

| RPO     | RTO     | Possible Data Protection Solutions   |
|---------|---------|--------------------------------------|
| Zero    | Minutes | 业务的双活方案。如Stretched clusters |
| Zero    | Minutes | 业务的主备方案。如CDP + VMware SRM   |
| Minutes | Minutes | Near CDP 方案或者VM备份方案。        |
| Hours   | Minutes | 一般备份和恢复方案                   |
| Hours   | Hours   | 一般备份和恢复方案。                 |


### 3 备份数据的可用性

备份时，除了要满足用户的不同业务备份需求外，更要保证备份的数据是可靠的，可用的。
一个数据保护方案，要达到备份数据的可用性，需要做很多工作。

#### 3.1 数据一致性

备份时的数据一致性问题，是备份方案必须要解决的问题，因为恢复的最终目的是要恢复中断的业务。
主机备份时，数据的一致性状态可以是以下三种，

1. **Crash Consistency**
   崩溃一致性。备份时系统没有任何静默(Quiesce)。想象一下运行的系统突然掉电或者崩溃时的状态。
   系统被恢复到这样的状态下可能因为数据丢失而无法再次启动。 

2. **File System Consistency**
   文件系统一致性。备份时操作系统被执行静默操作。操作系统的pending data被写入到硬盘。
   此时文件系统的状态是一致的。数据恢复到这个状态，文件系统不会有数据丢失，操作系统可以被正常启动。
   但应用程序状态是不可预测的，很有可能无法启动。
 
3. **Application Consistency**
   应用一致性。备份时操作系统和应用都执行了静默操作。应用和操作系统都保证在备份前把dirty data写入到硬盘，
   并且处于短暂暂停状态保证备份数据的一致性。数据恢复到这个状态，操作系统和应用都能保证正常启动运行。

由上述描述可见，实现数据一致性必须操作系统和应用程序支持静默(Quiesce)操作。为此，Windows操作系统提供了
[VSS(Volume Shadow Copy)机制](http://dcsblog.burtongroup.com/data_center_strategies/2009/02/linux-wheres-your-vss.html)。
该机制可以允许OS和应用程序在热拷贝发生时，实现自己的静默操作。VSS在Windows操作系统是被很多关键应用
[全面支持](https://en.wikipedia.org/wiki/Shadow_Copy)的。
而Linux操作系统目前还缺乏一个从内核到应用程序统一的框架来实现类似功能。Linux的某些文件系统快照功能可以实现文件系统一致性，
做快照时静默IO。但是还是缺乏一个通知应用程序执行静默的机制。

#### 3.2 数据完整性

数据存储到介质上可能遇到数据损坏(Data Corruption)的风险。企业级的备份或数据保护存储都需要针对数据完整性
(Data Integration)做一些工作。

- 数据校验
  数据存储时使用校验码算法(CRC32 or CRC64)，保证任何数据损坏都能被校验算法检测出来。
- 数据纠错
  当数据损坏发生时，可以借助RAID 5/6，或者EC(Erasure Code)的机制去恢复受损数据。

一些企业级数据保护存储还实现了
[端到端的数据完整性](http://www.emc.com/collateral/software/white-papers/h7219-data-domain-data-invul-arch-wp.pdf),
在存储栈的各个层次上做了不同的工作。

#### 3.3 备份测试和验证

备份的数据可能因为各种原因导致无法使用。例如存在一致性问题或者数据完整性问题。
因此，备份测试和验证是保证备份数据真正在恢复时可以使用的唯一可靠手段。这篇讲
[Facebook mysql备份](http://cenalulu.github.io/mysql/how-we-do-mysql-backup-in-facebook)
的文章也涉及了facebook的自动备份验证的内容。借用里面的一句话，

> 没有进行验证的备份是无效的。

备份测试和验证没有冗余的测试环境，没有自动化的手段是不行的，因此实施难度也很大。
随着虚拟机备份产品的流行，备份测试验证变得非常容易起来。


### 4. 备份软件

数据备份操作通常有不同的操作方式，

- 手工或者定制脚本和备份工具
- 专业备份软件

当需要备份的业务较多，而且备份计划很复杂时，简单的备份脚本和工具就很难满足业务需要了。
这时通常需要借助专业的备份软件来管理备份。

一个典型的传统专业备份软件可以包含以下几部分，

- **Backup Agent**

  即备份代理或备份客户端。备份代理必须按照在需要备份的主机的操作系统之上。因此备份客户端会有很多个。

  一个有良好设计的备份代理，必然要有平台无关的通用代码层和平台相关的插件。
  通用代码层处理各个平台上和备份相关的通用逻辑。而平台相关的模块则为不同操作系统的文件系统和应用程序提供了模块化的插件。
  正是因为有了这些模块化的插件，才让文件系统和应用的数据一致性可以得倒保证。


- **Backup Server**

  即备份服务器。备份服务器通过网络服务于备份代理，可以是一个，也可以是多个，来做到性能上的负载均衡和扩展。

  备份服务器通常可以有以下功能，

  * 备份和恢复任务的管理，调度和监控
  * 备份协议加速，Deduplication(去重)，Compress(压缩)，加密等
  * 备份元数据和数据存储，索引，查询。
  * 备份服务器的管理控制
  * 外部备份存储设备的管理。

有些备份服务器软件被集成到一个存储里，可以直接存储数据。即使是这样，也需要支持外置的专业备份存储设备。
例如，EMC的Avamar备份服务器就可以存储数据在自己的存储节点上。但同时，Avamar也支持备份数据到外置的EMC DataDomain存储。
此时，Avamar的备份代理直接把数据备份到Datadomain存储上，但备份的元数据还是会写到Avamar服务器上，
便于备份服务器统一管理备份数据。

### 5. 备份存储

不论是哪种方式备份，都需要决定数据最终存放的介质，即备份存储。

通常备份存储都被归类为数据保护存储(Data Protection Storage)，本文并不做区分。
备份存储设备通常可以根据存储介质或者备份数据的管理方式去分类。

#### 5.1 按照存储介质分类

按照备份数据的目标存储介质，备份存储可以分为，

- **磁带(Tape)存储**

  历史悠久，使用最广泛的备份存储。优点是可靠，廉价。缺点是性能差，不适合随机读
  写，而且存储密度低。磁带库的机械故障和占用空间都使得磁带存储的总体拥有成本
  (TCO)变得很高。

- **磁盘(Hard Disk)存储**

  磁盘的优点是性能和高存储密度。缺点是寿命和价格都不如磁带。然而，磁盘备份存储厂商通过利用技术手段
  (RAID，CRC校验，Data Scrubing，Deduplication，Compress)一定程度上克服了磁盘备份系统的不足。
  因此，磁盘备份系统在需要性能保证的备份和恢复的业务场景有很大的竞争力。但是在长期数据归档方面，
  还很难完全取代磁带。例如，磁带做归档可以做到离线，但磁盘不能，因为需要定期加电做数据校验。

- **光(Optical)存储**

  从早期的CD和DVD，到目前的蓝光，光存储因为介质的稳定和廉价被广泛的应用在归档系统，
  即冷存储领域。据说，Facebook就是利用蓝光系统实现其冷存储系统的。

- **闪存(SSD)存储**

  闪存的性能优势很明显。而且，由于闪存不包含磁盘那样的机械部件，在寿命和抗震性上都要优于磁盘。
  目前闪存做备份存储最大的障碍是两个，价格和数据可靠性。根据工业界的预测，未来几年闪存价格马上要追上磁盘价格。
  此外，闪存长时间不加电存放数据会丢失的情况可以利用一定的数据纠错算法来弥补，
  也就是说可以通过牺牲一定的空间来换取存储数据的可靠性。因此，未来磁盘备份系统全面被全闪存备份系统取代也并非痴人说梦。

#### 5.2 按照数据管理方式分类

按照数据管理的生命周期，备份存储可以分为，

- **在线(On-line)备份存储**

  在线备份存储可以选用主存储(Primary Storage)或者次级存储(Secondary Storage)。
  主存储虽然能提供很高性能的数据备份方式(如快照)，但时因为成本和数据隔离的需求，
  拥有一份以上的多个数据拷贝是备份的基本需求。因此，专业的次级备份存储在在线备份存储里才是主流。

- **离线(Off-line)备份存储**

  离线备份因为可以提供更可靠的隔离性和更低的维护成本而成为不可取代的备份选择。
  备份存储在线就意味着误操作或者其它安全事件随时可以威胁到备份数据的安全。
  而数据备份后离线存放可以彻底保证数据的安全隔离要求。

- **异地(Off-site)备份存储**

  为了抵御不同级别的天灾或者人祸。更高级别的灾备方案可能会有不同距离的异地备份要求。
  异地备份也可以通过同步或者异步地在线复制(Replication)或者离线运输备份介质的不同方式来实现。
  很多备份存储和备份软件都支持不同RPO的远程复制功能。而离线的备份介质运输虽然很古老，
  但是还是出现了一些新的创新，例如亚马逊最近推出的
  [Snowball服务](http://docs.aws.amazon.com/AWSImportExport/latest/DG/whatissnowball.html)。

正因为备份存储的多样性，有了数据的多个拷贝，多个历史版本，还有多个物理分布，
才使得不同的灾备计划可以抵御不同级别的自然灾害，人为灾害，软硬件故障和误操作。
备份计划，方法，还有基础设施的选择也和业务的连续性要求以及灾备预算密切相关。