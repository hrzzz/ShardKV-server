> 参考链接
>
> [MIT 6.824 Lab4 AB 完成记录_mit6.824 lab4_多宝鱼1998的博客-CSDN博客](https://blog.csdn.net/qq_40443651/article/details/118034894)
>
> [MIT-6.824-lab4A-2022(万字讲解-代码构建）_6.824 lab_幸平xp的博客-CSDN博客](https://blog.csdn.net/weixin_45938441/article/details/125386091?spm=1001.2014.3001.5502)
>
> 



我们要创建一个 “**分布式的**，拥有**分片功能**的，能够**加入退出成员**的，能够**根据配置同步迁移数据**的，Key-Value数据库服务”。

# lab a

实现一个ShardCtrl Cluster（可以模仿lab3的kv系统实现，只不过这里维护的是一个config[]）

config[]中存的都是history config

```go
configs []Config // indexed by config num

type Config struct {
	Num    int              // config number
	Shards [NShards]int     // shard -> gid
	Groups map[int][]string // gid -> servers[]
}

```



command 4个（而不是pu，get操作）

```go
Join ：加入一个新的组 ，创建一个最新的配置，加入新的组后需要重新进行负载均衡。
Leave: 将给定的一个组进行撤离，并进行负载均衡。
Move: 为指定的分片，分配指定的组。
Query: 查询特定的配置版本，如果给定的版本号不存在切片下标中，则直接返回最新配置版本。

所以相应的，也需要实现这4个RPC
```

负载均衡算法：

```go
其实不难，就是生成 Shards [NShards]int ，     // shard -> gid
旧Group:[4,3,3],增加2个Group后，要改为[2,2,2,2,2]
返回值是一个 Shards [NShards]int ，说明哪个shard属于哪个group

```

**易错点，负载均衡时，不同raft节点的config不一致：**map遍历的不确定性，同一条命令在不同节点上运行时会产生不同的结果。

需要注意的是，这些命令会在一个 raft 组内的所有节点上执行，因此需要保证同一个命令在不同节点上计算出来的新配置一致，而 go 中 map 的遍历是不确定性的，因此需要稍微注意一下确保相同地命令在不同地节点上能够产生相同的配置。

**解决方法：**把key都先读出来，并且排序，最后按key处理就可



**易错点：**在处理传进来的参数时，需要deepcopy，因为是多线程编程，你不知道是否有其他的线程在使用该变量，而且对变量的修改很有可能会影响到其他地方产生bug。

**解决方法：**大多数时候deepcopy，或者在RPC的时候用reqArgs，和replyArgs



**易错点：**多线程编程如何debug。





![image-20230619163936827](D:/md_pic/image-20230619163936827.png)



# lab b

从lab a可以得到一个用raft维护的分片控制器

> 参考：
>
> [MIT-6.824-lab4B-2022(万字思路讲解-代码构建）_6.824 lab_幸平xp的博客-CSDN博客](https://blog.csdn.net/weixin_45938441/article/details/125566763?spm=1001.2014.3001.5502)
>
> [6.5840 Lab 4: Sharded Key/Value Service (mit.edu)](https://pdos.csail.mit.edu/6.824/labs/lab-shard.html)
>
> lab b是基于raft、shardctrler，2个项目的代码，所以前3个lab必须完全实现

## 讨论

我们可以首先明确系统的运行方式：一开始系统会创建一个 shardctrler 组来负责配置更新，分片分配等任务，接着系统会创建多个 raft 组来承载所有分片的读写任务。此外，raft 组增删，节点宕机，节点重启，网络分区等各种情况都可能会出现。

对于**集群内部**，我们需要保证所有分片能够较为**均匀的分配**在所有 raft 组上，还需要能够支持**动态迁移和容错**。

对于**集群外部**，我们需要向用户保证整个集群表现的像一个永远不会挂的单节点 KV 服务一样，即具有线性一致性。

lab4b 的基本测试要求了上述属性，challenge1 要求及时清理不再属于本分片的数据，challenge2 不仅要求分片迁移时不影响未迁移分片的读写服务，还要求不同地分片数据能够独立迁移，即如果一个配置导致当前 raft 组需要向其他两个 raft 组拉取数据时，即使一个被拉取数据的 raft 组全挂了，也不能导致另一个未挂的被拉取数据的 raft 组分片始终不能在当前 raft 组提供服务。

> 首先明确三点：
>
> - 所有涉及修改集群分片状态的操作都应该通过 raft 日志的方式去提交，这样才可以保证同一 raft 组内的所有分片数据和状态一致。
> - 在 6.824 的框架下，涉及状态的操作都需要 leader 去执行才能保持正确性，否则需要添加一些额外的同步措施，而这显然不是 6.824 所推荐的。**因此配置更新，分片迁移，分片清理和空日志检测等逻辑都只能由 leader 去检测并执行。**
> - 数据迁移的实现为 pull 还是 push？其实都可以，个人感觉难度差不多，这里实现成了 pull 的方式。



下图：shardctrler也是一个raft集群控制config版本

shard是一个抽象概念，可以想象成data。



![image-20230621203013139](D:/md_pic/image-20230621203013139.png)

## client

```go

type Clerk struct {
	sm       *shardctrler.Clerk
	config   shardctrler.Config
	make_end func(string) *labrpc.ClientEnd
	// You will have to modify this struct.
	seqId    int
	clientId int64
}
func (ck *Clerk) Get(key string) string {
func (ck *Clerk) PutAppend(key string, value string, op string) {
// put 和get 都是RPC掉调用shardkv.Get or put
// clerk对key进行key2shard 哈希
    //再用config和shard得到Gid
    //通过Gid就得到servers
    //servers中就有Leader，这样就可以向leader发送RPC请求了
    

```

## server

```go
type ShardKV struct {
	mu           sync.Mutex
	me           int
	rf           *raft.Raft
	applyCh      chan raft.ApplyMsg
	makeEnd      func(string) *labrpc.ClientEnd
	gid          int
	masters      []*labrpc.ClientEnd
	maxRaftState int // snapshot if log grows this big

	// Your definitions here.

	dead int32 // set by Kill()

	Config     shardctrler.Config // 需要更新的最新的配置
	LastConfig shardctrler.Config // 更新之前的配置，用于比对是否全部更新完了

	shardsPersist []Shard // ShardId -> Shard 如果KvMap == nil则说明当前的数据不归当前分片管（可能保存多个片）,保存数据的地方
	waitChMap     map[int]chan OpReply
	SeqMap        map[int64]int
	sck           *shardctrler.Clerk // sck is a client used to contact shard master
}

```

## loop

loop

就是另一个协程，一直在接受raft发来的信息

```go
// applyMsgHandlerLoop 处理applyCh发送过来的ApplyMsg
func (kv *ShardKV) applyMsgHandlerLoop() 
func (kv *ShardKV) ConfigDetectedLoop()

```

## 如何更新config

shardctrler更新完config后，server查询得到最新的config，如何更新本地的database呢

<img src="D:/md_pic/image-20230624155258898.png" alt="image-20230624155258898" style="zoom:50%;" />

更新过程如下，也就是集群的shard迁移如下：

- 更新配置后发现有配置更新：
- 1、那么则通过不属于自己的切片
- 2、收到别的group的切片后进行AddShards同步更新组内的切片
- 3、成功发送不属于自己的切片或者超时则进行Gc

- tips:这里同样是会涉及到一个**去重**的问题，相比客户端RPC通过在client端进行seqId自增，关于的配置的自增，只要利用配置号进行就可以，只要配置更新，那么一系列的操作就都会与**最新的配置号有关**。

![image-20230624155419116](D:/md_pic/image-20230624155419116.png)