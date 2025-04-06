+++
title = "etcd raft"
date = "2025-04-05T09:57:03+08:00"
draft = false
categories = ["go"]
tags = ["etcd", "raft", "code"]
author = ["wiser"]
description = "Source code of etcd and raft"
ShowWordCount = true
+++

# 前言

在现如今微服务架构横行的今天，服务间的注册与发现是保证微服务架构稳定运行的关键。而 etcd 组件又常被用作服务注册与发现的基础组件，因此了解并熟悉它是每一个开发人员所必须的技能。在此记录笔者阅读源代码的过程以及一些分享。

这篇博客是基于 etcd v3.5.21 版本，打算从以下三个方面全面了解 etcd：

1. etcd 的核心 raft 共识算法的实现
2. etcd 的存储以及 MVCC 的实现
3. etcd 的 API

# etcd server

首先需要了解清楚 etcd server 的启动流程，etcd server 的 main 文件在 `server/main.go`，从这里一直追踪下去会发现每个 etcd server 都会启动一个 raft node，在 `etcdserver.NewServer` 函数中可以找到不同情况下 raft node 的启动。由于函数比较长，这里保留我认为比较重要的部分。

```go
func NewServer(cfg config.ServerConfig) (srv *EtcdServer, err error) {
    ...

	haveWAL := wal.Exist(cfg.WALDir())

    ...

	switch {
	case !haveWAL && !cfg.NewCluster:
        ...

		remotes = existingCluster.Members()
		cl.SetID(types.ID(0), existingCluster.ID())
		cl.SetStore(st)
		cl.SetBackend(be)
		id, n, s, w = startNode(cfg, cl, nil)
		cl.SetID(id, existingCluster.ID())

	case !haveWAL && cfg.NewCluster:
        ...

		cl.SetStore(st)
		cl.SetBackend(be)
		id, n, s, w = startNode(cfg, cl, cl.MemberIDs())
		cl.SetID(id, cl.ID())

	case haveWAL:
        ...

		if !cfg.ForceNewCluster {
			id, cl, n, s, w = restartNode(cfg, snapshot)
		} else {
			id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
		}

		cl.SetStore(st)
		cl.SetBackend(be)
		cl.Recover(api.UpdateCapability)
		if cl.Version() != nil && !cl.Version().LessThan(semver.Version{Major: 3}) && !beExist {
			os.RemoveAll(bepath)
			return nil, fmt.Errorf("database file (%v) of the backend is missing", bepath)
		}

	default:
		return nil, fmt.Errorf("unsupported bootstrap config")
	}

	if terr := fileutil.TouchDirAll(cfg.Logger, cfg.MemberDir()); terr != nil {
		return nil, fmt.Errorf("cannot access member directory: %v", terr)
	}

    ....

	return srv, nil
}
```

可以看到，这里分了不同情况去启动 raft node，包括是否有 WAL（write ahead log），是否是新集群；无论何种情况最终都会调用 `startNode` 函数启动 raft node。在 `startNode` 中会判断配置 `initial-cluster` 是否为空，如果为空，则进入 `raft.RestartNode` 函数重新校验参数，如果非法则 panic；否则调用 `raft.StartNode` 函数启动节点。

```go
// StartNode returns a new Node given configuration and a list of raft peers.
// It appends a ConfChangeAddNode entry for each given peer to the initial log.
//
// Peers must not be zero length; call RestartNode in that case.
func StartNode(c *Config, peers []Peer) Node {
	if len(peers) == 0 {
		panic("no peers given; use RestartNode instead")
	}
	rn, err := NewRawNode(c)
	if err != nil {
		panic(err)
	}
	rn.Bootstrap(peers)

	n := newNode(rn)

	go n.run()
	return &n
}
```

# Raft node states

从这里开始就进入 raft 共识算法的逻辑了。在继续阅读源码前，需要先了解 raft 共识算法状态转移过程，这样才会理解代码的运行逻辑。

