---
title: 'ARIES(1/n) - 使用 WAL 实现 原子性和一致性'
date: 2022-10-08 14:44:24
updated: 2022-10-08 14:44:30
categories: 
- paper-reading
tags: 
- ARIES
- paper
- db
description: ARIES 如何使用 WAL 实现 原子性和一致性
---


## concept

本文中讨论的 `持久化`（`Durability`）指：数据保存到了 非易失（non-voliatile）存储设备上，断电后不会丢失。

本文会混用 `刷盘`，`从内存写入数据库`，`把数据持久化`，他们都表达相同的含义。

本文基于假设：硬盘的`顺序 IO` 远快于 `随机 IO`。

## 背景
`ARIES`(Algorithm for Recovery and Isolation Exploiting Semantics) 是一个使用 WAL 来实现 数据库的 原子性 和 持久性 的算法。

如果没有 log 的话，数据库只由两部分组成：
- 内存中的 buffer pool
- 硬盘

为了保证 原子性 和 持久性，需要加上一些限制：

当 buffer pool 中的 page 被修改时，
- 标记 dirty page 为 pinned（数据未刷盘，不允许替换）

提交事务时，
1. 把 dirty page 同步到 disk
2. 标记 dirty page 为 unpinned（数据已刷盘，可以被替换）
3. 提交事务

这个方案存在问题：
- 如果事务很大，会需要`大量的 buffer pool 空间`，极端情况下，内存中需要存储和硬盘等量的数据。不可能提供这么大的内存空间。
- 如果一个事务修改了多个 page 的话，需要一个一个地同步到硬盘上，如果中途断电的话，会发生`部分提交`的情况。此外，page 同步到硬盘会产生大量的 `随机 IO`。

因此：
- 但凡 一个事务影响了多于 1 个页，就无法保证 原子性。
- 而且，一个事务修改的页可能并不相邻，需要进行大量的随机 IO。

重新定义一下上面的约束：
>- `no steal`: dirty page 不能刷盘，除非其关联的所有事务都提交了。
>- `force`: 事务提交时，需要将关联的所有 dirty page 刷盘。

ARIES 借助 WAL 实现了 `no force/steal`，高效地保证了 原子性 和 持久性。
### no force
> `no force`: 事务提交时，不需要将关联的所有 dirty page 刷盘。

Q：不强制 dirty page 刷盘的话，如果在 全部 dirty page 全部刷盘之前，断电的话，数据丢失怎么办？

A：把变更记录 (`redo log`) 写入硬盘，变更记录全部刷盘了，事务就可以提交了。
变更记录要小的多，而且是 顺序 IO 的，会大幅减少事务的耗时。

### steal
> `steal`: dirty page 可以随时刷盘，不需要关心其关联的事务是否提交。

Q：如果事务回滚了怎么办？脏数据就被刷盘了啊？

A：记录下回滚事务必需的信息 (`undo log`)，
具体地，
语句   | 回滚必需的信息
-------|------------------
insert | id
delete | id, 所有字段
update | id, 更新的列，更新的列的 旧值

## 实现

>WAL 协议：
>1. 在 内存中的 data page 写入硬盘之前，必须保证所有对这个 page 更改的 `undo log` 已经持久化。
>2. 在事务提交之前，必须保证事务对应的所有 `redo log` 已经持久化。

在 ARIES 设计中，buffer manager 通过满足 WAL 协议，来保证事务。具体实现上，由 `page 同步刷盘 (随机 IO，大)`，改为了 `log 刷盘 (顺序 IO，小)` + `异步 page 刷盘 (随机 IO，大)`。

### 判断 page 是否可以刷盘
和 data page 一样，log 也是先写入内存，在写入硬盘的。
每条 log 有一个递增的唯一的编号：`LSN(Log Sequence Number)`。
有了 LSN，就可以判断 buffer pool 中的页是否可以刷盘。

内存中维护一个 `flushedLSN`，标识：`已经持久化的 log 的 最大 LSN`

每个 data page 上有一个 `pageLSN` ，标识：`对这个 page 的最后一次更改对应的 log 的 LSN`。

