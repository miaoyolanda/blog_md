---
title: "跟 etcd 学习数据库中事务隔离的实现"
date: 2019-08-29T10:17:24+08:00
lastmod: 2020-01-03T22:56:24+08:00
draft: false
keywords: ["etcd", "database", "transaction", "isolation level", "mvcc", "cas", "乐观锁", "数据库", "事务", "多版本并发控制", "并发"]
---

众所周知，etcd 的数据模型是建立在 MVCC 基础上的（如果你不知，那一定是没看过我的[这篇博客]({{< ref "mvcc-implementation-in-etcd.md" >}})🙃）。在当前的实现中[^etcd-v3.3.10]，etcd 不仅提供了原子性的事务操作，还对外暴露了实现这些原子操作的 MVCC 版本信息。要知道，在大部分使用了 MVCC 机制的数据库中，你很难找到底层 MVCC 的影子，因为上层的数据库都把它屏蔽掉了，用户只能使用数据库封装好的一些能力。而在 etcd 中，最底层的版本信息是公开的，这就给我们提供了很多种可能性。比如在官方的客户端 clientv3 中，有一个 [concurrency](https://github.com/etcd-io/etcd/tree/v3.3.10/clientv3/concurrency) 包，这里面封装好的能力就足以覆盖 etcd 常见的几种使用场景：

* [mutex.go](https://github.com/etcd-io/etcd/blob/v3.3.10/clientv3/concurrency/mutex.go) 一个分布式锁的实现
* [election.go](https://github.com/etcd-io/etcd/blob/v3.3.10/clientv3/concurrency/election.go) 一个分布式选举的实现，多用来实现 Active-Standby 的高可用模式
* [stm.go](https://github.com/etcd-io/etcd/blob/v3.3.10/clientv3/concurrency/stm.go) 一个在**客户端**实现的事务框架，能将多条业务语句打包成一个原子操作

其中的 stm ([software transactional memory](https://en.wikipedia.org/wiki/Software_transactional_memory)) 实现的最有意思，它将原子性的事务实现得有模有样，还允许指定不同的隔离级别，就像在使用那些重量级的数据库一样。这让人不禁好奇，在拥有 MVCC 原语的基础上，它是怎样用 380 行代码实现出重量级的隔离效果的？

[^etcd-v3.3.10]: [etcd v3.3.10](https://github.com/etcd-io/etcd/tree/v3.3.10)

## 预备知识

在进入代码细节之前，我们先要了解常见的几种隔离级别是什么，以及 etcd 暴露出来的版本信息有哪些？

### 事务隔离级别

数据库的几种[事务隔离级别](https://zh.wikipedia.org/zh-cn/事务隔离) （Transaction Isolation Levels）一直是面试的热点，几乎逢面必问（不要问我怎么知道的）。当然这种理论知识稍微理解一下也不难记住：

* 未提交读（Read Uncommitted）：能够读取到其他事务中还未提交的数据，这可能会导致脏读的问题。
* 读已提交（Read Committed）：只能读取到已经提交的数据，即别的事务一提交，当前事务就能读取到被修改的数据，这可能导致不可重复读的问题。
* 可重复读（Repeated Read）：一个事务中，同一个读操作在事务的任意时刻都能得到同样的结果，其他事务的提交操作对本事务不会产生影响。
* 串行化（Serializable）：串行化的执行可能冲突的事务，即一个事务会阻塞其他事务。它通过牺牲并发能力来换取数据的安全，属于最高的隔离级别。

### etcd 中的版本

MVCC 全称是多版本并发控制，它保留每一次的修改，并用版本号来追踪这些修改。所以相应的 etcd 里面也有很多版本的概念，我们先来理清楚它们之间的区别[^etcd-revisions]：

* `Revision`是一个全局的状态，每一次有更改操作（Put, Delete, Txn）时，Revision 的值都会增加。它的作用有点像一个逻辑时钟，全局递增不重复。我们可以通过 [Response.Header.Revision](https://github.com/etcd-io/etcd/blob/v3.3.10/etcdserver/etcdserverpb/rpc.pb.go#L223) 获取到当前的值。
* `CreateRevision`仅作用于单个 key，它记录的是这个 key 被创建时的全局时钟（Revision）。一般通过 [KeyValue.CreateRevision](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/mvccpb/kv.pb.go#L64) 来获取。
* `ModRevision`也是单个 key 才有的概念，它表示这个 key 上一次被更改时，全局的版本是多少。也是通过 [KeyValue.ModRevision ](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/mvccpb/kv.pb.go#L66)获取到。
* `Version`仅仅是个计数器，与全局的 Revision 无关，代表了当前的 key 从创建到现在被更改了多少次。

[^etcd-revisions]: [What is different between Revision, ModRevision and Version?](https://github.com/etcd-io/etcd/issues/6518)

这里的概念有点像 Linux 文件系统里的各种时间。整个系统的时钟是一直往前走的（Revision），我们在08:01创建了一个文件`my.file`（CreateRevision），08:07分又更改了它（ModRevision）。系统会记录下这几个时间点：

```bash
$ stat my.file
...
Access: 2019-08-29 08:01:43.345815131 +0800
Modify: 2019-08-29 09:07:46.461380645 +0800
Change: 2019-08-29 09:07:46.461380645 +0800
 Birth: -
```

有了这个 Revision 之后，我们就像有了时光机一样，可以获取到系统在过去某个时间点的状态。比如`GET`函数就支持传入一个`WithRev(revision)`参数，来得到指定的 key 在指定的 revision 时的值。

### etcd 中的微事务

etcd v3 中引入了一个微事务的概念（mini-transaction），允许用户在一次修改中批量执行多个操作，这意味着这一组操作被绑定成一个原子操作，并共享同一个修订号。它的写法有点类似一个 [CAS](https://zh.wikipedia.org/wiki/比较并交换) ：

```go
Txn().If(cond1, cond2, …).Then(op1, op2, …).Else(op1’, op2’, …).commit()
```

如果`If`语句中的条件全部为真，则整个事务的返回值为`true`，并且`Then`中的操作会被执行，否则返回`false`并执行`Else`中的操作。

## 使用姿势

在并发编程领域，我们经常用银行转账的例子来说明原子操作的重要性。为了确保并发安全，要么使用一个悲观锁来避免冲突，要么就用乐观锁来检测冲突并重试。现在在 etcd 的场景下，有了`mini-transaction`和`ModRevision`的加持，我们很容易就想到用它们来实现一把乐观锁：

![cas transfer money](/image/2019/08/etcd-transaction-isolation-levels/cas-transfer-money.svg)

根据这个思路写出来的代码：

```go
func txnXfer(etcd *v3.Client, from, to string, amount uint) error {
    for {
        if ok, err := doTxnXfer(etcd, from, to, amount); err != nil {
            return err // 发生了错误，则直接给用户返回错误
        } else if ok {
            return nil  // 转账成功就返回
        }
        // 发生冲突了，需要重试
    }
}

func doTxnXfer(etcd *v3.Client, from, to string, amount uint) (bool, error) {
    // 利用 Txn 的原子性同时获取A和B的余额以及ModRevision
    getResp, err := etcd.Txn(etcd.Ctx()).Then(OpGet(from), OpGet(to)).Commit()
    if err != nil {
         return false, err
    }
    // 读取在当前时刻A和B的值
    fromKV := getResp.Responses[0].GetRangeResponse().Kvs[0]
    toKV := getResp.Responses[1].GetRangeResponse().Kvs[1]
    fromValue, toValue := toUInt64(fromKV.Value), toUint64(toKV.Value)
    if fromValue < amount {
        return false, fmt.Errorf(“insufficient value”)
    }
    // 再准备一个事务，其中的比较部分是查看 ModRevision 有没有变化
    txn := etcd.Txn(etcd.Ctx()).If(
        v3.Compare(v3.ModRevision(from), "=", fromKV.ModRevision),
        v3.Compare(v3.ModRevision(to), "=", toKV.ModRevision))
    // 如果从上次读取到现在还没有别人更改过，那就写入处理后的值
    txn = txn.Then(
        OpPut(from, fromUint64(fromValue - amount)),
        OpPut(to, fromUint64(toValue - amount))
    // 提交第二个事务
    putResp, err := txn.Commit()
    if err != nil {
        return false, err
    }
    // 如果事务的比较部分为真，putResp.Succeeded就是true，否则为false
    return putResp.Succeeded, nil
}
```

可以看到，这部分代码除了转账的业务逻辑外，大部分都是实现乐观锁的样板代码，包括两次事务操作、冲突检测、冲突发生时的重试等，写起来比较繁琐。幸好 stm.go 把这部分逻辑进行了封装，抽象出了一个公共的事务处理框架，我们只需把业务相关的代码传给 stm，它会做事务管理，并适时调用我们的业务代码。现在来看看用 stm 重构后的代码：

```go
func txnXfer(cli *v3.Client, from, to string, amount uint) error {
    // NewSTM创建了一个原子事务的上下文，并把我们的业务代码作为一个函数传进去
    _, err := concurrency.NewSTM(cli, func(stm concurrency.STM) error {
        // stm.Get封装好了事务的读操作
        fromV := toUInt64(stm.Get(from))
        toV := toUInt64(stm.Get(to))
        if fromV < amount {
            return fmt.Errorf(“insufficient value”)
        }
        // stm.Put封装好了事务的写操作
        stm.Put(to, fromUInt64(toV + amount))
        stm.Put(from, fromUInt64(fromV - amount))
        return nil
    })
    return err
}
```

眼花缭乱的`ModRevision`不见了，事务处理也不见了，整段代码变得清晰明了了很多。所以 stm 的使用特别简单——我们只需把业务相关的代码封装成可重入的函数传给 stm，然后 stm 会处理好其余所有的细节，cool~

## 内部实现

先来看看传给业务程序的`concurrency.STM`长什么样：

```go
// STM is an interface for software transactional memory.
type STM interface {
    // Get returns the value for a key and inserts the key in the txn's read set.
    Get(key ...string) string
    // Put adds a value for a key to the write set.
    Put(key, val string, opts ...v3.OpOption)
    // Rev returns the revision of a key in the read set.
    Rev(key string) int64
    // Del deletes a key.
    Del(key string)

    // commit attempts to apply the txn's changes to the server.
    commit() *v3.TxnResponse
    reset()
}
```

原来它是一个接口，提供了对某个 key 的 [CURD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) 操作。而实现了 STM 接口的总共有两个类：[stm](https://github.com/etcd-io/etcd/blob/v3.3.10/clientv3/concurrency/stm.go#L170) 和 [stmSerializable](https://github.com/etcd-io/etcd/blob/v3.3.10/clientv3/concurrency/stm.go#L300)。具体选择哪个实现，则是由我们指定的隔离级别决定的：

```go
func mkSTM(c *v3.Client, opts *stmOptions) STM {
    switch opts.iso {
    case SerializableSnapshot:
        s := &stmSerializable{...}
        s.conflicts = func() []v3.Cmp {return append(s.rset.cmps(), s.wset.cmps(s.rset.first()+1)...)}
        return s
    case Serializable:
        s := &stmSerializable{...}
        s.conflicts = func() []v3.Cmp { return s.rset.cmps() }
        return s
    case RepeatableReads:
        s := &stm{...}
        s.conflicts = func() []v3.Cmp { return s.rset.cmps() }
        return s
    case ReadCommitted:
        s := &stm{...}
        s.conflicts = func() []v3.Cmp { return nil }
        return s
    }
}
```

可以看到目前共支持了4种隔离级别，并且不同的隔离级别除了使用不同的 STM 实现外，他们的`conflicts`函数也不一样，这个我们待会再讲。先挑一个典型的`RepeatableReads`来看看它的实现：

### RepeatableReads

内部的状态：

```go
// stm implements repeatable-read software transactional memory over etcd
type stm struct {
    ...
    // rset holds read key values and revisions
    rset readSet
    // wset holds overwritten keys and their values
    wset writeSet
    // getOpts are the opts used for gets
    getOpts []v3.OpOption
    // conflicts computes the current conflicts on the txn
    conflicts func() []v3.Cmp
}
```

这里的两个字段`readSet`和`writeSet`，底层类型是一个`map`，用来缓存在当前事务中进行过的读写操作。`conflicts`是在上面的工厂函数`mkSTM`中赋值的，用来在事务提交的时候做冲突检测。

#### 读写

事务中的读写调用的是`stm.Get`和`stm.Put`函数。简化后的逻辑大概是这样的：

```go
func (s *stm) Get(key string) string {
    // 先看看之前有没有写过这个值
    if wv, ok := s.wset[key]; ok {
       return wv
    }
    // 再看看有没有读过
    if rv, ok := s.rset[key]; ok {
        return string(rv.Kvs[0].Value)
    }
    // 如果都没有，则向etcd发起一次查询请求
    rk, err := s.c.Get(s.ctx, key, s.getOpts...)
    // 查询后再缓存进read set
    s.rset[key] = rk
    return string(rk.Kvs[0].Value)
}

// 写操作都是先写进本地缓存里
func (s *stm) Put(key, val string) { s.wset[key] = val }
```

对于读请求，先以本地最新的更新为准，如果之前从没处理过这个 key ，则到 etcd 中查询，并缓存下来，这样就可以做到*可重复读* （RepeatableReads）。对于写操作，都是先写进本地缓存，直到事务提交时才真正写到 etcd 里。

#### 提交

当我们的业务代码调用结束后，stm 就来尝试提交事务了：

```go
func (s *stm) commit() *v3.TxnResponse {
    txnresp, err := s.client.Txn(s.ctx).If(
      s.conflicts()...).Then(s.wset.puts()...).Commit()
    ...
}
```

提交事务就是利用 etcd 的`Txn`，把`writeSet`中的数据写出去，这里要注意的是`If`中的冲突检测，就是我们在构建的时候指定的`conflicts`函数：

```go
s.conflicts = func() []v3.Cmp { return s.rset.cmps() }
```

看来`conflicts`函数中真正干活的是`readSet.cmps()`：

```go
// cmps guards the txn from updates to read set
func (rs readSet) cmps() []v3.Cmp {
    cmps := make([]v3.Cmp, 0, len(rs))
    for k, rk := range rs {
        cmps = append(cmps, 
                 v3.Compare(v3.ModRevision(k), "=", rk.Kvs[0].ModRevision))
    }
    return cmps
}
```

它的逻辑就是构造一堆比较操作符，确保`readSet`中的数据，在读取进缓存之后再也没有被更改过，即 ModRevision 没有变。

所以梳理一下，`RepeatableReads`实现的两个关键点：

* 用`readSet`缓存已经读过的数据，这样下次再读取同样数据的时候才能得到同样的结果，这确保了可重复读。
* 用`readSet`数据的`ModRevision`做冲突检测，确保本事务读到的数据都是最新的。

### ReadCommitted

`ReadCommitted`跟`RepeatableReads`的实现类似，唯一不同的是它的冲突检测函数：

```go
s.conflicts = func() []v3.Cmp { return nil }
```

可以看到，其实`ReadCommitted`啥冲突也不检测，它只是确保自己读到的是别人已经提交的数据，其他什么保障也没有啊？！

### Serializable

`Serializable`使用的是`stmSerializable`类，它与上面两个使用的`stm`一个主要不同点在于，它第一次读完了会把当时的版本号给记录下来，下次再向 etcd 发出读请求的时候会带上这个版本号，表示我就要当时那个版本的数据，这确保了该事务所有的读操作读到的都是**同一时刻**的内容。这就相当于在事务开始时，对 etcd 做了一个快照，这样它读取到的数据就不会受到其他事务的影响，从而达到事务串行化（Serializable）执行的效果。

```go
func (s *stmSerializable) Get(keys ...string) string {
    // ...
    firstRead := len(s.rset) == 0 // 是不是第一次读操作
    resp := s.stm.fetch(keys...)
    if firstRead {
        // 记录下first read的版本号，作为后续Get操作的Option参数传进去
        s.getOpts = []v3.OpOption{
            v3.WithRev(resp.Header.Revision),
            v3.WithSerializable(),
        }
    }
    return respToValue(resp)
}
```

可以看到，在第一个读操作完成后，我们就设置了两个`OpOption`给后续的读操作使用。第一个 Option 指定了要读的版本号，第二个 Option 是告诉 etcd server 直接把它当前所知道的结果返回，而不需要通过 raft 协议跟别人确认一下它目前的结果是不是最新的，这提升了读取时的性能同时也能满足该场景下的需求（默认不加这个参数时是线性 linearizable 一致性的读取策略，可靠但是慢）。

### SerializableSnapshot

`SerializableSnapshot`和`Serializable`也是类似，不同的是它的隔离级别更严格些：

```go
s.conflicts = func() []v3.Cmp {
    return append(s.rset.cmps(), s.wset.cmps(s.rset.first()+1)...)
}

// cmps returns a cmp list testing no writes have happened past rev
func (ws writeSet) cmps(rev int64) []v3.Cmp {
    cmps := make([]v3.Cmp, 0, len(ws))
    for key := range ws {
        cmps = append(cmps, v3.Compare(v3.ModRevision(key), "<", rev))
    }
    return cmps
}
```

即，它不仅要确保我读取过的数据是最新的，也要确保我要写入的数据也没有被别人更改过，这是最高的隔离级别，也是 stm 的默认隔离级别。在传统数据库中，要实现这样一个隔离级别，一般通过读写锁来限制竞态冲突，而这里则是通过极其严格的冲突检测，稍微有点不一样的地方它就认为发生冲突了，需要 stm 去重试。

### 区别汇总

综上所述，这四种隔离级别在实现上主要有两点区别：**读取数据的版本**和**冲突检测的方法**。如果我们将事务刚开始时的版本号称为`FIRST_REV`，将读操作真正发生时候的版本号称为`CURRENT_REV`，将 key 的 ModRevision 简称为`MOD_REV`，那它们的区别可以用一张表总结出来：

| Isolation Level      | Read Revision | Conflict Detection                                           |
| -------------------- | ------------- | ------------------------------------------------------------ |
| RepeatableReads      | CURRENT_REV   | readSet: MOD_REV == CURRENT_REV                              |
| ReadCommitted        | CURRENT_REV   | nil                                                          |
| Serializable         | FIRST_REV     | readSet: MOD_REV == FIRST_REV                                |
| SerializableSnapshot | FIRST_REV     | readSet: MOD_REV == FIRST_REV<br />writeSet: MOD_REV < FIRST_REV |

而对于 STM 接口的两个实现`stm`和`stmSerializable`，它们的区别在于，读取操作是否会受到其他事务的影响。如下图所示：

![serializable read](/image/2019/08/etcd-transaction-isolation-levels/serializable-read.svg)

## 总结

本文以 etcd 为例，介绍了事务隔离机制的实现。当然，这里的实现跟传统关系型数据库（MySQL、PostgreSQL）的实现还是有一定区别的。关系型数据库一般通过给一个 row 增加两个字段（transaction-created, transaction-deleted）来做数据可见性的隔离，再辅以悲观锁来做并发控制，而 etcd 里的实现类似于事务型内存，在一个事务的上下文里先缓存业务代码的所有读、写请求，在真正提交的时候根据不同的隔离级别做冲突检测，从而决定是否重试。可以看到，底层暴露出来的 MVCC 原语具有极强的表达能力，我们在客户端就可以做到原本在服务端才能做到的事情。

另外，就 etcd 常见的使用场景官方的 SDK 中都有现成的实现（参见 [concurrency](https://github.com/etcd-io/etcd/tree/v3.3.10/clientv3/concurrency), [contrib/recipes](https://github.com/etcd-io/etcd/tree/v3.3.10/contrib/recipes)），希望大家不要再重复造轮子了，官方的轮子又大又圆，而自己造的轮子往往都是 ctrl+c、ctrl+v 来的代码，到处是坑。多读读源代码，能让你少趟很多坑。