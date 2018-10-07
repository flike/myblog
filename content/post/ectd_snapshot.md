---
title: "Etcd raft lib的snapshot处理流程"
date: 2018-10-07T19:43:53+08:00
draft: false
tags: [
    "raft",
]
categories: [
    "raft",
]
---

snapshot的是系统状态的完整快照，其他系统接收和回放snapshot，将自身数据恢复到一个一致性状态。本文介绍一下etcd raft lib如何支持snapshot功能，主要包括：

*   生成snapshot
*   Leader发送snapshot
*   Follower接收和应用snapshot

## 1.数据结构

```
type ConfState struct {
    Nodes            []uint64 `protobuf:"varint,1,rep,name=nodes" json:"nodes,omitempty"`
    XXX_unrecognized []byte   `json:"-"`
}
type SnapshotMetadata struct {
    ConfState        ConfState `protobuf:"bytes,1,opt,name=conf_state,json=confState" json:"conf_state"`
    Index            uint64    `protobuf:"varint,2,opt,name=index" json:"index"`
    Term             uint64    `protobuf:"varint,3,opt,name=term" json:"term"`
    XXX_unrecognized []byte    `json:"-"`
}
 
type Snapshot struct {
    Data             []byte           `protobuf:"bytes,1,opt,name=data" json:"data,omitempty"`
    Metadata         SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata"`
    XXX_unrecognized []byte           `json:"-"`
}
```

## 2.生成snapshot

etcd会在每次apply entry的时候，判断是否需要生成snapshot。判断的条件就是apply raft index 和上一个snapshot 中的index差距是否超过一定范围。如果超过，则触发snapshot服务。具体函数流程如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/66307-acef615cfcac1a54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于etcd的snapshot生成和VDL生成snapshot不一样，这里就不做详细分析。

## 3.Leader发送snapshot

Leader在stepLeader函数中，会根据Follower的Next值向该Follower发送raft log。如果Leader上raft entry不存在，则会发送snapshot给follower。详细的函数调用流程如下图所示：

![image.png](https://upload-images.jianshu.io/upload_images/66307-e587f8ccb0c42a20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


各个函数作用如下：

*   sendAppend:Leader根据Follower的进度发送raft log。
*   r.raftLog.snapshot:调用存储引擎，获取最新的snapshot。
*   pr.becomeSnapshot:将该follower状态修改为ProgressStateSnapshot，表示正在发送snapshot。
*   send:将snapshot类型的message发送给follower。
*   newReady:将snapshot类型的message封装在一个Ready结构中，最终raft状态机会将Ready结构体交给应用来处理。
*   raftNode.start:在应用的外围循环中，会不断处理raft状态机传递出来的Ready结构体，对于snapshot会在r.processMessages函数中做处理。
*   r.processMessages:将包含snapshot信息的message传递给msgSnapC。
*   applyAll:该函数会接收msgSnapC中的消息，并调用transport发送snapshot。
*   transport.SendSnapshot:调用transport层发送snapshot。
*   peer.sendSnap:调用对应的peer的发送snapshot函数。
*   snapshotSender.send:真正的snapshot发送操作。

## 4.Follower接收和应用snapshot

Follower上对snapshot的处理主要分为接收和应用snapshot两个部分，http包会创建一个goroutine来专门接收snapshot。接收完成后，会将消息传入raft状态机，然后通过消息类型来驱动状态机。具体的函数调用流程如下所示：
[图片上传失败...(image-58cdb1-1522286696016)]

上述主要函数作用如下所示：

![image.png](https://upload-images.jianshu.io/upload_images/66307-c82f2e7046f9fd87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


*   snapshotHandler.ServeHTTP:snapshot的handler会一直等待接收snapshot，如果收到snapshot会将其写到磁盘。
*   snapshotter.SaveDBFrom:将收到snapshot会写到磁盘
*   Process:将收到snapshot的消息发送到raft 状态机
*   stepFollower:Follower收到MsgSnap消息后进入该函数。
*   handleSnapshot:snapshot对应的消息类型是:MsgSnap，会调用handleSnapshot。
*   r.restore:清空unstable中的entries，并将snapshot保存到unstable中。
*   newReady:根据unstable中的snapshot设置Ready结构体。
*   raftNode.start:将snapshot发送到applyAll对应的goroutine
*   applyAll:调用applySnapshot
*   applySnapshot:etcd中应用snapshot的过程。