![Raft State](/img/etcd-raft/Raft-server-states.png)

1. `start up`：起始状态，所有节点刚启动的时候自动进入 follower 状态
2. `time out, start election`：follower 启动之后，每个节点都会开启一个心跳定时器，当这个定时器到期时，将切换到 candidate 状态并发起选举（任期号递增），每次角色转换都会随机设置一个选举超时，保证迅速选出 leader
3. `time out, new election`：进入 candidate 状态后并选举，但是在选举超时，还没有选出一个新的 leader，那么还会保持 candidate 状态重新开始一次选举（任期号递增）
4. `receives votes from majority of servers`：当 candidate 状态的节点，收到了超过半数的节点选票，那么将状态切换成为 leader
5. `discovers current leader or new term`：candidate 状态的节点，如果收到了来自 leader 的消息，或者更高任期的消息，表示已经存在 leader，则将状态切换成 follower
6. `discovers server with higher term`：leader 状态下如果收到来自更高任期号的消息，将切换到 follower 状态

以上便是 raft node 的状态转移过程。因此当一个新节点启动时，它会进入 follower 状态，因此在 `raft.newRaft` 函数以及 `RawNode.Bootstrap` 方法中会都会调用 `raft.becomeFollower` 方法把节点置为 follower。

# Leader election

在 follower 的选举定时器超时，就会触发 leader election，那么选举定时器是在哪里设置的呢？首先要说明配置，在 `etcdmain.startEtcdOrProxyV2` 函数中调用了 `etcdmain.newConfig` 以及 `embed.NewConfig` 函数，这里会初始化和 Tick 有关的两个配置 `TickMs: 100` 以及 `ElectionMs: 1000`。

通过以上配置，会初始化 `ServerConfig` 对象，并设置 `ServerConfig.TickMs = TickMs` 以及 <span id="election-ticks">`ServerConfig.ElectionTicks = ElectionMs / TickMs`</span> 即 10；然后通过 `etcdserver.NewServer` 初始化 `EtcdServer` 对象，在这个对象里面包含了 raft node。接着调用 `EtcdServer.Start` 方法并调用 `raftNode.start` 方法，在这里会启用一个 goroutine 不停的处理 `raftNode` 的状态，ticker 以及存储等信息。

```go
go func() {
    defer r.onStop()
    islead := false

    for {
        select {
        case <-r.ticker.C:
            r.tick()
        case rd := <-r.Ready():
            ...
        }
    }
}
```

在上述 <span id="heartbeat-ticker">`r.ticker.C`</span> 的 channel 触发超时，就代表这节点超时没有收到 leader 或者 candidate 节点的信息，因此触发超时，发起选举。这里的超时的时间设置在 `etcdserver.NewServer` 函数中，通过 `heartbeat := time.Duration(cfg.TickMs) * time.Millisecond` 即 100ms。

当超时触发选举会调用 `raftNode.tick` 方法：

```go
// raft.Node does not have locks in Raft package
func (r *raftNode) tick() {
	r.tickMu.Lock()
	r.Tick()
	r.latestTickTs = time.Now()
	r.tickMu.Unlock()
}
```

这里 `t.Tick` 方法调用，实际是调用 `Node` interface 的方法；并且在 `EtcdServer` 对象中 `r` 是 `RaftNode` 对象，而 `RaftNode` 结构体声明如下：

```go
type raftNode struct {
	lg *zap.Logger

	tickMu *sync.RWMutex
	// timestamp of the latest tick
	latestTickTs time.Time
	raftNodeConfig

	// a chan to send/receive snapshot
	msgSnapC chan raftpb.Message

	// a chan to send out apply
	applyc chan apply

	// a chan to send out readState
	readStateC chan raft.ReadState

	// utility
	ticker *time.Ticker
	// contention detectors for raft heartbeat message
	td *contention.TimeoutDetector

	stopped chan struct{}
	done    chan struct{}
}

type raftNodeConfig struct {
	lg *zap.Logger

	// to check if msg receiver is removed from cluster
	isIDRemoved func(id uint64) bool
	raft.Node
	raftStorage *raft.MemoryStorage
	storage     Storage
	heartbeat   time.Duration // for logging
	// transport specifies the transport to send and receive msgs to members.
	// Sending messages MUST NOT block. It is okay to drop messages, since
	// clients should timeout and reissue their messages.
	// If transport is nil, server will panic.
	transport rafthttp.Transporter
}
```

