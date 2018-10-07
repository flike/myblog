---
title: "Braft的日志存储引擎实现分析"
date: 2018-10-07T18:43:53+08:00
draft: false 
tags: [
    "storage",
    "raft",
]
categories: [
    "storage",
    "raft",
]
---

## 1.架构设计
### 1.1 函数接口说明
日志存储引擎是用于存储raft lib产生的日志。提供的接口如下：

```
class LogStorage {
public:
    virtual ~LogStorage() {}

    // init logstorage, check consistency and integrity
    virtual int init(ConfigurationManager* configuration_manager) = 0;

    // first log index in log
    virtual int64_t first_log_index() = 0;

    // last log index in log
    virtual int64_t last_log_index() = 0;

    // get logentry by index
    virtual LogEntry* get_entry(const int64_t index) = 0;

    // get logentry's term by index
    virtual int64_t get_term(const int64_t index) = 0;

    // append entries to log
    virtual int append_entry(const LogEntry* entry) = 0;

    // append entries to log, return append success number
    virtual int append_entries(const std::vector<LogEntry*>& entries) = 0;

    // delete logs from storage's head, [first_log_index, first_index_kept) will be discarded
    virtual int truncate_prefix(const int64_t first_index_kept) = 0;

    // delete uncommitted logs from storage's tail, (last_index_kept, last_log_index] will be discarded
    virtual int truncate_suffix(const int64_t last_index_kept) = 0;

    // Drop all the existing logs and reset next log index to |next_log_index|.
    // This function is called after installing snapshot from leader
    virtual int reset(const int64_t next_log_index) = 0;

    // Create an instance of this kind of LogStorage with the parameters encoded 
    // in |uri|
    // Return the address referenced to the instance on success, NULL otherwise.
    virtual LogStorage* new_instance(const std::string& uri) const = 0;

    static LogStorage* create(const std::string& uri);
};
```
LogStorage只是一个抽象类，只定义了函数接口。具体的日志操作由SegmentLogStorage实现。
### 1.2 存储引擎的数据组织
SegmentLogStorage实现了LogStorage的全部接口。其数据组织格式如下：
![image.png](https://upload-images.jianshu.io/upload_images/66307-26d6c05e299f2316.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* segment名字为first_raft_index-last_raft_index，表示该segment的raft index范围。
* 只有最后一个segment可读写，其文件名为log_inprogress_first_raft_index，其他segment只读。
* segment文件对应的index entry，Segment文件初始化时构造出来，存储在内存中。不会持久化到磁盘。因此追加一次Log Entry只会引起一次磁盘操作。

## 2.核心流程实现
### 2.1 存储引擎的接口函数
### 2.2 存储引擎的初始化
存储引擎的初始化操作主要检查文件信息，将segment的索引信息加载到内存，为读写操作做准备。
![image.png](https://upload-images.jianshu.io/upload_images/66307-8e690f46ac9d5454.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


函数主要功能如下所述：

* init函数是SegmentLogStorage初始化的入口函数，调用load_meta函数，list_segment函数和load_segment函数。
* load_meta函数：从log_meta文件中读取从SegmentLogStorage的第一个raft index值。
* list_segment函数：建立起segment的范围信息，并将范围异常的segment文件删除。范围信息存储在一个map表中，map的key是first_raft_index,value是segment对象。
* load_segments函数：构建出每个segment对应的索引项，通过解析segement内容完成。索引项存储在一个vector中。至此，就可以根据范围信息来定位到某个raft_index对应的文件偏移。
 
### 2.3 写数据流程
写数据到存储引擎，会涉及到两个函数:

```
 // append entry to log
    int append_entry(const LogEntry* entry);
 // append entries to log, return success append number
   int append_entries(const std::vector<LogEntry*>& entries);
```
append_entry表示追加单条Log Entry到日志存储引擎，append_entries用于同时追加多条Log Entry到日志存储引擎。两个函数主要流程相差不大，我们以append_entries为例，分析一下写入Log Entry的主要流程。函数流程图如下所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-84bb92948772875b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* 检查日志连续性：主要检查last_raft_index 是否和追加的Log Entry保持连续。
* 获取Last_Segment：检查last_segment是否超过Max_Segment_Size，如果超过则进行rolling操作（保存最后一个segment，并生成一个新的segment）。如果文件大小未超过Max_Segment_Size，则直接返回。
* 循环追加日志：追加Log Entry到文件末尾。
* Last_Segment强制刷盘：调用fsync函数强制刷盘。

### 2.4 读数据流程
根据raft_index读取对应的raft Log，根据我们前面提到的索引信息，braft很容易实现，流程图如下所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-ff64112d601ce14c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


get_entry是入口函数，get_segment函数主要是通过raft_index来定位到segment，通过之前建立的Map范围信息很容易定位到。然后根据每个segment的Vector索引数组，定位到raft_index对应的文件偏移信息。然后读取文件。

### 2.5 删除数据流程
删除数据分为两类：

* 从前往后删除，对应的函数是：

```
SegmentLogStorage::truncate_prefix(const int64_t first_index_kept)
```
truncate_prefix函数先将first_index_kept保存到Log_meta文件中，这样保证了即使后续的文件删除操作失败时，也可以知道整个日志的起始raft_index是多少。保存完first_index_kept之后，将first_index_kept之前的segment文件全部删除。

* 从后往前删除，对应的函数是：

```
int SegmentLogStorage::truncate_suffix(const int64_t last_index_kept) 
```
主要用于raft lib中删除未达成一致的Log Entry。根据last_index_kept找到对应的文件偏移，然后截断文件。如果跨文件，还需要删除最后一个segment文件，然后再截断之前一个segment的内容。

##3.测试
在test/test_log.cpp文件中，包含SegmentLogStorage类中主要的接口函数的单元测试，对理解SegmentLogStorage有比较大的帮助。

## 4.总结
Braft的日志存储引擎，主要用于存储raft log。当执行完一次snapshot操作后，就可以进行Log Compaction。将snapshot之前的raft log全部删除。这使得Braft可以将Log的索引信息全部存储在内存中，因为存储引擎中的Raft Log Entry不会太大。这样追加或读取Raft Log只需要一次磁盘操作，性能方面有保证。



