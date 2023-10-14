paper：

> [In Search of an Understandable Consensus Algorithm (mit.edu)](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)

guild:

You may find this [guide](https://thesquareplanet.com/blog/students-guide-to-raft/) useful, as well as this advice about [locking](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt) and [structure](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt) for concurrency.

# Raft

> 参考：
>
> [MIT6.824-lab2B-2022篇（万字推导思路及代码构建）_mit6.824 lab2b_幸平xp的博客-CSDN博客](https://blog.csdn.net/weixin_45938441/article/details/124797074)

## 2A leader election

### 1. ticker

heartbeat =100ms // tester要求leader一秒钟内不要发送超过十次心跳 

electionTimeOut = 150~300ms

```go
sleep heartbeat
Time after用法
resetElectionTime
```



**worker重置选举时间：**

- 收到现任 Leader 的心跳请求，如果 AppendEntries 请求参数的任期是过期的(`args.Term < currentTerm`)，不能重置；
- 节点开始了一次选举；
- **节点投票给了别的节点**（没投的话也不能重置）
- 得到相应从candidate变为follower

1请求投票、2投给别人票、3收到心跳，都要reset 选举时间



### 2. 选举过程：

- 发送拉票

- 得到半数票 （这里的处理方式很多，channel或者sync Once）

- 成为leader，发送心跳

拉票过程：若收到心跳，直接变follower。**如果都没选出来，啥也不需要做**，**等选举时间结束重新选举**

**两个RPC都是异步用go func**

```go
for i := 0; i < rf.peerCount(); i++ {
    if i == rf.me {
        // 不发给自己
        continue
    }
    go rf.candidateRequestVote(i)
}
for i := 0; i < rf.peerCount(); i++ {
    if i == rf.me {
        // 不发给自己
        continue
    }
    go rf.leaderSendEntries(i)
}

```

## 易错点：

- 如何设置合理的resetElectionTimeOut和hearbeat，也就是ticker函数
- 处理接受到的vote票数，过半直接当选
- 当了leader直接发送心跳给别人
- 收到candidate的RequestVote：
- 收到Leader的心跳：

- 在修改state的地方都要加锁

# 2B

hint

> - Your first goal should be to pass `TestBasicAgree2B()`. Start by implementing `Start()`, then write the code to send and receive new log entries via `AppendEntries` RPCs, following Figure 2. Send each newly committed entry on `applyCh` on each peer.
> - You will need to implement the election restriction (section 5.4.1 in the paper).
> - One way to fail to reach agreement in the early Lab 2B tests is to hold repeated elections even though the leader is alive. Look for bugs in election timer management, or not sending out heartbeats immediately after winning an election.
> - Your code may have loops that repeatedly check for certain events. Don't have these loops execute continuously without pausing, since that will slow your implementation enough that it fails tests. Use Go's [condition variables](https://golang.org/pkg/sync/#Cond), or insert a `time.Sleep(10 * time.Millisecond)` in each loop iteration.
> - Do yourself a favor for future labs and write (or re-write) code that's clean and clear. For ideas, re-visit our the [Guidance page](https://pdos.csail.mit.edu/6.824/labs/guidance.html) with tips on how to develop and debug your code.
> - If you fail a test, look over the code for the test in `config.go` and `test_test.go` to get a better understanding what the test is testing. `config.go` also illustrates how the tester uses the Raft API.

## 需要修改的地方

```go
type LogEntry struct {
	Term    int
	Index   int
	Command interface{}
}
log的结构
```

```go
	applyCh   chan ApplyMsg //用于检测log是否成功？commit？收到？
	applyCond *sync.Cond
//raft中新添加这两个，用于commit和apply
```

make初始化raft

```go
//初始化时，提前append一个log（文中写的log index从1开始）
rf.log = append(rf.log, LogEntry{-1, 0, 0})
//除了ticker还要applier()
go rf.ticker()
go rf.applier()
```

Start函数是client输入command的入口

```go
//比较简单，leader收到cmd就添加一个log即可。不需要提前append，等心跳append
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if rf.state != LEADER {
		return -1, -1, false
	}
	term := rf.currentTerm
	index := rf.getLastLog().Index + 1 //第几个log，index从1开始
	log := LogEntry{
		Index:   index, //index 从1开始
		Term:    term,
		Command: command,
	}
	rf.log = append(rf.log, log)
	rf.persist()
	return index, term, true
}
```



leaderElection部分需要修改

```go
//当选leader后，需要更新nextIndex和matchIndex
becomeLeader.Do(func() { //保证多个协程，但只运行1次这里
			rf.state = LEADER
			lastLogIndex := rf.getLastLog().Index
			for i, _ := range rf.peers { //成为leader 初始化
				rf.nextIndex[i] = lastLogIndex + 1
				rf.matchIndex[i] = 0
			}
			rf.readyAppendEntries(true)
		})
```

RequestVote部分

```go
//投票处理
 - term比我小的不投
 - term比我大不一定投，但我一定变follow
 - term相等，并且log得比我新才能投
 如何定义log新
term越大越新
term相等比 lastLogindex，index大的新

type RequestVoteArgs struct {
	// Your data here (2A, 2B).
	Term         int
	CandidateId  int
	LastLogIndex int //自己最后一个日志号 ，告诉对方我比你新
	LastLogTerm  int //自己最后一个日志的term
}
```

此时leader的apendEntries需要修改

```go
//readyAppendEntries()
//发送时要带上Prelog 和commit，用于一致性检查和commit

//如何发送空包呢？？？？？？？
preLog := rf.log[nextIndex-1]
args := AppendEntriesArgs{
    Term:         rf.currentTerm,
    LeaderId:     rf.me,
    PreLogIndex:  preLog.Index, // 当前peer nextindex pre
    PreLogTerm:   preLog.Term,
    Entries:      make([]LogEntry, lastLog.Index-nextIndex+1), //发送多少个log呢？缺多少发多少
    LeaderCommit: rf.commitIndex,
}
copy(args.Entries, rf.log[nextIndex:])
```

收到返回的success处理

```go
//follower比我小的啥也不干
	if reply.Term == rf.currentTerm {
		//是否要commit当前的log
		if reply.Success {
			match := args.PreLogIndex + len(args.Entries)
			next := match + 1
			rf.nextIndex[serverId] = max(rf.nextIndex[serverId], next)
			rf.matchIndex[serverId] = max(rf.matchIndex[serverId], match)

		} else { // rollback
			rf.nextIndex[serverId]--
		}
		rf.leaderCommit() //每次收到一个success就判断一次是否提交
	}
```

**难点是，follower如何leader的RPC**

```go
代码太多了，需要仔细分析
```

最后说一下 commit和apply

```go
leader只能提交自己term内的log

leader收到RPC的返回会更新matchIndex
对每一个没commit的log，判断matchIndex中是否半数的follower都match了，过半就更新commitIndex，apply

leadercommit后，通过	rf.applyCond.Broadcast()，其实就是提交给client
```





## bug

- 每次都调用getlastlog，不要保存为变量再使用
- 初始化变量的时候要注意

```
*sync.Cond的使用
```

### follower如何追赶leader的日志

1. follow缓慢

   follow太慢了，leader不停的发送append，哪怕已经commit了，follwer还是得慢慢接受append一个一个追上来

2. follower宕机

   raft **一致性检查**生效，保证follower能按顺序恢复日志

   leader在发appendEntries RPC时，**会带上前一个日志的term和index**，此时follwer会返回失败，leader就发送前一个日志，以此类推。

   可能这个算法有点慢，但是可以优化

3. leader宕机

   leader上有未提交的日志，通过强制follower复制leader日志来解决日志不一致的问题



- leader不需要做什么处理，只需要发AppendRPC即可

- follower需要根据AppendRPC来一致性检查，就自动恢复了

- leader从来不会覆盖和删除自己的entry

常见问题：

什么是一致性检查

如果leader正常，那么日志顺序就正常，反之亦然

**安全性**：

- leader宕机：如何保证新leader中包含老leader的commit日志？

  > ```go
  > 
  > ```
  > 
  
- leader宕机：新leader是否需要提交之间term的log

raft的RPC都是幂等的

由于只有日志在被多数结点复制之后才会被提交并返回，所以如果一个 Candidate 并不拥有最新的已被复制的日志，那么他不可能获得多数票，从而保证了 Leader 一定具有所有已被多数拥有的日志（**Leader Completeness**），在后续同步时会将其同步给所有结点。





# 2C 

persistence

Raft集群重启后，应该能保持以前的状态，每次状态改变都要persist

> Complete the functions `persist()` and `readPersist()` in `raft.go` by adding code to save and restore persistent state. 
>
> You will need to encode (or "serialize") the state as an array of bytes in order to pass it to the `Persister`.
>
>  Use the `labgob` encoder; see the comments in `persist()` and `readPersist()`. `labgob` is like Go's `gob` encoder but prints error messages if you try to encode structures with lower-case field names. 
>
> For now, pass `nil` as the second argument to `persister.Save()`. Insert calls to `persist()` at the points where your implementation changes persistent state. Once you've done this, and if the rest of your implementation is correct, you should pass all of the 2C tests.

1. 完善两个persist函数

2. 在什么时候加persist()

	只要volatile变量变了就执行一次persist()函数

Figure 8一直无法通过

lastLog可能有错

Xterm没加可能有问题

# 2D

snapshot

```go
snapshot()
 InstallSnapshot RPC
```

一个好的开始是修改您的代码，以便它能够仅存储从某个索引 X 开始的日志部分。最初，您可以将 X 设置为零并运行 2B/2C 测试。然后让 Snapshot(index) 丢弃 index 之前的日志，并设置 X 等于 index。如果一切顺利，您现在应该可以通过第一个 2D 测试。
