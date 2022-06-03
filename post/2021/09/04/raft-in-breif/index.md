# Raft in breif


## 概览

{{<image src="https://img13.360buyimg.com/ddimg/jfs/t1/156556/6/21896/461723/61323085E123fac2c/b209f3cc162ba3de.png" width=100% caption="算法实现概要" >}}

## 安全性保证

Raft保证以下五条准则在算法运行期间始终成立(Invariance):

- `Election Safety`: 一个世代`term`内至多产生一个leader.
- `Leader Append-Only`: leader在自己的日志中只会追加新Entry而不会去修改或删除某一项Entry.
- `Log Matching`: 若俩日志中某Entry有着相同的index和term, 则该Entry之前的日志项都相同.
- `Leader Completeness`: 若一个Entry在某世代term内已committed, 则此后任何term内该Entry都存在.
- `State Machine Safety`: 若状态机应用了某Entry, 则该index处不应再应用其他Entry.

## Leader选举

{{<image src="https://img10.360buyimg.com/ddimg/jfs/t1/44230/18/16643/87041/6132f98dEe6502809/d5cbf26072e83dbe.png" width=100% caption="节点角色状态转移" >}}

选举`Leader`思想很简单: 某个节点开始竞选某个世代`term`的`leader`, 向其他节点发送`RequestVote RPC`, 如果他得到的支持票数满足[quorum](https://en.wikipedia.org/wiki/Quorum_(distributed_computing))机制, 则成为该`term`内合法的`leader`. 这里面的关键问题是如何防止出现`split brain`问题(确保每个`term`内仅会出现一个`leader`).

首先, 对于一个给定的`Term`, 服务器只会进行一次投票(且只会投给`Term`更高的), 且为了避免出现过多的竞选者, 各节点的触发选举超时时间应随机化. 此外, `RequestVoteRPC`里还应带上候选者的日志信息, 避免选出落后的`Leader`而导致日志覆盖(违背线性化).

## 日志复制

要点:

1. 大部分节点(`quorum`)已知晓的`Entry`的状态即为已`committed`, Leader会将其应用到状态机（持久化），返回客户端执行结果.

2. Leader在下一次的`AppendEntries RPC`中带上`commitedIndex`，这样followers就知道自己相应的日志项也该应用到状态机了.

3. Leader在`AppendEntries RPC`中带上新`Entry`之前的`Entry`信息，follower若检查到自己没有该`Entry`的话，就会知道自己之前的复制可能出了差错并拒绝掉该`Entry`. 只要follower的结果返回了成功, 我们就知道该follower与leader至少在新`Entry`之前是保持一致了的.

4. Leader宕机带来的不一致问题(复制以Leader为准则): Leader通过`AppendEntries RPC`不断来回的试探到follower与之匹配的日志记录, 然后将其覆盖重写为与自己一致的项.

5. 新Leader产生后, 可以先复制一个`no-op Command`的`Entry`从而更快的获得各follower的复制进度. 对前任的遗留日志需通过提交自己任期的日志去进行间接提交. 原因如下:

{{<image src="https://img14.360buyimg.com/ddimg/jfs/t1/62224/21/16485/388062/61388b2cEb7a2d9cd/660a2b352057874b.jpg" width=100% caption="提交前任遗留日志">}}

- `a`: 此时$S_1$成为`term 2`的leader并产生了Entry, 只复制到了$S_2$后就crash了;
- `b`: $S_5$成为`term 3`的leader并产生了Entry, 但还没复制就crash了;
- `c`: $S_1$成为`term 4`的leader并产生了Entry, 他先复制前任的遗留Entry到了大多数节点, 但还没持久化提交就又crash了;
- `d`: $S_5$成为新term的leader, 未产生Entry, 接着它复制前任遗留的Entry, 并导致了日志重写.

可以看到, 由于策略是上来就提交前任的遗留Entry, 导致了日志被重写. 正确做法的结果应如`e`:` c`时刻$S_1$成为leader后, 等待到提交自己任期的Entry, 前任的遗留日志自然会被复制从而被间接提交. 且此后$S_5$由于落后的日志也不会再成功竞选.

## 成员(配置)变更

变更配置时, 可不是简单的直接从旧的配置换到新的上去就行(换配置然后重启). 为保证系统可用性和安全性, 应使用`2-phase`: 

1. 第一步: 禁止旧配置的系统继续接受新的请求 
2. 第二步: 切换到新的配置文件上 

在新旧配置转换过渡期间会同时存在两种配置, 这期间系统可能会产生`split brain`. Raft采用的`2-phase`机制叫`joint consensus`: 其使得即使在过渡期间系统仍能正确的, 安全的运行: 如在$C_{old}$到$C_{new}$的过渡期: $C_{old, new}$ , 为防止`split brain`, 决议要同时通过两个配置的joint集合的quorum通过才行.

思考如下的配置拓扑:

{{< image src="https://img13.360buyimg.com/ddimg/jfs/t1/198113/20/6506/14514/6132338aE54742997/4057840d076c07ef.png" width=100% caption="3区域配置拓扑" >}}

如上图, 3个region(X, Y, Z)中配置$C_1$($X_2$,  $Y_1$,  $Z_1$) 组成的raft集群($Y_1$是leader)现在想要切换到配置$C_2$(新增$X_1$, 移除$X_2$). 先看看直接切换: 

{{<image src="https://img12.360buyimg.com/ddimg/jfs/t1/194427/29/21560/157836/61322987Ec0ee8b82/5780f7b71bf82e64.gif" width=100% caption="直接配置切换" >}}

可以看到, $Y_1$收到confchange后并切换到$C_2$, 与$X_1$组成了$C_2$的quorum, 与此同时, $X_2$与$Z_2$也组成了$C_1$的quorum, 这就导致了`split brain`. 而其中的原因是我们太过贪心, 将$X_1$的加入和$X_2$的移除同时进行了.

所以接下来我们先增加$X_1$, 再移除$X_2$(1 by 1, 让大部分节点知晓到第一次变更后再处理下一条). 则先$C_2$ = ($X_1$, $X_2$, $Y_1$, $Z_2$), 其quorum为3, 此时先变更的$X_1$与$Y_1$一起并无法做出决议. 但这样就真的完美解决了我们的问题吗?

若这时region X整个不可达时, 整个集群就得等待其重新上线才能恢复服务. 这时就引出了`joint consensus`:

针对上例, 我们在进行配置变更时, 引入过渡期配置$C_{1,2}$ = $C_1$ && $C_2$ = ($X_2$, $Y_1$, $Z_2$) && ($X_1$, $Y_1$, $Z_2$) , 过渡期间任何决议都得同时通过两个配置中的quorum通过才行:

{{<image src="https://img13.360buyimg.com/ddimg/jfs/t1/198734/35/6469/21828/61323eb6Ea8789b67/a17116a5928e124d.png" width=100% caption="Joint consensus">}}

这样, 即使X region整个失联, 集群仍是能够正常的运行.   

使用`joint consensus`进行成员配置变更的大致流程如下:

{{<image src="https://img11.360buyimg.com/ddimg/jfs/t1/47295/15/16793/18627/6132e9aaE34afee39/1d095425383d34f5.png" width=100% caption="配置变更流程" >}}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           

其中, 需要注意的问题是如果leader发生crash了会怎样? 由于有`Leader completeness`的保证, 在$C_{1,2}$ committed后, crash后新的leader的配置也一定会是$C_{1,2}$. 还有一种情况: $C_2$ committed后, leader可能并不在此配置中, 此时leader自己降级为follower, 进行$C_2$配置内的选举. 

此外, Raft还针对新增的成员引入了一个新的角色: learner(可以看看这篇文章: [etcd learner design](https://etcd.io/docs/v3.5/learning/design-learner/)). 在节点加入集群初期, 由于日志复制状态通常会远远落后于leader, learner在日志追赶未完成前不会参与决议投票. 而对于已移除(不在新配置中)但未下线的节点, 其在心跳超时后会发起`RequestVote RPC`反复干扰现有配置的集群, 避免的方式为只要节点认为现有leader还在(最小选举超时机制),就不理会新的`RequestVote RPC`. 

## 日志压缩

我们不能允许日志记录无休止的增长, 必须对其进行压缩以使获得更小的空间占用和更高效的日志复制:

通过`check point`处的快照(Snapshot)记录保留committed entries的最新状态, 旧的日志条目就可以删除掉.

{{<image src="https://img10.360buyimg.com/ddimg/jfs/t1/197044/17/6504/34435/6132f98dE5f9ae71e/c369de6313fbb493.png" width=100% caption="Snapshot" >}}

参考Etcd raft库的设计: 该库将raftLog分为`storage(stable)`和`unstable`.

`Snapshot`定义:

```protobuf
message SnapshotMetadata {
	optional ConfState conf_state = 1 [(gogoproto.nullable) = false];
	optional uint64    index      = 2 [(gogoproto.nullable) = false];
	optional uint64    term       = 3 [(gogoproto.nullable) = false];
}

message Snapshot {
	optional bytes            data     = 1;
	optional SnapshotMetadata metadata = 2 [(gogoproto.nullable) = false];
}
```

`storage.go`:

```go
type Storage interface {
	// TODO(tbg): split this into two interfaces, LogStorage and StateStorage.
	// InitialState returns the saved HardState and ConfState information.
	InitialState() (pb.HardState, pb.ConfState, error)
	// Entries returns a slice of log entries in the range [lo,hi).
	// MaxSize limits the total size of the log entries returned, but
	// Entries returns at least one entry if any.
	Entries(lo, hi, maxSize uint64) ([]pb.Entry, error)
	// Term returns the term of entry i, which must be in the range
	// [FirstIndex()-1, LastIndex()]. The term of the entry before
	// FirstIndex is retained for matching purposes even though the
	// rest of that entry may not be available.
	Term(i uint64) (uint64, error)
	// LastIndex returns the index of the last entry in the log.
	LastIndex() (uint64, error)
	// FirstIndex returns the index of the first log entry that is
	// possibly available via Entries (older entries have been incorporated
	// into the latest Snapshot; if storage only contains the dummy entry the
	// first log entry is not available).
	FirstIndex() (uint64, error)
	// Snapshot returns the most recent snapshot.
	// If snapshot is temporarily unavailable, it should return ErrSnapshotTemporarilyUnavailable,
	// so raft state machine could know that Storage needs some time to prepare
	// snapshot and call Snapshot later.
	Snapshot() (pb.Snapshot, error)
}

// MemoryStorage implements the Storage interface backed by an
// in-memory array.
type MemoryStorage struct {
	// Protects access to all fields. Most methods of MemoryStorage are
	// run on the raft goroutine, but Append() is run on an application
	// goroutine.
	sync.Mutex
	hardState pb.HardState
	snapshot  pb.Snapshot
	// ents[i] has raft log position i+snapshot.Metadata.Index
	ents []pb.Entry
}
```

`log.go`:

```go
type raftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage
	// unstable contains all unstable entries and snapshot.
	// they will be saved into storage.
	unstable unstable
	// committed is the highest log position that is known to be in
	// stable storage on a quorum of nodes.
	committed uint64
	// applied is the highest log position that the application has
	// been instructed to apply to its state machine.
	// Invariant: applied <= committed
	applied uint64
	logger Logger
	// maxNextEntsSize is the maximum number aggregate byte size of the messages
	// returned from calls to nextEnts.
	maxNextEntsSize uint64
}
```

`log_unstable.go`:

```go
// unstable.entries[i] has raft log position i+unstable.offset.
// Note that unstable.offset may be less than the highest log
// position in storage; this means that the next write to storage
// might need to truncate the log before persisting unstable.entries.
type unstable struct {
	// the incoming unstable snapshot, if any.
	snapshot *pb.Snapshot
	// all entries that have not yet been written to storage.
	entries []pb.Entry
	offset  uint64
	logger Logger
}
```

用一张图来直观的展示:

{{<image src="https://img13.360buyimg.com/ddimg/jfs/t1/199141/28/10401/59336/61527466E14991b7b/f743750a346e0b7e.png" width=100% caption="Etcd raft日志结构" >}}

## See Also

Raft Paper: [https://raft.github.io/raft.pdf](https://raft.github.io/raft.pdf)

Etcd Raft lib: [https://github.com/etcd-io/etcd/tree/main/raft](https://github.com/etcd-io/etcd/tree/main/raft)

Availability and Region Failure: Joint Consensus in CockroachDB: [https://www.cockroachlabs.com/blog/joint-consensus-raft](https://www.cockroachlabs.com/blog/joint-consensus-raft/)

