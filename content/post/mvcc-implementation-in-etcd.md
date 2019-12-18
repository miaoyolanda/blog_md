---
title: "MVCC 在 etcd 中的实现"
date: 2018-12-24T18:26:52+08:00
lastmod: 2020-01-15T17:16:23+08:00
keywords: ["mvcc", "etcd", "数据库", "事务", "多版本并发控制"]
---

## 简介

在数据库领域，面对高并发环境下数据冲突的问题，业界常用的解决方案有两种：

1. 想办法避免冲突。使用**悲观锁**来确保同一时刻只有一人能对数据进行更改，常见的实现有读写锁（Read/Write Locks）、两阶段锁（Two-Phase Locking）等。

2. 允许冲突，但发生冲突的时候，要有能力检测到。这种方法被称为**乐观锁**，即先乐观的认为冲突不会发生，除非被证明（检测到）当前确实产生冲突了。常见的实现有逻辑时钟（Logical Clock）、[MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)（Multi-version Cocurrent Control）等。

在这些解决方案中，MVCC 由于其出色的性能优势，而被越来越多的数据库所采用，比如Oracle、PostgreSQL、MySQL InnoDB、etcd 等。它的基本思想是保存一个数据的多个历史版本，从而解决事务管理中数据隔离的问题。

这里的版本一般选择使用时间戳或事务 ID 来标识。在处理一个写请求时，MVCC 不是简单的用新值覆盖旧值，而是为这一项添加一个新版本的数据。在读取一个数据项时，要先确定一个要读取的版本，然后根据版本找到对应的数据。这种写操作创建新版本，读操作访问旧版本的方式使得读写操作彼此隔离，他们之间就不需要用锁来协调。换句话说，在 MVCC 里读操作永远不会被阻塞，因此它特别适合 etcd 这种读操作比较多的场景。