如果 `pageLSN <= flushedLSN`，说明：对这个 page 的所有更改都已经记录到硬盘上了，page 可以随意地刷盘。(如果内存需要从硬盘加载新的 page，空间不够的话，这个 page 可以刷盘，然后释放出来)
![](https://yzp-note.4everland.store/img/aries/flushedLSN-1.png)
![](https://yzp-note.4everland.store/img/aries/flushedLSN-2.png)

### 数据结构
![](https://yzp-note.4everland.store/img/aries/data-structure.png)
#### LogRecords

LogRecords 有 3 种：
|          | 作用级别      | 用途                 |
|----------|---------------|---------------------|
| redo log | Page-oriented | 记录变更记录         |
| undo log | Logical       | 回滚事务必需的信息   |
| CLR      | Page-oriented | undo log 的 redo log |

Page-oriented：描述对具体页的修改

Logical：描述业务操作，比如：修改表 a 的第 3 个字段 为 'bbb'；undo log 使用 Logical，是为了提高事务的并行度。如果 undo log 设计为 Page-oriented 的话，两个事务就没法同时生成 undo log，锁要加到 page 粒度上。

CLR(Compensation Log Record) 是 undo log 在 page 层面的体现，用于回滚失败时的重试。

#### memory
在内存中维护两张表：
`Transaction Table`，每一行代表一个活跃的事务，包含字段：
- XID
- Status(running, committing, aborting)
- lastLSN (这个事务最后写入的 LSN)

`Dirty Page Table`，每一行代表一个 buffer pool 中的 dirty page，包含字段：
- pageId
- recLSN (导致 clean page 变为 dirty page 的那个 LSN)

#### disk
数据库系统周期性地创建 checkpoint，以减少崩溃恢复时需要进行的工作。
checkpoint 过程：
1. 记录 `begin_checkpoint record` 到 log
2. 保存 `transaction table` 和 `dirty page table` 到硬盘
3. 记录 `end_checkpoint record` 到 log
4. 记录 `begin_checkpoint record` 的 LSN 到硬盘

`master record`

### 崩溃恢复的过程
崩溃恢复的伪代码：
```cpp
RESTART(Master_Addr);

// 1
Restart_Analysis(Master_Addr, Trans_Table, Dirty_Pages, RedoLSN);
// 2
Restart_Redo(RedoLSN, Trans_Table, Dirty_Pages);

buffer pool Dirty_Pages table := Dirty_Pages;

remove entries for non-buffer-resident pages from the buffer pool Dirty_Pages table;
// 3
Restart_Undo(Trans_Table);

reacquire locks for prepared transactions;

checkpoint();

RETURN;

```
![](https://yzp-note.4everland.store/img/aries/recovery_process.png)

包括 3 个阶段：
#### analysis pass

从 master record 读取 `begin_checkpoint record` 的 LSN；

从 LSN 遍历 log，重建系统崩溃时的 `trans table` 和 `dirty page table`。

从 `trans table` 找出 checkpoint 之后，所有的 `committed trx` 和 `failed（包括进行中的和取消的）trx`：

(** 跳过对于 committing trx 的处理)

#### redo pass

执行所有 没有作用到 disk 的 redo log（包括 取消的事务对应的 redo log），
这个起点就是`RedoLSN`，在第一步中，通过计算 `dirty page table` 中最小的 recLSN 得到。

因为 `steal`，所以 dirty page 可能已经刷盘了，而事务还没有提交，在执行 redo log 时，需要检查 是否真的需要执行：
- 如果 log 对应的 page 不在 `dirty page table` 中，说明 dirty page 已经刷盘；不执行；
- 如果 log 对应的 page 在 `dirty page table` 中，但 recLSN > LSN，说明 LSN 对应的更改已经刷盘；不执行；
- 如果 disk page 的 pageLSN >= LSN，说明 LSN 对应的更改已经刷盘；不执行；


Q: 为什么要 执行 所有的 redo log，可以只处理 committed trx 对应的 redo log 吗？（`Selective Redo`)

不能。

见下图，redo log(T2,LSN=20) 如果不执行的话，
undo 会去回滚 (T2,LSN=20) 这个操作（因为 T2 失败了），而这个操作并没有发生，数据会产生不一致。

Q2: 那能不能 失败的事务，不执行 undo log，redo、undo 都不执行，数据不就一致了？

不能。

因为 `steal`，失败事务的改动可能刷盘了。
所以必须要 先执行 redo log，把 dirty page 同步到 和 redo log 最新的一致，再执行 undo log，把这个失败事务的所有改动再回滚掉，然后，这个 dirty page 就是`正确`的了。
![](https://yzp-note.4everland.store/img/aries/selective-redo_problem.png)

在下图中，redo log(T2,LSN=20) 不执行，而对应的 undo log 执行的话，并不会导致不一致，因为 redo log(T2,LSN=20) 虽然没执行，但是数据被刷盘了（因为 `steal`），但系统不能依赖这样的刷盘。
![](https://yzp-note.4everland.store/img/aries/selective-redo_problem-free.png)
#### undo pass

执行所有 failed trx 对应的 undo log


## ref

- [ARIES: A transaction recovery method supporting fine-granularity locking and partial rollbacks using write-ahead logging](https://scholar.google.com/scholar?cluster=2142924814045750364)
  - [论文解读_阿莱克西斯_zhihu](https://zhuanlan.zhihu.com/p/143173278)
  - [论文解读_zhihu](https://zhuanlan.zhihu.com/p/440465035)
  - [CS186_bilibili](https://www.bilibili.com/video/BV13a411c7Qo)
