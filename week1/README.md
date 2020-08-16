# 高性能TiDB课程系列 - week 1
博客原文请见 https://elon.fun/posts/hptidb-hello-transaction

本文记录做[《高性能TiDB课程系列》week 1 - TiDB 总体架构](https://docs.qq.com/sheet/DSlBwS3VCb01kTnZw?tab=BB08J2) 作业的过程。由于之前学习过 SparkSQL 源码，所以对 SQL 解析、优化、执行的流程略有了解，但是这次学习 TiDB 的源码仍然感觉非常有趣，会坚持把这个课程做完🤘。

## 题目描述
> 本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：
> * 1 TiDB
> * 1 PD
> * 3 TiKV
> 改写后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的日志

> 输出：一篇文章介绍以上过程


## 编译源代码
对于 TiDB 这么成熟的开源项目，编译源代码应该是一件非常轻松的事情。事实证明，TiDB 的官方文档和博客写得挺全，编译过程除了网络问题外，一切都还是比较容易的。

### TL;DR
为了让编译步骤容易复现，我把所有步骤都用 [Dockerfile](https://github.com/dragonly/hp-tidb-course/blob/master/week1/run/Dockerfile) 记录下来，只需要在 [hp-tidb-course](https://github.com/dragonly/hp-tidb-course) 项目的 `week1` 目录下执行 `./run/run.sh` 即可完成所有编译流程。为了简单，并未使用 docker-compose。

**以下所有步骤均可参考上述 Dockerfile。**

### 编译 PD
PD 用于管理和调度 TiKV 集群，编译方式非常简单，安装好 golang 以后，按照[官方文档](https://github.com/pingcap/pd)执行 make 即可编译。编译完成后，二进制文件位于项目根目录的 `bin/` 目录中。

### 编译 TiKV
TiKV 的编译有一点意思，在 [CONTRIBUTING] (https://github.com/tikv/tikv/blob/master/CONTRIBUTING.md)文档中说明了编译依赖于操作方式，但是由于网络问题，我无法安装 rustup。最终通过[《使用国内镜像加速 Rust 更新与下载》](https://www.chainnews.com/articles/804852093639.htm)中的方式替换镜像地址成功下载。

编译结束后，我一度找不到二进制文件地址，后来根据 [cargo 文档](https://doc.rust-lang.org/cargo/commands/cargo-build.html) 找到，`--target-dir` 选型默认在项目根目录的 `target/debug/` 目录中。

### 编译 TiDB
TiDB 用于处理用户请求，生成和优化 SQL 执行计划，并调度 TiKV 执行 SQL。TiDB 的编译类似 PD，装好 golang 后照着[官方文档](https://github.com/pingcap/community/blob/master/contributors/README.md#tidb)执行 make 即可编译。吐槽一点，在 TiDB 的 [github 主页](https://github.com/pingcap/tidb)并没有找到编译方法🙄。

## 部署
参考 [TiKV 官方部署二进制的文档](https://github.com/tikv/tikv/blob/master/docs/how-to/deploy/using-binary.md)，执行以下命令启动 PD\*1 与 TiKV\*3 实例。
```bash
# start pd, 1 instance
./pd-server \\
  --name=pd1 \\
  --data-dir=pd1 \\
  --client-urls="http://127.0.0.1:2379" \\
  --peer-urls="http://127.0.0.1:2380" \\
  --initial-cluster="pd1=http://127.0.0.1:2380" \\
  --log-file=pd1.log &
# start tikv, 3 instances
./tikv-server \\
  --pd-endpoints="127.0.0.1:2379" \\
  --addr="127.0.0.1:20160" \\
  --data-dir=tikv1 \\
  --log-file=tikv1.log &
./tikv-server \\
  --pd-endpoints="127.0.0.1:2379" \\
  --addr="127.0.0.1:20161" \\
  --data-dir=tikv2 \\
  --log-file=tikv2.log &
./tikv-server \\
  --pd-endpoints="127.0.0.1:2379" \\
  --addr="127.0.0.1:20162" \\
  --data-dir=tikv3 \\
  --log-file=tikv3.log &
```

可以使用以下命令验证 TiKV 和 PD 部署成功
```bash
./pd-ctl store -u http://127.0.0.1:2379
```

正常应该返回如下 JSON，表示 3 个 TiKV 实例都正常与 PD 建立连接
```bash
{
  "count": 3,
  "stores": [
    {
      "store": {
        ...
        "state_name": "Up"
      },
      "status": {...}
    },
    ...
  ]
}
```

至于 TiDB，直接按照[官方文档](https://docs.pingcap.com/zh/tidb/stable/command-line-flags-for-tidb-configuration)，使用如下命令启动
```bash
./tidb-server \\
  --store=tikv \\
  --path="127.0.0.1:2379" \\
  --log-file=tidb1.log &
```

看到 tidb1.log 文件中输出
```bash
["server is running MySQL protocol"]
```

代表所有部署大功告成🎉！

## 修改源码，打印事务开始
<p>
搞了一通终于可以开始干正事了，下面进入艰难的源码阅读理解环节
<span style={{ fontSize: '2em' }}>🤯</span>
。
</p>

这个过程非常曲折，一开始我试图直接从 `conn.go` 往里面走，找了好几个疑似开启事务的地方，打印日志重新编译无果，令人沮丧。后来我突然明白，这么巨大的代码库，加上其中蕴含的复杂设计，怎么能让我一下找到呢，于是我开始在网上找解读 TiDB 分布式事务源码部分的文章和视频，逐渐有了些思路。

首先从[官方文档 - TiDB 事务概览](https://docs.pingcap.com/zh/tidb/stable/transaction-overview)先稍微了解下 TiDB 中的“事务”到底是怎么一回事：分为乐观与悲观事务，可以由 `BEGIN` 等 SQL 语句显示开启；或者由默认的 autocommit 模式隐式控制，即每条 SQL 语句运行后，TiDB 会自动将修改提交到数据库中。另外 PingCAP 博客中的[#TiDB-源码阅读 系列](https://pingcap.com/blog-cn/#TiDB-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB)也很不错地介绍了 TiDB 源码的整体结构，非常值得一看。关于事务的部分，我根据[TiDB 新特性漫谈：悲观事务](https://pingcap.com/blog-cn/pessimistic-transaction-the-new-features-of-tidb/)这篇文章了解了大概，不就是基于 percolator 改造的两种 2pc 实现乐观/悲观锁嘛，代码走起👨‍💻~

基于 [TiDB 源码阅读系列文章（三）SQL 的一生](https://pingcap.com/blog-cn/tidb-source-code-reading-3/)走了一遍 SQL 处理流程，我感觉核心的事务数据结构入口在 [session/txn.go:43](https://github.com/pingcap/tidb/blob/c5c7bf87bb/session/txn.go#L43)处，摘抄一下附近注释：
```go
// TxnState wraps kv.Transaction to provide a new kv.Transaction.
// 1. It holds all statement related modification in the buffer before flush to the txn,
// so if execute statement meets error, the txn won't be made dirty.
// 2. It's a lazy transaction, that means it's a txnFuture before StartTS() is really need.
type TxnState struct {
	// States of a TxnState should be one of the followings:
	// Invalid: kv.Transaction == nil && txnFuture == nil
	// Pending: kv.Transaction == nil && txnFuture != nil
	// Valid:	kv.Transaction != nil && txnFuture == nil
```

其实一开始这段注释对我而言没有太多信息量，在我认真翻看 PingCAP 博客系列文章后发现，才基本就理解这里啥意思。先贴一张从 [TiKV 源码解析系列文章（十二）分布式事务](https://pingcap.com/blog-cn/tikv-source-code-reading-12/)一文中借来的图，说明一下一次分布式事务大致流程。

![image](../../static/hptidb-hello-transaction/transaction-flow.png)
[<center><small>from https://pingcap.com/blog-cn/tikv-source-code-reading-12/</small></center>](https://pingcap.com/blog-cn/tikv-source-code-reading-12/#%E4%BA%8B%E5%8A%A1%E7%9A%84%E6%B5%81%E7%A8%8B)

其实 TiDB 中的分布式事务主要由 TiKV 中基于 percolator 模型实现，所以上面这段注释现在再看一下，有几个点：

1. `TxnState` 是用来封装 kv 中的分布式事务的，这里之所以没写 TiKV 是因为只要实现相关接口都可以替换在这里做这个 kv
2. 这个实现是个 lazy 实现，一开始只是记录了一个 `txnFuture` 对象，用于在需要的时候利用 Timestamp Oracle 获取 percolator 模型中需要的单调递增时间戳。具体为什么是 lazy 应该是为了方便取消，我其实也不太清楚什么情况会从中得到好处😂。
3. `TxnState` 有三种状态：invalid、pending、valid，具体对应上述三种 `kv.Transaction` 和 `txnFuture` 的组合，有了这个知识就比较好理解代码中的 `validOrPending()` 之类的函数在干嘛。

这样顺着往 kv 里面看，最后找到了 [`store/tikv/txn.go:newTikvTxnWithStartTS()`](https://github.com/pingcap/tidb/blob/c5c7bf87bb/store/tikv/txn.go#L96) 函数，签名为
```go
func newTikvTxnWithStartTS(store *tikvStore, startTS uint64, replicaReadSeed uint32) (*tikvTxn, error)
```

表示在这里使用获取到的 timestamp 开启一个新的 TiKV 事务。虽然从逻辑上讲，这里获取到 timestamp 代表已经开启了事务，但是又没法说服自己获取 timestamp 的地方一定代表事务开始，于是就在这个函数开头加了一行
```go
logutil.BgLogger().Info("hello transaction")
```

重新编译运行，`tail -f tidb1.log` 发现，每一秒出现两次 `hello transaction`，每三秒多出现 4 次。打开 debug 级别日志发现，前者是 `ddl_worker` 每秒触发打印一次 `[ddl] wait to check DDL status again`，后者暂时没找到。

然后就比较好玩了，可以构造各种事务去看日志打印，过滤掉上述固定切规律的事务日志，就是额外触发的事务了。比如先创建一个 temp 表
```sql
CREATE TABLE temp (i int);
```

此时除了 `hello transaction` 外，还会打印如 `CRUCIAL OPERATION` 之类的日志，代表这是一个重要操作，毕竟新增了一张表。

再测试一个比较好玩的事情，连续两次提交 `BEGIN` 命令，发现第二次提交后，会打印日志
```bash
"NewTxn() inside a transaction auto commit"
```
这是因为第二次 `BEGIN` 会先结束掉前一次开启的事务。

这次就到这里了，不一定理解的正确，待再多学习 TiDB 后再说吧😉。
