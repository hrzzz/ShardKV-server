# 运行

主要就是修改这三个文件

![image-20230306161649769](D:/md_pic/image-20230306161649769.png)

coordinate作为master，分配任务

worker作为worker，执行任务

rpc负责中间通信

> 问题：为什么要写进程模拟分布式
>
> 分布式计算完全没问题，应为是RPC通信。但是存储没有实现分布式的，所以先都放在单机上运行，主要是为了方便存储

> 参考博客
>
> [(2条消息) Mit6.824 lab1全解析（推导历程+代码）_ligen1112的博客-CSDN博客](https://blog.csdn.net/ligen1112/article/details/120931874)
>
> [(2条消息) MIT 6.824-lab1_mit6.824_东东儿的博客-CSDN博客](https://blog.csdn.net/qq_35102066/article/details/110678699?spm=1001.2101.3001.6650.14&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-14-110678699-blog-120931874.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-14-110678699-blog-120931874.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=21)

[GitHub - OneSizeFitsQuorum/MIT6.824-2021: 4 labs + 2 challenges + 4 docs](https://github.com/OneSizeFitsQuorum/MIT6.824-2021)

# 思路

启动一个coordinate，多个worker，rpc中定义通信的数据结构

coordinate统一**管理，分配，监控**所有任务的状态

- `GetTask`被worker调用，返回一个任务

- `ReportTask`函数提供给worker，当任务完成了，worker调用report

- `HandleTimeout`处理worker超时或者掉线，超时就标记未完成，重新分配

worker任务很简单，执行`Get`到的任务，完成后`report`即可，get和report都是RPC

> 问题：getTask和report函数，直接写在worker中不好吗，为啥一定要放在master上
>
> getTask中要检查所有任务的状态，是否执行完毕。worker无法知道还剩几个任务，以及任务的状态，也不应该让worker来管理，多个worker之间是不存在通信的，meta data应该由maser管理。假如getTask放在worker中，由于worker之间不存在通信，所以必定会造成资源冲突，同时获得同一个任务，而且不知道其他人完成的是哪个任务。
>
> report和getTask类似，report返回的是任务的type和state，如果写在worker中，你完成了别人不知道呀，所以需要在centra master中
>
> 简单来说master是worker之间间接通信的桥梁

RPC 只定义了master和worker之间的 **通信数据结构**

> 问题：为什么需要使用RPC，其他的HTTP不行吗
>
> 
>
> 

# 如何一步步完成

1. 首先要实现的是串行的map reduce任务，这个流程要清楚，然后再了解分布式

![image-20230308150800929](D:/md_pic/image-20230308150800929.png)

2. 再了解真正mapReduce的工作流程 论文：[mapreduce-osdi04.pdf (googleusercontent.com)](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)

   这个论文还没看，需要再看看

3. 细节如上图所示

# 代码细节

mutex：

> 什么地方需要加锁？
>
> - 分配任务时，多个worker同时调用GetTask，只能有一个访问，防止分配到同一个task**（需要）**
> - repor t时不加锁会出现 DATA RACE**（需要）**，race是什么？？
> - Done，判断master是否结束时也需要**（需要）**，能PASS否则RACE
> - HandleTimeout，不加可以PASS但是会RACE**（需要）**
>
> 一般来说，读不用，写需要，修改task状态，或者分配task的时候

常量的写法：

```go
// type,给worker
const (
	Map = iota
	Reduce
	Sleep
	Complete
)

// task status已经用于record
const (
	NotStarted = iota
	Processing
	Finished
	Timeout
)
```

通信数据结构：

一般很难一次写出满意的数据结构，可以慢慢改和添加，应该由更好的方式

```go

type Task struct { // 用来记录那个task的状态
	Id           int // 任务id,用于pool
	Type         int
	MFileName    string
	MReturnFiles []string //map返回的结果

	RID       int      //reduce Group
	RFilName  []string //reduce的输入
	ReduceNum int
	Status    int
}

// get task
type ReqArgs struct {
	X int
}
type RespArgs struct {
	Task Task //一定要大写
}

// report task
type ReportReqArgs struct {
	//Task Task //一定要大写
	FilesName []string //如果是map任务，需要告知master中间文件的信息
	TaskId    int      //该任务的名字
}
type ReportRespArgs struct {
	X int
}

```

map写入中间文件：

可以使用json方便

```go
	for _, kv := range intermediate { //将数据写入到对应的文件中。为了方便reduce读取，所以选择以json格式写入
		index := ihash(kv.Key) % task.ReduceNum
		enc := json.NewEncoder(outFiles[index])
		enc.Encode(&kv)
		//写入文件也可以使用其他方式
	}
```

# 难点

如何分配map和reduce任务

> 分别用两个map维护这种任务是否完成

如何处理crash或者timeOut

> 把分配出去的任务，放进pool中，单独开个10s的协程监督该任务，如果协程时间到了发现任务还在pool中，那么这个就是超时

任务完成如何report

> 当worker调用了report时，一定是完成了任务，所以report只需要更新taskList是否Finished，然后从pool中删除，重点就是防止重新分配出去。
>
> 而且还要接受到map返回过来的immediate file位置，以提供给reduce使用



# coding遇见的一些问题

结构体该大写的变量就要大写

getTask不停的分配任务，应该是report时，没有从pool中剔除

task默认任务应该给sleep

reduce这边要给个RID，也即是group id，要和task id区分开

主要是在master这边的两个函数

GlobeID

for遍历reduceRecord而不是遍历reduceNum

# test

测试了一些测试案例

- 提前worker退出
- worker crash

indexer wc等

# TODO

- 真正的mapreduce的啥样的，以及论文没看
- 使用channel实现，去mutex
- test-many还是会出错，运行crash的时候

## channel

构建2个record chan

什么时候关闭chan，report的时候而不是Get的时候

close后还能cap(chan)？

查看chan关闭，只能取一个吗

chan总会遇到各种小问题，可能是设计的不合理，mutex清晰明了

等chan再学学再来写吧。。。

可能crash master吗