可以看到，最终包含了 `raft.Node` interface，而这个 `raft.Node` interface 最终是由 `etcdserver.startNode` 函数返回的 <span id="raft-node">`raft.node`</span> 结构体实现。在 `raft.StartNode` 函数中会调用 `raft.newNode` 函数返回 `raft.node` 对象，并启动一个 goroutine 运行 `node.run` 方法：

```go
// StartNode returns a new Node given configuration and a list of raft peers.
// It appends a ConfChangeAddNode entry for each given peer to the initial log.
//
// Peers must not be zero length; call RestartNode in that case.
func StartNode(c *Config, peers []Peer) Node {
	if len(peers) == 0 {
		panic("no peers given; use RestartNode instead")
	}
	rn, err := NewRawNode(c)
	if err != nil {
		panic(err)
	}
	rn.Bootstrap(peers)

	n := newNode(rn)

	go n.run()
	return &n
}
```

`node.run` 就是 raft 节点处理不同事件的方法：

```go
func (n *node) run() {
    ...

	for {
        ...

		select {
		// TODO: maybe buffer the config propose if there exists one (the way
		// described in raft dissertation)
		// Currently it is dropped in Step silently.
		case pm := <-propc:
			m := pm.m
			m.From = r.id
			err := r.Step(m)
			if pm.result != nil {
				pm.result <- err
				close(pm.result)
			}
		case m := <-n.recvc:
			// filter out response message from unknown From.
			if pr := r.prs.Progress[m.From]; pr != nil || !IsResponseMsg(m.Type) {
				r.Step(m)
			}
		case cc := <-n.confc:
            ...
		case <-n.tickc:
			n.rn.Tick()
        ...
		case <-n.stop:
			close(n.done)
			return
		}
	}
}
```

这里比较关键的有：

- `propc`：处理客户端发送的请求<span id="prop-channel"></span>
- `recvc`：处理从其他节点接收的消息<span id="recv-channel"></span>
- `confc`：处理配置的变更以及集群节点变化
- `tickc`：心跳超时处理
- `n.stop`：停止信号

