# RootServer Major Freeze

## 背景

Oceanbase 1.0数据分为动态数据和基线数据，动态数据是全部放在内存里的，我们称之为memtable。内存的空间
有限，随着数据的增删，memtable占用的内存空间会越来越多，此时我们需要将当前memtable冻结住，并生成新
版本的memtable，之后新的数据只会往新版本的memtable插，老版本的memtable不再更改，这个过程我们称之为
freeze（冻结）。老的memtable之后会持久化到磁盘上生产转储文件，这个过程叫做dump（转储）。之后在业务
的低峰期时，转储文件会跟基线数据进行合并生成新版本的基线数据，合并操作一般每日做一次，我们称之为
每日合并。

在ob0.5中，我们只有一个UpdateServer，所有动态数据都在同一台server上，major freeze是一个单机事务。在ob1.0
中，每一个partition都有自己的memtable，partition是分布到所有server上的，任何一个observer都可以
执行写事务，动态数据是分布在所有server上的，所以ob1.0中major freeze是一个分布式事务。

## 设计目标

* 避免冻结过程耗时过长

冻结一个partition需要禁止在该partition上开新的写事务，如果当前有写事务在执行过程中，等待一定时间后
该写事务仍未结束，则会直接把该写事务kill掉。由于在冻结过程中需要停写，如果整个冻结过程耗时过长，则
会极大地影响业务。

* 能应对server宕机等异常情况

分布式系统中，server宕机是很常见的情况；冻结的过程需要生成新的memtable，申请资源是可能失败的。即使
这些异常情况发生，major freeze也需要能继续进行下去。

## 设计思路及折衷

分布式事务一般采用两阶段提交来实现，major freeze一开始采用的方案是所有租户的partition一起做冻结，
首先所有partition做prepare_major_freeze，如果都成功了，rs先对\_\_all_core_table的partition做
commit_major_freeze操作，如果成功了则认为整个分布式事务成功了。由于冻结的过程中内部表不可写，所以
major freeze是commit还是abort不能通过写内部表来持久化，而是通过写\_\_all_core_table的partition的
commit_major_freeze日志来持久化。

为了拿到一个统一的版本，冻结的时候需要停写，所有租户的partition一起做冻结会导致所有租户有冻结的
过程中都无法写，这对业务的影响还是比较大的，而且，如果有某些partition做prepare特别慢，这会导致在
它前面prepare成功的 partition需要等它prepare成功后才可以commit，当参与冻结的partition数量越多时，
发生这种情况的概率就越高，最后，即使只有一个partition prepare失败了，整个事务都需要abort掉，然后
从头再做一遍，参与冻结的partition数越多，事务重做的概率就越大。因此，最终采用按租户进行冻结。

rs按租户进行冻结，每个租户的冻结都是一个两阶段提交的过程。rs先冻结普通租户，最后冻结root租户。
按租户冻结使得冻结造成的停写局限在一个租户，并且由于一个租户涉及的partition数量不会很多，涉及的
server有限，出错的概率想对于全局冻结小，所以停写时间不会太长。

每个租户的冻结是commit和abort都需要持久化，由于普通租户在root租户之前冻结，所以普通租户的冻结状态
可以考虑写到内部表里。这里为了简化，所有租户冻结逻辑一致：选择(table_id, partition_id)最小的partition做为
coordinate partition (系统租户会选\_\_all_core_table partition)，所有操作先操作coordinate partition。

另外，RS可能因为线程未及时退出，状态未及时设置等，在RS发生切换时可能会出现多个RS同时做major freeze
的情况。针对这种情况，RS在次做major freeze时生成一个时间戳，并在所有RPC中都带上此时间。OB Server端在
prepare_major_freeze时记录此时间，并通过log同步到所有副本。只有时间戳相同的commit或abort才被接受。

RS通过以下RPC完成major freeze: (每个rpc都包含需要操作的partition列表，以及RS执行本轮major freeze的timestamp)

- prepare_major_freeze rpc: 接收到的observer会对指定的partition做prepare操作，包括停写，
冻结当前memtable，生成新memtable，写prepare日志并同步给所有副本。此rpc会带上RS端最新的schema版本，
如果observer端schema版本不相等，则报错。
- commit_major_freeze rpc: 接收到的observer会对指定的partition做commit操作，包括写commit日志和恢复写，
commit_major_freeze会一直重试直到所有server都返回成功。
- abort_major_freeze rpc: 回滚之前的prepare操作，恢复对之前memtable的写。
跟commit_major_freeze一样，abort_major_freeze会一直重试知道所有server都返回成功。

(在不引起岐义的情况下，这些rpc会分别简称为prepare, commit, abort)


OB Server端partition针对一个版本的major freeze有3个状态：INIT, PREPARED, COMMITED

```nohighlight

+------+      prepare       +----------+      commit          +----------+
| INIT |------------------->| PREPARED |--------------------->| COMMITED |
+------+                    +----------+                      +----------+
  ^                            |
  |           abort            |
  +----------------------------+

```

## 流程

### 整体冻结流程如下：

