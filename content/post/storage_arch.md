---
title: "从零开始开发一个单机存储引擎"
date: 2018-10-07T14:43:53+08:00
draft: false
tags: [
    "storage",
]
categories: [
    "storage",
]
---

## 1 VDL Logstore概述
如何设计存储引擎，使得读写接口的性能足够高，如何保证在机器宕机时，存储引擎能够将已存储的数据恢复到一个一致性状态。如何测试存储引擎的正确性？本文将着重介绍一下VDL系统的日志存储引擎--Logstore的架构设计与核心流程实现，及为了保证Logstore的正确性，我们做了哪些工作；为了进一步提高Logstore的读写性能，我们又做了哪些工作。希望通过这篇文章，给大家介绍一下设计和开发一个存储引擎的『前世今生』。

### 1.1 Logstore提供的功能
VDL中有两种日志形态，一种是raft日志（以下称为raft log），由raft算法产生和使用，另一种是用户形态的Log（以下称为user log），由用户产生和使用。Logstore作为VDL日志存储引擎，同时存储着VDL的raft log 和user log。Logstore在设计中，将两种Log形态组合成一个Log Entry。只是通过不同的头部信息来区分。Logstore需要同时提供两种不同形态的Log操作接口，主要有以下几类：

* 读取，根据索引信息，读取对应的Log。
* 写入，将用户产生的Log，封装成相应的user Log和Raft Log写入到Logstore中。
* 删除，删除用户不再使用的Log，以文件为粒度，从最开始位置往后删除。
* 转换，由Raft Log获取对应的user Log。
* 截断，截断一部分Log，主要是为了支持raft lib中删除未达成一致的Log的功能。

## 2.Logstore的架构设计
### 2.1系统架构
Logstore由数据文件和索引文件组成，同时Logstore还会在内存中缓存最新的一段Log Entry，用于Raft lib能够快速地从内存中读取到最近Raft log，同时用户也能够快速读取到最新存储到Logstore中的user log。Logstore的组成如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-f32b0b7fbe63ef64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* segment: 用于存储log的文件，大小固定（默认是512MB）。Segment文件从前到后代表着log的顺序，Logstore通过追加的方式不断将Log Entry写入到segment中。Logstore只追加Log Entry到最后的Segment文件中，对于整个Logstore只有最后一个segment可读可写，其他Segment文件只读。由于Segment文件大小固定，我们采用mmap函数方式对segment文件进行读写。
* index: 用于存储对应的segment中的log entry的元信息，例如：log entry在segment文件中的偏移，raft log index等。每个索引项大小固定。用于加速查找raft log和user log。
* MemCache: 缓存最后一段log entry数据，保证VDL能够从内存中读取最新的一段log entry数据。

segment由一条一条的raft log entry组成，raft log的data部分存放的是user log。每个segment文件对应一个index文件，index file由index entry组成，index 文件中的索引项纪录了对应raft log的位置和大小等信息。示意图如下所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-3c08dbe4cdc058a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




## 3. Logstore的核心流程实现
### 3.1 读数据流程
Logstore读数据分为两种情况：

Read in MemCache,MemCache的元数据记录了缓存的Log范围信息，当读取范围刚好落在MemCache内时，则Logstore直接从MemCache中读取Log并返回。
Read in Segment,当上层读取的Log范围未完全落在MemCache中时，则会从segment文件中读取。Logstore记录了每个segment的Log范围元数据信息，先通过segment范围元数据信息，定位到读取的开始segment，然后在通过索引来定位具体的文件偏移。例如，读取raft index 为10010-10019这段范围的raft log,segment范围如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-63d10fc1b202720a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


根据segment的Log范围元数据信息，我们可以知道此次读取范围开始位置和结束位置都在segment_2中，由于Raft log entry的长度是不固定的，如何定位读取开始位置和结束位置的文件偏移呢？这时候就需要用到索引项，在Logstore中每个Log entry对应的索引项大小是固定的，索引项纪录了该raft log entry在segment文件内的文件偏移。segment_2对应的index文件第一个索引项纪录的是raft index为10001的raft log entry索引项，所以需要在index文件中超找raft log index范围是：10010-10019，就非常简单了。直接读取index 文件的第10到第19范围的索引项，然后根据索引项内的文件偏移到segment上读取raft log。大概的流程如下图所示：