因此，**当上述 [`ticker`](#heartbeat-ticker) 超时而调用 `r.Tick` 方法时，实际调用的就是 `raft.node.Tick` 方法。**

```go
// Tick increments the internal logical clock for this Node. Election timeouts
// and heartbeat timeouts are in units of ticks.
func (n *node) Tick() {
	select {
	case n.tickc <- struct{}{}:
	case <-n.done:
	default:
		n.rn.raft.logger.Warningf("%x A tick missed to fire. Node blocks too long!", n.rn.raft.id)
	}
}
```

当一个 follower 收到超时信号，就会调用 `raft.RawNode.raft.tick` 函数，而不同的角色这个 tick 函数是不同的，分别在 `raft.becomeFollower`，`raft.becomeCandidate` 以及 `raft.becomeLeader` 中设置，这里方便后续说明统一称为 `become`[^1] 方法，另外不同角色还有 `step`[^2] 函数用于处理接收的消息。

```go
func (r *raft) becomeFollower(term uint64, lead uint64) {
	r.step = stepFollower
	r.reset(term)
	r.tick = r.tickElection
	r.lead = lead
	r.state = StateFollower
	r.logger.Infof("%x became follower at term %d", r.id, r.Term)
}

// tickElection is run by followers and candidates after r.electionTimeout.
func (r *raft) tickElection() {
	r.electionElapsed++

	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```

可以看到，当触发超时，follower 会给自己发送 `MsgHup` 消息，在 `raft.Step` 函数中会根据不同的消息类型做不同的处理。在处理 `MsgHup` 消息会调用 `raft.hup` 方法以及 `raft.campaign` 方法。

```go
func (r *raft) campaign(t CampaignType) {
    ...

	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
		r.becomePreCandidate()
		voteMsg = pb.MsgPreVote
		// PreVote RPCs are sent for the next term before we've incremented r.Term.
		term = r.Term + 1
	} else {
		r.becomeCandidate()
		voteMsg = pb.MsgVote
		term = r.Term
	}
	if _, _, res := r.poll(r.id, voteRespMsgType(voteMsg), true); res == quorum.VoteWon {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			r.campaign(campaignElection)
		} else {
			r.becomeLeader()
		}
		return
	}
	var ids []uint64
	{
		idMap := r.prs.Voters.IDs()
		ids = make([]uint64, 0, len(idMap))
		for id := range idMap {
			ids = append(ids, id)
		}
		sort.Slice(ids, func(i, j int) bool { return ids[i] < ids[j] })
	}
	for _, id := range ids {

        ...

		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

**Follower 节点会把自己变为 candidate 角色并增大 term，然后给其他节点发送 `voteMsg`，发起选举。**

这里特别说明以下，在 `becomeFollower`，`becomeCandidate` 以及 `becomeLeader` 方法中，都会调用 `raft.reset` 方法，在 `reset` 方法中会调用 `resetRandomizedElectionTimeout` 方法：

```go
func (r *raft) resetRandomizedElectionTimeout() {
	r.randomizedElectionTimeout = r.electionTimeout + globalRand.Intn(r.electionTimeout)
}
```

因此，每次的角色转换都会随机设置 `randomizedElectionTimeout` 字段，其中 `electionTimeout` 是在 `raft.newRaft` 函数中初始化为上述的 [`ElectionTicks`](#election-ticks)。

当节点变为 candidate 时，会先调用 `becomeCandidate` 方法，然后处于 candidate 角色的节点，在每次心跳超时，都会调用 `raft.tickElection` 方法，并增大 `electionElapsed`；当 `electionElapsed` 大于等于 `randomizedElectionTimeout` 时，还没有选举出 leader，说明当前集群中可能有其他节点也发起了 leader election 并造成平票的情况；因此，它会再次调用 `raft.Step` 方法并发送 `MsgHup` 消息，接着增大 term 并重置 `randomizedElectionTimeout`，如此循环直到选出 leader 节点。

处于其他角色的节点收到 `voteMsg` 消息时，会判断消息的 term 与自身节点的 term，如果比自己大，则会通过 `raft.Step` 方法中的逻辑判断自己是否能够投票，并且是投赞成票还是反对票。

```go
....

	case pb.MsgVote, pb.MsgPreVote:
		// We can vote if this is a repeat of a vote we've already cast...
		canVote := r.Vote == m.From ||
			// ...we haven't voted and we don't think there's a leader yet in this term...
			(r.Vote == None && r.lead == None) ||
			// ...or this is a PreVote for a future term...
			(m.Type == pb.MsgPreVote && m.Term > r.Term)
		// ...and we believe the candidate is up to date.
		if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
			r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
			if m.Type == pb.MsgVote {
				// Only record real votes.
				r.electionElapsed = 0
				r.Vote = m.From
			}
		} else {
			r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
		}
...

