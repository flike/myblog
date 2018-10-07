---
author: "flike"
date: 2018-10-07
title: "Raft的PreVote实现机制"
tags: [
    "raft",
    "etcd",
]
categories: [
    "raft",
]
---

## **1\. 背景**

在Basic Raft算法中，当一个Follower与其他节点网络隔离，如下图所示：
![image](http://upload-images.jianshu.io/upload_images/66307-64cbd82b30ffb525.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Follower_2在electionTimeout没收到心跳之后,会发起选举，并转为Candidate。每次发起选举时，会把Term加一。由于网络隔离，它既不会被选成Leader，也不会收到Leader的消息，而是会一直不断地发起选举。Term会不断增大。

一段时间之后，这个节点的Term会非常大。在网络恢复之后，这个节点会把它的Term传播到集群的其他节点，导致其他节点更新自己的term，变为Follower。然后触发重新选主，但这个旧的Follower_2节点由于其日志不是最新，并不会成为Leader。整个集群被这个网络隔离过的旧节点扰乱，显然需要避免的。

## **2\. Provote算法**

Raft作者博士论文《CONSENSUS: BRIDGING THEORY AND PRACTICE》的第9.6节 "Preventing disruptions when a server rejoins the cluster"提到了PreVote算法的大概实现思路。

在PreVote算法中，Candidate首先要确认自己能赢得集群中大多数节点的投票，这样才会把自己的term增加，然后发起真正的投票。其他投票节点同意发起选举的条件是（同时满足下面两个条件）：

*   没有收到有效领导的心跳，至少有一次选举超时。
*   Candidate的日志足够新（Term更大，或者Term相同raft index更大）。

PreVote算法解决了网络分区节点在重新加入时，会中断集群的问题。在PreVote算法中，网络分区节点由于无法获得大部分节点的许可，因此无法增加其Term。然后当它重新加入集群时，它仍然无法递增其Term，因为其他服务器将一直收到来自Leader节点的定期心跳信息。一旦该服务器从领导者接收到心跳，它将返回到Follower状态，Term和Leader一致。

## **3\. Etcd的Provote实现流程**

Etcd针对发起PreVote的节点增加了一个角色状态：StatePreCandidate。

```
const (
    StateFollower StateType = iota
    StateCandidate
    StateLeader
    StatePreCandidate
    numStates
)

```

## **3.1 节点发起PreVote流程**

1.首先节点超时，会进入Step函数，然后触发选举流程，如果配置了prevote，则会进入预选举流程，代码片段如下所示：

```
case pb.MsgHup:
        if r.state != StateLeader {
            ents, err := r.raftLog.slice(r.raftLog.applied+1, r.raftLog.committed+1, noLimit)
            if err != nil {
                r.logger.Panicf("unexpected error getting unapplied entries (%v)", err)
            }
            if n := numOfPendingConf(ents); n != 0 && r.raftLog.committed > r.raftLog.applied {
                r.logger.Warningf("%x cannot campaign at term %d since there are still %d pending configuration changes to apply", r.id, r.Term, n)
                return nil
            }

            r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
            if r.preVote {
                r.campaign(campaignPreElection)
            } else {
                r.campaign(campaignElection)
            }
        } else {
            r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
        }

```

2.节点调用r.campaign(campaignPreElection)，发送投票请求。函数流程如下所示：

```
func (r *raft) campaign(t CampaignType) {
    var term uint64
    var voteMsg pb.MessageType
    if t == campaignPreElection {
        r.becomePreCandidate()
        voteMsg = pb.MsgPreVote
        // PreVote RPCs are sent for the next term before we've incremented r.Term.
        //关键点：这里raft的term不会增加，先以r.Term + 1询问其他节点，而不增加自己的真实term
        term = r.Term + 1
    } else {
        r.becomeCandidate()
        voteMsg = pb.MsgVote
        term = r.Term
    }
    //检查投票是否过半，第一次进入该函数不会执行这段逻辑。
    //流程3，会统计投票结果
    if r.quorum() == r.poll(r.id, voteRespMsgType(voteMsg), true) {
        // We won the election after voting for ourselves (which must mean that
        // this is a single-node cluster). Advance to the next state.
        if t == campaignPreElection {
            r.campaign(campaignElection)
        } else {
            r.becomeLeader()
        }
        return
    }
    //向其他节点发送投票请求
    for id := range r.prs {
        if id == r.id {
            continue
        }
        r.logger.Infof("%x [logterm: %d, index: %d] sent %s request to %x at term %d",
            r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), voteMsg, id, r.Term)

        var ctx []byte
        if t == campaignTransfer {
            ctx = []byte(t)
        }
        r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
    }
}

```

3.当发起prevote节点收到响应消息以后，会进入stepCandidate函数，stepCandidate函数是PreCandidate状态和Candidate状态共用的。当收到其他节点对投票的响应时，重新计算自己的票数。如果达到大多数，PreCandidate会变为Candidate状态，发起真正的选举。代码片段如下所示：

```
func stepCandidate(r *raft, m pb.Message) error {
    // Only handle vote responses corresponding to our candidacy (while in
    // StateCandidate, we may get stale MsgPreVoteResp messages in this term from
    // our pre-candidate state).
    var myVoteRespType pb.MessageType
    if r.state == StatePreCandidate {
        myVoteRespType = pb.MsgPreVoteResp
    } else {
        myVoteRespType = pb.MsgVoteResp
    }
    switch m.Type {
    ...
    case myVoteRespType:
      //统计赞成票和反对票
        gr := r.poll(m.From, m.Type, !m.Reject)
        r.logger.Infof("%x [quorum:%d] has received %d %s votes and %d vote rejections", r.id, r.quorum(), gr, m.Type, len(r.votes)-gr)
        switch r.quorum() {
        case gr:
          //当赞成票过半后，PreVote直接转入第二个阶段：正式选举
            if r.state == StatePreCandidate {
                r.campaign(campaignElection)
            } else {
              //如果已经是StateCandidate,则直接变为Leader，选举结束。
                r.becomeLeader()
                r.bcastAppend()
            }
        case len(r.votes) - gr:
            // pb.MsgPreVoteResp contains future term of pre-candidate
            // m.Term > r.Term; reuse r.Term
            //如果反对票已过半，这直接变为Follower，并且不增加term
            r.becomeFollower(r.Term, None)
        }
    ...
    }
    return nil
}

```

## **3.2 节点响应PreVote流程**

节点收到Prevote请求，都会进入Step函数，然后做相应的响应处理：
1.如果当前节点未选举超时，并且存在Leader，则不响应投票请求
2.如果满足投票要求，并且日志最新，则投赞成票，否则投反对票。

```
func (r *raft) Step(m pb.Message) error {
    // Handle the message term, which may result in our stepping down to a follower.
    switch {
    ...
    //#1
    case m.Term > r.Term:
        if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote {
            force := bytes.Equal(m.Context, []byte(campaignTransfer))
            inLease := r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout
            if !force && inLease {
                // If a server receives a RequestVote request within the minimum election timeout
                // of hearing from a current leader, it does not update its term or grant its vote
                r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d, index: %d] at term %d: lease is not expired (remaining ticks: %d)",
                    r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term, r.electionTimeout-r.electionElapsed)
                return nil
            }
        }
    ...
    case pb.MsgVote, pb.MsgPreVote:
    ...
    //#2
        // We can vote if this is a repeat of a vote we've already cast...
        canVote := r.Vote == m.From ||
            // ...we haven't voted and we don't think there's a leader yet in this term...
            (r.Vote == None && r.lead == None) ||
            // ...or this is a PreVote for a future term...
            (m.Type == pb.MsgPreVote && m.Term > r.Term)
        // ...and we believe the candidate is up to date.
        if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
            ...
            r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
            if m.Type == pb.MsgVote {
                // Only record real votes.
                r.electionElapsed = 0
                r.Vote = m.From
            } else {
            ...
            r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
        }

```

## **4\. 总结**

Prevote是一个典型的2PC协议，第一阶段先征求其他节点是否同意选举，如果同意选举则发起真正的选举操作，否则降为Follower角色。这样就避免了网络分区节点重新加入集群，触发不必要的选举操作。