```nohighlight

rs major freeze main workflow:

try_frozen_version: __all_zone.name = 'try_frozen_version'
frozen_version: __all_zone.name = 'frozen_version'

  begin major freeze
          |
          v
  update try_frozen_version += 1
          |
          v
  try_frozen_version == frozen_version
          |                     |
          |false                +------------------------------------+
          v                                                          |
  get partition distribution by iterating partition table            |
          |                                                          |
          v                                                          |
  normal tenants do major freeze one by one                          |
          |                                                          |
          v                                                          |
  root tenant do major freeze                                        |
          |                                                          |
          v                                                          |
  update frozen_version = try_frozen_version                         |
          |                                                          |
          v                                                          |
  finish major freeze <----------------------------------------------+

```

### 租户冻结的流程如下：

租户major freeze主分为两个阶段：

- STEP1: 从未完成流程或错误中恢复

```nohighlight

 +-----+
 |begin|
 +-----+
   |
   v
 +--------------------------------------------------------+
 |1. get all partitions' major freeze status && timestamp |
 +--------------------------------------------------------+
   |
   v
 +-------------------------------+   YES    +------------------------+
 |coordinate partition commited? |--------->|2. commit all partition |
 +-------------------------------+          +------------------------+
   |                                           |
   | NO                                        +------------------+
   v                                                              |
 +-------------------------+   YES      +----------------------+  |
 |has partition prepared?  |----------->|3. abort all partition|  |
 +-------------------------+            +----------------------+  |
   |                                       |                      |
   | NO                                    |                      |
   v                                       |                      |
 +----+                                    |                      |
 |end |<-----------------------------------+----------------------+
 +----+

```

上面的1, 2, 3都需要重试到成功为止，其中:

- (1) 取状态时须要最后取coordinate partition状态。
- (3) abort所有partition时须要最先abort coordinate partition

STEP1中不生成新的时间戳，commit和abort使用的时间戳经第1步获取状态时得到。

- STEP2: 开始一个全新的冻结


```nohighlight
 +-----+
 |begin|
 +-----+
   |
   v
 +-------------------------+
 |1. generate new timestamp|
 +-------------------------+
   |
   v
 +-------------------------------+    FAIL
 |2. prepare coordinate partition|---------------------------+
 +-------------------------------+                           |
   |                                                         |
   | SUCCESS                                                 |
   v                                                         v
 +-------------------------------+    ANY PARTITION FAIL   +-----+
 |3. prepare all other partition |------------------------>|ERROR|
 +-------------------------------+                         +-----+
   |                                                         ^
   | SUCCESS                                                 |
   v                                                         |
 +-------------------------------+    FAIL                   |
 |4. commit coordinate partition |---------------------------+
 +-------------------------------+
   |
   | SUCCESS
   v
 +------------------------------+
 |5. commit all other partition |
 +------------------------------+
   |
   v
 +---+
 |end|
 +---+

```

STEP2 中所有rpc使用第1步中生成的时间戳。其中第5步中commit所有partition时，必须重试到所有
partition成功为止。其它步骤中，如果是rpc超时这样的未决状态，必须重试rpc到取得明确成功/失败状态为止。
如果最终进入到了ERROR状态，则RS应进入STEP1，从错误中恢复。


整个流程如下：

```nohighlight
+-------+
| begin |
+-------+
  |               +-----+
  |    +----------|sleep|-------------+
  |    |          +-----+             |
  v    v                              | NO
+--------+  ERROR    +-----+        +-------------+
| STEP2  |---------->|STEP1|------->|all commited?|
+--------+           +-----+        +-------------+
  |                                   |
  | SUCCESS                           | YES
  |                                   |
  v                                   |
+-----+                               |
| end |<------------------------------+
+-----+

```

其中，如果在在STEP1中，如果不是将所有partition都commit了，则需要sleep一段时间再尝试STEP2
(避免停写时间过长)。sleep时间从10s以x2的速度增加，最长sleep时间为10分钟。

## 异常流程处理

### 冻结过程中发生切主

冻结root租户的过程中，如果所有partition都prepare了，在做commit之前，如果有partition发生
切主，由于partition table已经不可写了，所以新的leader位置信息没法写到partition table里面，
此时rs会一直重试，但由于从partition table里面读到的leader不对或者leader不存在，rs的重试
会一直失败。对于这种情况，root租户在做recover major freeze的过程中，要先把partition table
的partition commit/abort 掉。

### 冻结过程中整个集群重启

冻结root租户的过程中，如果所有partition都prepare了，在做commit之前，如果整个集群重启，
那么所有partition都要重新选主，rs重启的过程中，是先刷schema，再reload all zone表，
再发起继续做未完成的major freeze的任务, 由于partition table不可以写，刷schema会因为一直
往老的leader发sql而失败，all_zone表也不可读，所以一直无法重启成功。对于这种情况，rs启动
时，要先判断是否有未完成的冻结，然后先将all_core_table和all_root_table commit/abort掉，
只要这两个表可写了，所有系统表就可读了，do restart任务就能完成，未完成的major freeze也可以
成功发起并最后完成。但判断是否有未完成的冻结需要读__all_zone表，在commit/abort all_core_table
和all_root_table前__all_zone表是不可读的，所以我们在__all_core_table里也记录frozen_version
和try_frozen_versio。另外，commit/abort partition需要带上schema version，但由于此时刷schema
还没做所以拿不到schema version，考虑到all_core_table和all_root_table不会建索引表，所以commit
/abort这两个表我们不检查schema_version是否一致。