```

处于 candidate 角色的节点，通过上述 [`recvc`](#recv-channel) channel 收到消息会调用 `raft.Step` 函数以及 `raft.step` 方法，`raft.step` 方法每个角色不同，并在 `become` 方法中设置；因此，对于 candidate 角色的节点来说 `raft.step` 方法就是 `raft.stepCandidate` 函数。

```go
// stepCandidate is shared by StateCandidate and StatePreCandidate; the difference is
// whether they respond to MsgVoteResp or MsgPreVoteResp.
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
	case pb.MsgProp:
		r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
		return ErrProposalDropped
	case pb.MsgApp:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleAppendEntries(m)
	case pb.MsgHeartbeat:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleHeartbeat(m)
	case pb.MsgSnap:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleSnapshot(m)
	case myVoteRespType:
		gr, rj, res := r.poll(m.From, m.Type, !m.Reject)
		r.logger.Infof("%x has received %d %s votes and %d vote rejections", r.id, gr, m.Type, rj)
		switch res {
		case quorum.VoteWon:
			if r.state == StatePreCandidate {
				r.campaign(campaignElection)
			} else {
				r.becomeLeader()
				r.bcastAppend()
			}
		case quorum.VoteLost:
			// pb.MsgPreVoteResp contains future term of pre-candidate
			// m.Term > r.Term; reuse r.Term
			r.becomeFollower(r.Term, None)
		}
	case pb.MsgTimeoutNow:
		r.logger.Debugf("%x [term %d state %v] ignored MsgTimeoutNow from %x", r.id, r.Term, r.state, m.From)
	}
	return nil
}
```

在这个函数中，对于不同消息类型，candidate 节点会做不同的处理逻辑；当收到 `MsgVoteResp`  消息时，会调用 `raft.poll` 方法统计收到的票数；总共有三种结果：

- `quorum.VoteWon`：表示赢得选举，变成 leader
- `quorum.VoteLost`：表示选举失败，变成 follower
- `quorum.VotePending`：等待下一个 `MsgVoteResp` 消息

在上述 [raft 节点状态](#raft-node-states)转换的其他情况，都可以在 `become` 方法以及 `step` 函数中找到对应的状态转换。

# Leader term

当一个节点选举成为 leader 后，leader 节点对收到的消息或者心跳的发送，都在 `raft.stepLeader` 函数中：

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	case pb.MsgBeat:
		r.bcastHeartbeat()
		return nil
	case pb.MsgCheckQuorum:
		// The leader should always see itself as active. As a precaution, handle
		// the case in which the leader isn't in the configuration any more (for
		// example if it just removed itself).
		//
		// TODO(tbg): I added a TODO in removeNode, it doesn't seem that the
		// leader steps down when removing itself. I might be missing something.
		if pr := r.prs.Progress[r.id]; pr != nil {
			pr.RecentActive = true
		}
		if !r.prs.QuorumActive() {
			r.logger.Warningf("%x stepped down to follower since quorum is not active", r.id)
			r.becomeFollower(r.Term, None)
		}
		// Mark everyone (but ourselves) as inactive in preparation for the next
		// CheckQuorum.
		r.prs.Visit(func(id uint64, pr *tracker.Progress) {
			if id != r.id {
				pr.RecentActive = false
			}
		})
		return nil
	case pb.MsgProp:
        ...

		if !r.appendEntry(m.Entries...) {
			return ErrProposalDropped
		}
		r.bcastAppend()
		return nil

        ...
	}

	// All other message types require a progress for m.From (pr).
	pr := r.prs.Progress[m.From]
	if pr == nil {
		r.logger.Debugf("%x no progress available for %x", r.id, m.From)
		return nil
	}
	switch m.Type {
	case pb.MsgAppResp:
		pr.RecentActive = true

		if m.Reject {
			nextProbeIdx := m.RejectHint
			if m.LogTerm > 0 {
				nextProbeIdx = r.raftLog.findConflictByTerm(m.RejectHint, m.LogTerm)
			}
			if pr.MaybeDecrTo(m.Index, nextProbeIdx) {
				r.logger.Debugf("%x decreased progress of %x to [%s]", r.id, m.From, pr)
				if pr.State == tracker.StateReplicate {
					pr.BecomeProbe()
				}
				r.sendAppend(m.From)
			}
		} else {
			oldPaused := pr.IsPaused()
			if pr.MaybeUpdate(m.Index) {
				switch {
				case pr.State == tracker.StateProbe:
					pr.BecomeReplicate()
				case pr.State == tracker.StateSnapshot && pr.Match >= pr.PendingSnapshot:
					pr.BecomeProbe()
					pr.BecomeReplicate()
				case pr.State == tracker.StateReplicate:
					pr.Inflights.FreeLE(m.Index)
				}

				if r.maybeCommit() {
					// committed index has progressed for the term, so it is safe
					// to respond to pending read index requests
					releasePendingReadIndexMessages(r)
					r.bcastAppend()
				} else if oldPaused {
					// If we were paused before, this node may be missing the
					// latest commit index, so send it.
					r.sendAppend(m.From)
				}
				// We've updated flow control information above, which may
				// allow us to send multiple (size-limited) in-flight messages
				// at once (such as when transitioning from probe to
				// replicate, or when freeTo() covers multiple messages). If
				// we have more entries to send, send as many messages as we
				// can (without sending empty messages for the commit index)
				for r.maybeSendAppend(m.From, false) {
				}
				// Transfer leadership is in progress.
				if m.From == r.leadTransferee && pr.Match == r.raftLog.lastIndex() {
					r.logger.Infof("%x sent MsgTimeoutNow to %x after received MsgAppResp", r.id, m.From)
					r.sendTimeoutNow(m.From)
				}
			}
		}
	case pb.MsgHeartbeatResp:
		pr.RecentActive = true
		pr.ProbeSent = false

		// free one slot for the full inflights window to allow progress.
		if pr.State == tracker.StateReplicate && pr.Inflights.Full() {
			pr.Inflights.FreeFirstOne()
		}
		if pr.Match < r.raftLog.lastIndex() {
			r.sendAppend(m.From)
		}

		if r.readOnly.option != ReadOnlySafe || len(m.Context) == 0 {
			return nil
		}

		if r.prs.Voters.VoteResult(r.readOnly.recvAck(m.From, m.Context)) != quorum.VoteWon {
			return nil
		}

		rss := r.readOnly.advance(m)
		for _, rs := range rss {
			if resp := r.responseToReadIndexReq(rs.req, rs.index); resp.To != None {
				r.send(resp)
			}
		}
    ...
	}
	return nil
}

```

