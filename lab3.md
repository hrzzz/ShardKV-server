> 参考链接：
>
> [mit6.824 2022 lab3_最佳损友1020的博客-CSDN博客](https://blog.csdn.net/freedom1523646952/article/details/127996075?csdn_share_tail={"type":"blog","rType":"article","rId":"127996075","source":"freedom1523646952"})
>
> [MIT6.824-lab3AB-2022（万字推导思路及代码构建）_幸平xp的博客-CSDN博客](https://blog.csdn.net/weixin_45938441/article/details/125286772?spm=1001.2014.3001.5502)





还需要保证数据的一致性，也就是1条command不能被多次执行

> 实验二中raft已经基本实验，现在是在raft的基础上搭建一个kv存储系统



kv server中的操作都是先经过Raft保存后才会进行读或写，并且为Raft执行设一个计时器，超时则失败。

```go
//先进行raft保存log
	index, _, isleader := kv.rf.Start(op)
	if !isleader {
		reply.Err = ErrWrongLeader
		return
	}
	//管道1
	ch := kv.getWaitCh(index) //  为该op创建一个管道
	defer func() {
		kv.mu.Lock()
		delete(kv.waitChMap, op.Index)
		kv.mu.Unlock()
	}()

	//管道2
	// 设置超时ticker
	timer := time.NewTicker(100 * time.Millisecond)
	defer timer.Stop()
```



Raft收到command会执行Start函数，start中先本地持久化，然后会根据Raft自动分发给其他follower同步，当都commit后，就会执行`applier`函数把所有节点都同步完成的信息返回给kv，`applier`函数的触发是由`rf.applyCond`控制，`rf.applyCond`由`apply`函数调用。所以commit其实就是apply。leader和follower都可以apply，leader的apply信息会提交给上层，follower的也会提交给上层，同样follow的kv也会执行该操作。也就是说，kv执行的并不是client传过来的op，而是raft传过来的op。



raft通过`applyCh`传递给上层，`applyCh`又传递给`waitChMap`，`waitChMap`用于判断raft底层是否持久化成功。

> 
>
> sync.Cond 经常用在多个 Goroutine 等待，一个 Goroutine 通知（事件发生）的场景。如果是一个通知，一个等待，使用互斥锁或 channel 就能搞定了。
>
> 用mutex和channel也能实现，但是不方便。
>
> **实现同步操作**



底层raft和上层kv server通信是使用`applyCh`管道

Get和append操作都会被Raft保存为log



![image-20230618204103620](D:/md_pic/image-20230618204103620.png)



对于重复的op如何处理？

这个交给kv server层处理

# 实现Snapshot()函数

`maxraftstate`阈值

`applyMsgHandlerLoop`函数

底层raft可能返回给server2种消息，**一种是普通command**，server执行就好。**一种是快照消息**，从ApplyMsg中读取Snapshot信息。

每次执行1次command都要检查是否超过阈值，保存当前状态为快照。



> 保存快照的细节处理：
>
> - 当前Raft状态的size大于阈值
>
> - 且 kv.commitIndex > kv.lastSnapshotIndex。因为防止安装相同的镜像，新镜像一定要大于就镜像的最后提交
>
> - 由此又考虑了另外一个问题，快照只能减小由commit log带来的空间开销，如果log中的日志全部为未提交，那么再怎么生成快照也没用。所以存在一种可能，不断进入的client请求可以让raftstate超过maxraftstate（**难点**）
>
> 	解决：在快接近阈值的时候就生成镜像，并且让后来的log加入空间，假如空间都又被占满了就只能等待了，假如还断电了，那么就真的会丢失log，但这些log都还未提交，所以问题也不大。
>
> 	所以看之前有个建议非常好，每当节点被选举为leader，就向日志中加入一条空指令，这样就能最快地更新commit index。
>
> - 

每次收到raft底层的命令后，需要`maxraftstate`和raftstate的大小比较，raftstate保留了log，index，currentTerm等信息

其中还包括了日志压缩

# 常见问题

- 对于重复请求如何处理

- Snapshot如何实现

- follower如何安装快照？

	leader安装完快照后，应该发送一个RPC给follower？不需要，因为leader节点执行snapshot是因为到达阈值了，同理follower节点也会收到raft发来的执行命令，那么follower也会检查是否到达阈值，follower自己就执行了snapshot，不需要leader通知。

	`InstallSnapShot`，这个RPC是follower节点重启后需要恢复log，log得问leader要**（自己有快照，为啥不用自己的？，kvserver重启后会从Raft读取快照，所以不影响）**，此时leader 的log已经压缩成快照了，所以就调用这个RPC，并且发送ApplyMsy给上层，。

	follow挂了后，leader什么时候发送快照给follow？

	

![image-20230619153413052](D:/md_pic/image-20230619153413052.png)

生成Snapshot，由上层server调用

安装Snapshot，leader发送给follower快照，follower安装

