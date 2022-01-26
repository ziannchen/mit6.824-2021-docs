# Lab2: Raft

- [结构体](#结构体)
- [leader election](#leader-election)
	- [sender](#sender)
	- [handler](#handler)
- [log replication](#log-replication)
	- [replicator](#replicator)
	- [handler](#handler)
- [persistence](#persistence)
- [log compaction](#log-compaction)
- [applier](#applier)


lab原链接 <https://pdos.csail.mit.edu/6.824/labs/lab-raft.html>

Raft是一种基于复制的状态机协议，通过在多个副本服务器上存储其状态（即数据）的完整副本来实现容错。

Raft将客户端请求组织成一个称为日志的序列，并通过复制确保所有副本服务器都看到相同的日志。每个副本按日志顺序执行客户端请求，并将它们应用于本地的状态机副本。由于所有副本服务器都看到相同的日志内容，因此它们都以相同的顺序执行相同的请求，从而继续具有相同的服务状态。如果服务器出现故障但随后恢复，Raft保证只要至少半数的服务器存活，并且可以相互通信，就可以保证正常对外服务。

在本次lab中我们的任务是使用go语言实现raft。参考论文 [raft-extended](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)，我们需要实现除了集群成员变更之外的绝大部分内容。论文中我认为最核心的就是描述3个RPC的(Figure 2)这张图，我的实现大体上遵循了这张图。此外我也参考了一些工业级的raft实现，比如SOFAJraft、etcd，做了一些优化。在我秋招面试美团的一个做分布式存储的部门时，他们问了我很多关于raft的内容（虽然最后挂了）。


有些需要注意的点：
- 当收到的RPC中的term大于自身时，无条件跟随term并转为follower，这在不同的RPC handler中的处理略有不同。
- 在lab的一些测试用例中，网络将是不稳定的，带来大量随机的RPC丢包、乱序、超时。对于过期的RPC，直接抛弃不处理即可。对于是否过期的判断体现在term太小、身份不正确之类（例如follow收到append entries response）。
- 锁的使用：在接发RPC、读写channel时一定不要持有锁，不然很有可能死锁。此外有许多代码块对Raft结构中各字段是只读的，我使用了读写锁。

## 结构体

Raft结构中的各个变量和论文大致一样。
```go
type Raft struct {
	rw        sync.RWMutex        // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()
	// Your data here (2A, 2B, 2C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.

	currentState   State
	currentTerm    int
	votedFor       int
	voteFrom       map[int]bool
	logs           []LogEntry
	commitIndex    int
	lastApplied    int
	nextIndex      []int
	matchIndex     []int
	electionTimer  *time.Timer
	heartbeatTimer *time.Timer
	applyCh        chan ApplyMsg
	applierCond    sync.Cond
	replicatorCond []sync.Cond
}

func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	// Your initialization code here (2A, 2B, 2C).
	rf := &Raft{
		rw:             sync.RWMutex{},
		peers:          peers,
		persister:      persister,
		me:             me,
		dead:           -1,
		currentState:   Follower,
		currentTerm:    0,
		votedFor:       -1,
		voteFrom:       make(map[int]bool),
		logs:           make([]LogEntry, 1),
		nextIndex:      make([]int, len(peers)),
		matchIndex:     make([]int, len(peers)),
		electionTimer:  time.NewTimer(RandomizedElectionTimeout()),
		heartbeatTimer: time.NewTimer(StableHeartbeatTimeout()),
		applyCh:        applyCh,
		replicatorCond: make([]sync.Cond, len(peers)),
	}

	rf.applierCond = *sync.NewCond(&rf.rw)
	rf.logs[0] = LogEntry{0, 0, nil}
	// initialize from state persisted before a crash
	rf.readPersist(persister.ReadRaftState())
	rf.commitIndex, rf.lastApplied = rf.logs[0].Index, rf.logs[0].Index
	for i := 0; i < len(peers); i++ {
		rf.matchIndex[i], rf.nextIndex[i] = 0, rf.getLastLogEntry().Index+1
		if i == me {
			continue
		}

		rf.replicatorCond[i] = *sync.NewCond(&sync.Mutex{})
		go rf.replicator(i)
	}

	// start ticker goroutine to start elections
	go rf.ticker()

	go rf.applier(rf.applyCh)

	return rf
}
```
根据论文，日志的index和term都从1开始，所以在logs[0]处存放一个index和term均为0的dummy value。

在Make函数中启动了一些后台协程

- replicator：共len(peers)-1个，用于管理leader对每一个follower的日志复制，下文会详细介绍。
- ticker：用来触发选举和心跳timeout。
- applier：用于向applyCh中提交已经commit的日志。

## leader-election

### sender

在ticker函数中需要循环使用select监听两个timer的channel，lab的提示中说使用timer可能会有问题但我没有遇到过，懒得改了。

如果是选举计时器到期，则发起一轮选举；如果是心跳计时器到期，则发起一轮心跳。二者都要首先判断当前身份是否正确。我使用了一个map来记录当前term中投票给自己的peer，需要在每次转换为candidate时清空map。也可以每次start election时声明一个得票计数，之后使用闭包来计算。
```go
func (rf *Raft) ticker() {
	for !rf.Killed() {
		select {
		case <-rf.electionTimer.C:
			rf.rw.Lock()
			if rf.currentState != Leader {
				rf.currentState = Candidate
				rf.voteFrom = make(map[int]bool)
				rf.currentTerm++
				rf.startElection()
			}

			rf.electionTimer.Reset(RandomizedElectionTimeout())
			rf.rw.Unlock()

		case <-rf.heartbeatTimer.C:
			rf.rw.Lock()
			if rf.currentState == Leader {
				DPrintf("[Server %d] boardcast heartbeat at term %d", rf.me, rf.currentTerm)
				rf.boardcastHeartbeat(true)
			}

			rf.rw.Unlock()
		}

	}
}
```
选举需要异步对每个peer发送request vote，不然就太慢了。异步才不会阻塞ticker，能快速重置计时器。response handler中要先判断是否仍满足rf.currentTerm == args.Term && rf.currentState == Candidate，若不满足说明RPC过期，直接抛弃不处理。

我之所以没有使用闭包是因为这样难以抽象出一个 handleRequestVoteResponse 函数，代码结构不够统一。
```go
func (rf *Raft) startElection() {
	args := rf.getDefaultRequestVoteArgs()
	rf.votedFor, rf.voteFrom[rf.me] = rf.me, true
	rf.persist()
	DPrintf("[Server %d] start election at term %d", rf.me, rf.currentTerm)
	for index := range rf.peers {
		if index == rf.me {
			continue
		}

		go func(i int) {
			reply := RequestVoteReply{}
			if rf.sendRequestVote(i, &args, &reply) {
				rf.rw.Lock()
				rf.handleRequestVoteResponse(i, &args, &reply)
				rf.rw.Unlock()
			}
		}(index)
	}
}
```

### handler

handler的实现完全参照论文，先判断term是否小于自身，再判断term、voteFor和日志是否满足条件。判断voteFor时要先满足args.Term == rf.currentTerm，这是由于args.Term > rf.currentTerm时需要无条件跟随term并重置voteFor。

需要注意的是只有同意投票时才需要重置election timer，这在课程的TA的guidance中有提及，有利于在网络不稳定时仍能快速选出leader。
```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	defer rf.rw.Unlock()
	defer DPrintf("[Server %d] reply [%s] for RequestVote to %d", rf.me, reply, args.CandidateId)

	rf.rw.Lock()
	DPrintf("[Server %d][state %s term %d vote %d lastindex %d lastterm %d] receive RequestVote [%s] from %d", rf.me, StateName[rf.currentState], rf.currentTerm, rf.votedFor, rf.getLastLogEntry().Index, rf.getLastLogEntry().Term, args, args.CandidateId)

	if args.Term < rf.currentTerm || (args.Term == rf.currentTerm && rf.votedFor != -1 && rf.votedFor != args.CandidateId) {
		reply.Term, reply.VoteGranted = rf.currentTerm, false
		return
	}

	needToPersist := false
	if args.Term > rf.currentTerm {
		rf.currentTerm, rf.votedFor = args.Term, -1
		DPrintf("[Server %d] change state from Leader to Follower at term %d", rf.me, rf.currentTerm)
		rf.currentState = Follower
		needToPersist = true
	}

	if !rf.isLogUpToDate(args.LastLogIndex, args.LastLogTerm) {
		reply.Term, reply.VoteGranted = rf.currentTerm, false
		if needToPersist {
			rf.persist()
		}

		return
	}

	reply.Term, reply.VoteGranted = rf.currentTerm, true
	rf.votedFor = args.CandidateId
	rf.persist()
	rf.electionTimer.Reset(RandomizedElectionTimeout())
}
```

## log-replication

### replicator

根据每个peer的nextIndex判断发送entries或是snapshot。

response handler的实现参照论文，先判断是否过期，再判断是否成功。若成功，则更新match、next index。找到最新的复制到超过半数peer且term等于当前term的日志，更新commit。需要注意日志的term必须和当前term一致才能更新commit，不然可能会有安全性问题导致已经commit的日志被覆盖，我忘了哪个测试一直过不了后来发现就是这个原因，所以论文一定要非常仔细读。

```go
func (rf *Raft) doReplicate(i int) {
	rf.rw.RLock()
	if rf.currentState != Leader {
		rf.rw.RUnlock()
		return
	}

	if rf.nextIndex[i] <= rf.getDummyLogntry().Index {
		args := rf.getDefaultInstallSnapshotArgs()
		rf.rw.RUnlock()

		reply := InstallSnapshotReply{}
		if rf.sendInstallSnapshot(i, &args, &reply) {
			rf.rw.Lock()
			rf.handleInstallSnapshotResponse(i, args, reply)
			rf.rw.Unlock()
		}
	} else {
		args := rf.getDefaultAppendEntriesArgs(i)
		rf.rw.RUnlock()

		reply := AppendEntriesReply{}
		if rf.sendAppendEntries(i, &args, &reply) {
			rf.rw.Lock()
			rf.handleAppendEntriesReponse(i, &args, &reply)
			rf.rw.Unlock()
		}
	}
}
```

这里我参考了 [LebronAI](https://github.com/LebronAl/MIT6.824-2021/blob/master/docs/lab2.md#%E5%A4%8D%E5%88%B6%E6%A8%A1%E5%9E%8B) 的设计。

如果为每一次Start、心跳都广播发送一次append entries，则将下层的日志同步与上层的提交新指令强绑定了，会造成RPC数量过多，还会重复发送很多次相同的日志项。每次发送 rpc 都不论是发送端还是接收端都需要若干次系统调用和内存拷贝，rpc 次数过多也会对 CPU 造成不必要的压力。

这里可以做一个batching的优化，也将二者之间解耦。这里原作者参考了SOFAJraft的日志复制模型，让每个peer对于其他所有peer各维护一个replicator协程，负责在自己成为leader时对单独一个peer的日志复制。

这个协程利用条件变量 `sync.Cond` 执行 `Wait` 来避免耗费 cpu，每次需要进行一次日志复制时调用 `Signal` 唤醒。它在满足复制条件时会尽最大努力将[nextIndex, lastIndex]之间的日志复制到peer上。

由于leader使用replicator维护对于一个peer的日志复制，同一时间下最多只会发送一个RPC，若RPC丢失、超时很可能触发re-election。因此：
- 心跳计时器到期，很急，要立即发送RPC。leader commit更新时也要立即发送RPC，这个是为啥我忘记了。
- Start被调用，不急，只需调用条件变量的 `Singal`，让replicator慢慢发。

```go
func (rf *Raft) replicator(peer int) {
	rf.replicatorCond[peer].L.Lock()
	defer rf.replicatorCond[peer].L.Unlock()

	for !rf.Killed() {
		for !rf.needToReplicate(peer) {
			rf.replicatorCond[peer].Wait()
		}

		rf.doReplicate(peer)
	}
}

func (rf *Raft) needToReplicate(peer int) bool {
	rf.rw.RLock()
	defer rf.rw.RUnlock()

	return rf.currentState == Leader && rf.nextIndex[peer] <= rf.getLastLogEntry().Index
}
 
func (rf *Raft) boardcastHeartbeat(isHeartbeat bool) {
	for index := range rf.peers {
		if index == rf.me {
			continue
		}

		if isHeartbeat {
			go rf.doReplicate(index)
		} else {
			rf.replicatorCond[index].Signal()
		}
	}

	rf.heartbeatTimer.Reset(StableHeartbeatTimeout())
}
```

### handler

完全按照论文图中伪代码实现，包括了课程视频中提到的加速解决日志冲突的优化。

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.rw.Lock()
	DPrintf("[Server %d][term %d lastindex %d lastterm %d commit %d] receive AppendEntries %+v from %d", rf.me, rf.currentTerm, rf.getLastLogEntry().Index, rf.getLastLogEntry().Term, rf.commitIndex, args, args.LeaderId)
	defer rf.rw.Unlock()
	defer DPrintf("[Server %d] reply [%s] for AppendEntries to %d", rf.me, reply, args.LeaderId)

	needToPersist := false
	if args.Term < rf.currentTerm {
		reply.Success, reply.Term = false, rf.currentTerm
		return
	}

	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		needToPersist = true
	}

	rf.currentState = Follower
	rf.electionTimer.Reset(RandomizedElectionTimeout())

	if args.PrevLogIndex < rf.getDummyLogntry().Index {
		reply.Success, reply.Term = false, 0
		if needToPersist {
			rf.persist()
		}

		return
	}

	if !rf.isLogMatch(args.PrevLogIndex, args.PrevLogTerm) {
		reply.Term, reply.Success = rf.currentTerm, false
		reply.XIndex, reply.Term = rf.getConflictEntry(args.PrevLogIndex)
		if needToPersist {
			rf.persist()
		}

		return
	}

	lastLogIndex := rf.getLastLogEntry().Index
	for index, entry := range args.Entries {
		if entry.Index > lastLogIndex || rf.logs[rf.getSliceIndex(entry.Index)].Term != entry.Term {
			rf.logs = append(rf.logs[:rf.getSliceIndex(entry.Index)], args.Entries[index:]...)
			DPrintf("[Server %d] append Follower's last log index from %d to %d", rf.me, lastLogIndex, rf.getLastLogEntry().Index)
			needToPersist = true
			break
		}
	}

	rf.maybeAdvanceFollowerCommit(args.LeaderCommit)
	reply.Success = true
	if needToPersist {
		rf.persist()
	}
}
```

## persistence

论文中提到有三个变量是需要持久化的：currentTerm、votedFor、log[]，这三个量每次改变之后都应该持久化。

持久化应当在被其他协程感知（发送RPC、释放锁）之前完成，而每个函数中如果没有改变这三个量（如加锁之后发现RPC过期）则不用持久化，若有也只需持久化一次，所以我在很多地方都使用了一个 needToPersist 布尔量进行判断。这样写感觉不够优雅，暂时没想到其他方法。

## log-compaction

对于leader，在replicator中根据next index判断出需要给peer发送快照时，调用 `persister.ReadSnapshot` 获得快照并发送。

对于接收方，需要判断如果 args.LastIncludedIndex <= rf.commitIndex，则拒绝接收快照。这说明本地状态机已经至少比该快照更新（或者将要，因为applier协程已经在apply这些日志的过程中了），可能导致raft回到旧的状态。应当等待service层调用 `Snapshot` 函数来安装快照。接收快照后，异步写入到applyCh中。

对于两个service层给raft层安装快照的函数，它们的区别是：`Snapshot` 是由service层在处理apply message时判断raft state's size是否达到阈值，主动调用。`CondInstallSnapshot` 是service层在处理apply message中leader发来的更新的快照时调用，也需要再次判断是否 LastIncludedIndex <= rf.commitIndex，安装快照之后应该更新lastApplied、commitIndex。

安装快照后需要压缩日志，但是需要记录下包含在快照中的最新的日志项的index和term，我将其记录在dummy entry（即rf.log[0]）中。此外被删除的日志项需要被正确的删除使其能够被gc。

## applier

根据论文，一旦commitIndex > lastApplied，则需要将[lastApplied+1, commitIndex]中的所有日志项apply到状态机并增加lastApplied。

一开始我的实现是每次commitIndex更新，都异步起一个协程将[lastApplied+1, commitIndex]间日志写入applyCh。但是因为写channel时不能持有锁，所以这个过程只能是：

加锁 -> 深拷贝日志项 -> 释放锁 -> 写channel -> 加锁 -> 更新lastApplied -> 释放锁

日志在push完之前不会更新lastApplied，这样容易造成相同的日志项被重复apply，存在资源浪费。所以这里也可以参考之前replicator的实现思路，后台起一个applier协程，平时调用一个条件变量的 `Wait` ，被 `Signal` 唤醒时将[lastApplied+1, commitIndex]中的所有日志项apply到状态机，每次更新commitIndex时调用 `Signal`。这样即能避免日志被重复apply，也完成了 apply 日志到状态机和 raft 提交新日志之间的解耦。

```go
func (rf *Raft) applier(applyCh chan ApplyMsg) {
	for !rf.Killed() {
		rf.rw.Lock()
		for !rf.needToApply() {
			rf.applierCond.Wait()
		}

		lastApplied, commitIndex := rf.lastApplied, rf.commitIndex
		needToApply := DeepCopy(rf.logs[rf.getSliceIndex(lastApplied+1) : rf.getSliceIndex(commitIndex)+1])

		rf.rw.Unlock()
		for _, entry := range needToApply {
			applyMsg := ApplyMsg{
				CommandValid:  true,
				Command:       entry.Command,
				CommandIndex:  entry.Index,
				CommandTerm:   entry.Term,
				RaftStateSize: rf.persister.RaftStateSize(),
			}

			applyCh <- applyMsg
			DPrintf("[Server %d] applied log [index %d] at term %d", rf.me, entry.Index, entry.Term)
		}

		rf.rw.Lock()
		if commitIndex > rf.lastApplied {
			rf.lastApplied = commitIndex
		}

		rf.rw.Unlock()
	}
}
```

需要注意因为写channel时是不加锁的，而写channel是可能出现并发的，可能存在follower接受leader的 `InstallSnapshot` 之后将新的snapshot写入channel。此时channel的写入顺序可能是：旧日志1 -> 新快照 -> 旧日志2。

service层读channel是线性的，在读出snapshot并调用 `CondInstallSnapshot` 后会更新lastApplied、commitIndex。因此在raft层apply完日志之后，重新获得锁去更新lastApplied时要注意不能回退，在这二者之间可能service层已经对更新的快照调用过 `CondInstallSnapshot` 了（新快照的 lastIncludeIndex 一定大于 commitIndex ）。