主要分析发送 `MsgHeartbeat` 和 `MsgApp` 以及接收 `MsgHeartbeatResp` 和 `MsgAppResp` 的逻辑。

## Heartbeat

Leader 节点的会通过心跳机制维持它的权威，因此当 leader 节点的 ticker 超时，会触发 `raft.tickHeartbeat` 函数。

```go
// tickHeartbeat is run by leaders to send a MsgBeat after r.heartbeatTimeout.
func (r *raft) tickHeartbeat() {
	r.heartbeatElapsed++
	r.electionElapsed++

	if r.electionElapsed >= r.electionTimeout {
		r.electionElapsed = 0
		if r.checkQuorum {
			r.Step(pb.Message{From: r.id, Type: pb.MsgCheckQuorum})
		}
		// If current leader cannot transfer leadership in electionTimeout, it becomes leader again.
		if r.state == StateLeader && r.leadTransferee != None {
			r.abortLeaderTransfer()
		}
	}

	if r.state != StateLeader {
		return
	}

	if r.heartbeatElapsed >= r.heartbeatTimeout {
		r.heartbeatElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgBeat})
	}
}
```

在 `tickHeartbeat` 函数中，最要包含两个逻辑：

1. 触发检查机制：当 leader 节点的选举超时后，会发送 `MsgCheckQuorum` 消息给自己，在 `raft.stepLeader` 函数中，收到该消息后会检查与自己保持通信的 follower 节点，查看 follower 是否仍然满足半数以上的数量，如果不满足，则自己变为 follower 节点；这里的检查机制是通过 `RecentActive` 字段判断，当收到 follower 节点的 `MsgAppResp` 或者 `MsgHeartbeatResp` 消息后，就会把 `RecentActive` 置为 ture，而检查结束后，就会把该字段置为 false；因此检查时就根据该字段判断 follower 是否仍与自己保持通信。