![image.png](https://upload-images.jianshu.io/upload_images/66307-9ba37bc712105e45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 3.2 写数据流程
raft算法要求写入的raft log必须强制落盘后，才能返回成功。通过将log entry批量异步写入segment文件，并调用sync_file_range函数强制刷盘。为了提升写入segment性能，segment文件创建时就预分配了512MB的磁盘空间，这种预分配文件空间的方式有助于提升写性能。将索引信息写入index文件是异步写完后就返回。同步写segment，异步写index的方式降低了raft log写耗时，但又不影响raft算法的正确性。因为raft算法是以segment中的数据作为参考标准的。

Logstore写入流程如下图所示：

![image.png](https://upload-images.jianshu.io/upload_images/66307-ee40bae97ee50651.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### 3.3 数据恢复流程
Logstore必须要考虑到在VDL系统异常退出时，存储的数据有可能出现不一致。例如在Logstore写数据过程中，机器突然宕机。这时候就有可能只写入了部分数据，在设计Logstore时就必须考虑到如何支持数据恢复操作，保证写入Logstore的数据的一致性。

在Logstore中，只有最后一个segment文件可能出现数据不一致的可能。因为Logstore在写满一个segment文件后，会创建一个新的segment文件。在创建新的segment文件之前，Logstore通过sync系统调用让最后的segment对应的index文件内容强制刷盘，并且最后一个segment文件写入本身就是同步写。通过这种机制保证了只有最后一个segment写入的数据存在部分写的可能。而在这之前的segment文件和index文件内容都是完整的。

有了上面的保证，数据恢复我们只需要考虑最后一个segment及其index文件中的数据是否完整。Logstore通过一个标识文件来标识系统是否正常退出，如果文件存在且里面的标记为正常退出，Logstore就走正常启动流程，否则，转入数据恢复流程，Logstore数据恢复流程，主要操作如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-1cd3af3df6dd9424.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 4.Logstore的测试
为保证Logstore的正确性，我们对Logstore对外提供的接口函数及内部调用的核心函数都做了单元测试，通过gitlab+jenkins持续集成的方式，保证每次提交都会触发脚本将所有的单元测试重新运行一次，如果新增代码或改动代码，导致单元测试失败，我们可以立刻发现。通过这种持续集成的方式，我们可以保证每次代码提交的质量。

仅仅有单元测试还是不够的，因为我们无法预测Logstore某个接口函数异常，对整个VDL系统造成什么影响。所以，我们还对Logstore进行了异常测试，通过一个自研工具FIU，对Logstore中特定的函数注入各种异常条件，测试Logstore的在异常情况下，对系统的影响。我们在Logstore相关代码中插入固定的异常代码，然后通过FIU来触发相应的异常点。这样就可以让Logstore走入指定的异常逻辑代码。异常注入测试主要分为两类：

* 增加读或写延迟，Logstore向上层提供读写raft log和user log等操作。例如，读取raft log增加3s的延迟、写入user log增加1s-3s的随机延迟。我们测试在这类异常场景下，对上层VDL会造成什么影响，结果是否跟我们的预期一致。
* 部分写问题，机器突然宕机，有可能导致Logstore部分写操作。也就是segment有可能只写入了部分数据，或者index文件只写入了部分数据。同样，我们也是在写入segment文件逻辑和index文件逻辑中增加异常点，利用FIU触发指定的异常逻辑。这样就可以测试到在Logstore出现部分写时，Logstore的数据恢复流程是否能够正常工作，是否符合预期。

有了这类异常测试，我们可以提前去模拟线上有可能出现的异常场景，并修复可能存在的未知缺陷。保证VDL上线后更加稳定、可靠。并且添加异常各类异常测试用例是一个持续的过程，伴随着VDL系统开发和演进的全过程。
## 5.Logstore的性能优化

为保证Logstore具有高性能的读写，在设计阶段就考虑到了。比如通过文件空间预分配来提升写性能，通过mmap方式读日志数据，提升读性能。在代码开发完成后，结合go pprof和火焰图来定位Logstore的性能开销较大的系统调用或代码段，并做相应优化。性能优化方面的工作，比较有意义的几点，可以分享一下：

* 批量写数据，不管是写segment还是写index文件，都是将数据先组合在一个内存空间中，然后批量写入到磁盘。减少IO调用带来的开销。
* index文件异步刷盘，在前面的设计中，我们谈到在segment rolling操作中，需要将index文件同步刷盘后，再创建新的segment文件。通过持续观察发现，每次index文件刷盘都要消耗4ms-8ms的时间。写入操作如果需要segment rolling时，这次的写入延迟额外会增加4ms-8ms。Logstore的写入就会出现抖动。经过分析，我们可以发现index文件同步刷盘所做的操作就是将index文件对应的内存脏页更新到磁盘。如果我们能够减少segment rolling操作时index文件对应的内存脏页数量。就可以缩短index刷盘的耗时。我们采用的方式是每次写index文件时，再调用sync_file_range操作异步将index文件数据刷盘，这样就可以分摊最后一次刷盘的压力。经过优化后的index文件刷盘操作耗时缩短到200us-300us。使得整个Lostore的写入耗时更加平滑。
在核心函数调用中Logstore记录相关metric信息，在Logstore上线后，通过日志收集系统，收集metric信息到influxdb，然后通过grafana展示出来。有了grafana的直观展示，我们可以监控到耗时比较长的系统调用，并做针对性地优化。目前关键的读取和写入操作都达到了预期的性能目标。

## 6.总结

本文介绍了Logstore在设计、开发、测试和性能优化等方面，我们所做的工作。希望能够给读者在设计和开发分布式存储系统时，提供一定的参考思路。在后续演进中，我们希望结合业务场景，对数据做冷热分离，进一步降低生产系统的成本。到时候有新的心得体会，我们继续给大家分享。