MVCC 的基本原理就是这么简单，下面来结合具体代码，看看 etcd 是怎么实现的。跟[上篇]({{< ref "raft-implementation-in-etcd.md" >}})一样，这里的分析仍以etcd [v3.3.10](https://github.com/etcd-io/etcd/tree/v3.3.10) 为例。

## 数据结构

etcd 的代码中，有关 MVCC 的实现都在一个叫 [mvcc](https://github.com/etcd-io/etcd/tree/v3.3.10/mvcc) 的包里。在正式介绍读写流程之前，我们先来了解下这个包里面定义的几个基础数据结构。

### revision

[revision](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/revision.go) 即对应到MVCC中的版本，etcd 中的每一次`key-value`的操作都会有一个相应的 `revision`。

```go
// A revision indicates modification of the key-value space.
// The set of changes that share same main revision changes the key-value space atomically.
type revision struct {
    // main is the main revision of a set of changes that happen atomically.
    main int64

    // sub is the the sub revision of a change in a set of changes that happen
    // atomically. Each change has different increasing sub revision in that
    // set.
    sub int64
}
```

这里的 main 属性对应事务 ID，全局递增不重复，它在 etcd 中被当做一个逻辑时钟来使用。sub 代表一次事务中不同的修改操作（如put和delete）编号，从0开始依次递增。所以在一次事务中，每一个修改操作所绑定的`revision`依次为`{txID, 0}`, `{txID, 1}`, `{txID, 2}`...

### keyIndex

[keyIndex](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/key_index.go)用来记录一个key的生命周期中所涉及过的版本（revision）。

```go
type keyIndex struct {
    key         []byte
    modified    revision // the main rev of the last modification
    generations []generation
}
```

它保存的是当前`key`的具体值（key），最近一次修改的版本号（modified），以及记录`key`生命周期的`generation`，一个`generation`代表了一个`key`从创建到被删除的过程。因为一个`key`可能会经历创建 -> 删除 -> 创建的好几个轮回，所以它有可能会有一个或多个`generation`。下面来看看一个`generation`包含哪些内容：

```go
// generation contains multiple revisions of a key.
type generation struct {
    ver     int64
    created revision // when the generation is created (put in first revision).
    revs    []revision
}
```

可以看到，一个`generation`中最重要的信息就是那个`revision`数组，代表在它这一代中，所经历过的版本变更。

不知所云？没关系，我刚开始看这段代码的时候也是一脸懵逼，不明白为啥要这样层层包装。不过还好我们有[UT](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/key_index_test.go)，跟着UT里面的测试样例大概梳理出这样的逻辑：

* 假设我们有一个值为`foo`的key

```go
ki := &keyIndex{key: []byte("foo")}
```

* 为它记录两次版本变更

```go
ki.put(zap.NewExample(), 2, 0) // 第一次变更发生在ID为2的事务中，且是第一次操作
ki.put(zap.NewExample(), 4, 2) // 第二次变更发生在ID为4的事务中，且是第三次操作
```

* 那这个`keyIndex`变成这样的结构

```yaml
key: "foo"
modified: {4, 2}
generations:
    [{2, 0}, {4, 2}]
```

* 如果在此后的事务中这个key经历了被删除，再创建的过程：

```go
ki.tombstone(zap.NewExample(), 6, 0)  // 在第6次事务中被删除
ki.put(zap.NewExample(), 8, 0)
ki.put(zap.NewExample(), 10, 0)
```

* 那它会多一个`generation`

```yaml
key: "foo"
modified: {10, 0}
generations:
    [{2, 0}, {4, 2}, {6, 0}(t)],
    [{8, 0}, {10, 0}]
```

好了，在知道了key的版本信息及相应的操作后，我们来看看怎么把它们组织起来：

### treeIndex

[treeIndex](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/index.go)顾名思义就是一个树状索引，它通过在内存中维护一个[B树](https://github.com/google/btree)，来达到加速查询key的功能。

这棵树的每一个节点都是`keyIndex`，它实现了[Item](https://github.com/google/btree/blob/v1.0.0/btree.go#L59)接口，其中的[比较函数](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/key_index.go#L281)主要是比较`key`的大小。这样，给定一个`key`就能很快查到它对应的`keyIndex`，进而可以得知它所有的版本信息。树状结构比较简单，看看图就能明白，我就不再分析了。

{{< figure src="/image/mvcc-in-etcd/treeIndex.svg" caption="treeIndex结构" data-edit="https://www.draw.io/#G1qAcogN4yVxoAZKj31WMkTTslxEYxM-7q">}}

值得注意的是，这里只存有key的信息，value保存在磁盘里，一方面key一般比较小，同样的内存可以支持存储更多的key；另一方面，value和版本是一一对应的，内存中维护了版本的信息，也就没必要再多存一份value了。

看完了key以及key在内存中的组织结构，我们再来看看value是怎么处理的。

### backend

backend封装了etcd中的存储，按照设计，backend可以对接多种存储，当前使用的是[boltdb](https://github.com/etcd-io/bbolt)，但作为一个CNCF项目，官方也在考虑接入更多的存储引擎[^pluggable-backend]。boltdb是一个纯Go语言实现的支持事务的KV存储，etcd的事务就是基于boltdb的事务实现的。

[^pluggable-backend]: [Meta issue: Pluggable backend #10321](https://github.com/etcd-io/etcd/issues/10321)

etcd在boltdb中存储的key是`reversion`，value是etcd自己的`key-value`组合。因此，每次更新时，新的`reversion`会被记在`keyIndex`中，同时，这个`reversion`所对应的`key-value`组合会被存到boltdb中。

举个例子： 用etcdctl通过批量接口写入两条记录：

```bash
etcdctl txn <<<'
put key1 "v1"
put key2 "v2"
'
```

再通过批量接口更新这两条记录：

```bash
etcdctl txn <<<'
put key1 "v12"
put key2 "v22"
'
```

boltdb中其实有了4条数据：

```bash
rev={3 0}, key="key1", value="v1"
rev={3 1}, key="key2", value="v2"
rev={4 0}, key="key1", value="v12"
rev={4 1}, key="key2", value="v22"
```

所以总的来说，**内存 btree 维护的是 key 到 keyIndex 的映射，keyIndex 内维护多版本的 revision 信息，而 revision 可以映射到磁盘bbolt中具体的value。下面这个图很好的表示了这个过程[^lichuang]**：

[^lichuang]: [etcd存储的实现](https://lichuang.github.io/post/20181125-etcd-server/)

{{< figure src="/image/mvcc-in-etcd/etcd-keyindex.png" caption="MVCC数据流程">}}

基本概念就介绍到这里了，下面我们把这些概念串起来，看看读写操作的完整流程是怎样的。

## 写操作

首先客户端的`Put`请求会被`EtcdServer`的[Put函数](https://github.com/etcd-io/etcd/blob/v3.3.10/etcdserver/v3_server.go#L111)接受到，它经过一套 raft 协议的流转后，在 apply 阶段被写入，精简后的写入逻辑大概是这样的：

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/kvstore_txn.go#L151
func (tw *storeTxnWrite) put(key, value []byte, leaseID lease.LeaseID) {
    rev := tw.beginRev + 1

    ibytes := newRevBytes()
    idxRev := revision{main: rev, sub: int64(len(tw.changes))}
    revToBytes(idxRev, ibytes)

    kv := mvccpb.KeyValue{
        Key:            key,
        Value:          value,
        CreateRevision: c,
        ModRevision:    rev,
        Version:        ver,
        Lease:          int64(leaseID),
    }
    d, err := kv.Marshal()

    tw.tx.UnsafeSeqPut(keyBucketName, ibytes, d)
    tw.s.kvindex.Put(key, idxRev)
    tw.changes = append(tw.changes, kv)
}
```

正如前面所分析的，这里的写入操作首先获取了当前事务开始时的ID（tw.beginRev），构造出了新的`revision`，然后将 value 写入到后端的`boltdb`中，并更新了内存中 key 的索引（treeIndex）。

## 读操作

读操作的处理函数是由`EtcdServer`的[Range函数](https://github.com/etcd-io/etcd/blob/v3.3.10/etcdserver/v3_server.go#L86)处理的，在走完一段`Linearizable Read`协议后，我们发现最终的操作是由`storeTxnRead.rangeKeys`来完成的：

```go
// https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/kvstore_txn.go#L111
func (tr *storeTxnRead) rangeKeys(key, end []byte, curRev int64, ro RangeOptions) (*RangeResult, error) {
    rev := ro.Rev

    revpairs := tr.s.kvindex.Revisions(key, end, rev)

    limit := int(ro.Limit)
    kvs := make([]mvccpb.KeyValue, limit)
    revBytes := newRevBytes()

    for i, revpair := range revpairs[:len(kvs)] {
        revToBytes(revpair, revBytes)
        _, vs := tr.tx.UnsafeRange(keyBucketName, revBytes, nil, 0)
        kvs[i].Unmarshal(vs[0])
    }
    return &RangeResult{KVs: kvs, Count: len(revpairs), Rev: curRev}, nil
    }
```

这里函数的参数`curRev`代表当前事务ID。拿到了`key`以及感兴趣的`revision`之后，我们去`treeIndex`中查询，满足指定条件的`revision`有哪些，然后再根据这些`revision`去`boltdb`中查询当时存的`key-value`是什么，并封装成`RangeResult`返回出去。

## 小结

本文将乐观锁的原理及其在 etcd 中的实现 MVCC 梳理了一遍，希望对读者在理解和使用乐观锁上能有所帮助。

在 etcd 中，MVCC 的实现还是比较简单直接的。但在阅读代码的过程中，我也有一些看不明白的地方，毕竟一段代码在演化的过程中，会多出各种各样的细节处理，比如边界情况，性能调优等。这些细节会扰乱视线，让我们难以抓住主干。一般碰到这种情况，我会采用两种方法：

1. 阅读UT，看他们都在测试什么。一般UT的输入输出都比较确定，它能够向我们揭示这段代码的作者期望完成的功能是什么。有了这个目标后我们再去看实现，就能剔出哪些是跟主干功能有关的，哪些是细节处理。
2. 查找较早的提交。既然代码是演进出来的，那它刚开始的几个版本一般都比较粗糙，有很多细节没有处理，但那也是这个功能最直接的实现。通过阅读作者早期的实现，也能帮助我们快速抓住主干。