2. 触发心跳机制：这里 `heartbeatTimeout` 的值为 1，因此当 leader 节点的 ticker 超时，就会触发心跳机制，发送 `MsgBeat` 消息，在 `raft.stepLeader` 中，当收到 `MsgBeat` 消息，会调用 `raft.bcastHeartbeat` 方法，在该方法中会给 follower 节点发送 `MsgHeartbeat` 消息；当 follower 节点收到 `MsgHeartbeat` 消息时，会把自身的 `electionElapsed` 置为 0，防止发生超时发起选举，并回复 `MsgHeartbeatResp` 消息。

## Propose

在 `server/etcdserver/v3_server.go` 文件中，`EtcdServer` 会实现 `api/etcdserverpb/rpc.proto` 文件中定义的 service `KV`，以 `rpc Put(PutRequest) return (PutResponse)` 为例：

```go
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
	ctx = context.WithValue(ctx, traceutil.StartTimeKey, time.Now())
	resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
	if err != nil {
		return nil, err
	}
	return resp.(*pb.PutResponse), nil
}
```

`EtcdServer` 会调用 `EtcdServer.raftRequest` 方法并依次调用 `EtcdServer.raftRequestOnce`，`EtcdServer.processInternalRaftRequestOnce` 方法，最终调用到 `raft.Node` 接口的 `Propose` 方法；在上述过程中可得治 `raft.Node` 接口由 [`raft.node`](#raft-node) 对象实现，因此查看 `raft.node.Propose` 方法：

```go
func (n *node) Propose(ctx context.Context, data []byte) error {
	return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}
```

`raft.node` 对象会调用 `node.stepWait` 方法，并最终调用 `node.stepWithWaitOption` 方法，由 [`node.propc`](#prop-channel) channel 发送。


```go
func (n *node) stepWithWaitOption(ctx context.Context, m pb.Message, wait bool) error {
	if m.Type != pb.MsgProp {
		select {
		case n.recvc <- m:
			return nil
		case <-ctx.Done():
			return ctx.Err()
		case <-n.done:
			return ErrStopped
		}
	}
	ch := n.propc
	pm := msgWithResult{m: m}
	if wait {
		pm.result = make(chan error, 1)
	}
	select {
	case ch <- pm:
		if !wait {
			return nil
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	select {
	case err := <-pm.result:
		if err != nil {
			return err
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	return nil
}
```

在 `node.run` 方法中，从 `propc` 中接收到的消息由 `raft.Step` 方法处理，最终由 `stepLeader` 函数处理； 当收到 `MsgProp` 消息后，leader 节点会通过 `raft.bcastAppend` 方法向 follower 节点发送 `MsgApp` 消息；当 follower 节点收到 `MsgApp` 消息后，会通过 `raft.handleAppendEntries` 方法处理消息，并回复 `MsgAppResp` 消息。

这里就省略了 Leader 收到 `MsgAppResp` 消息后的处理，具体可以查看 `stepLeader` 方法中 [`case pd.MsgAppResp`][#MsgAppResp] 中的逻辑。

# Reference

[^1]: 表示 [`becomeFollower`][#become-follower]，[`becomeCandidate`][#become-candidate] 以及 [`becomeLeader`][#become-leader] 方法

[^2]: 表示 [`stepLeader`][#step-leader]，[`stepCandidate`][#step-candidate] 以及 [`stepFollower`][#step-follower] 函数

[#MsgAppResp]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L1100
[#become-follower]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L680
[#become-candidate]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L689
[#become-leader]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L718
[#step-leader]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L985
[#step-candidate]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L1370
[#step-follower]: https://github.com/etcd-io/etcd/blob/a17edfd59754d1aed29c2db33520ab9d401326a5/raft/raft.go#L1415
