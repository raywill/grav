---
title: RootServer Daily Merge Schedule
taxonomy:
    category: docs
---



## 背景

OceanBase 1.0数据为分动态数据有基线数据，动态数据会定期合并到基线数据。在OceanBase 1.0之前，
我们一般一天只做一次这样的合并，这个过程叫做每日合并(Daily Merge)。每日合并，由统一冻结后由RS调度
OB Server执行合并，本文描述RS调度每日合并过程。

## 名词解释

* 大版本冻结 (major freeze): 所有OB Server冻结memetable，新开新版本memtable用于写入
* 每日合并 (daily merge): 一般一天只发生一次的基线数据与动态数据合并过程
* 错峰合并 (stagger merge): RS调度每日合并策略，按集群做合并，错开合并压力，保证在合并期间有集群能提供正常服务

*TODO baihua*: 错峰好像没有必要？

## 设计目标

调度各cluster执行错峰合并。

## 设计思路及折衷

Daily Merge Schedule在RS内部为一单独线程，轮询当前系统状态。如果发生大版本冻结则发起每日合并流程。
每日合并一般执行错峰合并，但也可所有集群一起合并，由enable_stagger_merge配置项控制。

cluster合并前流量切换与0.5不大一样，不是由client端主动切换而是通过leader改选，将leader切到其它集群。
由于我们只访问partition的leader，leader切换后流量自然发生切换。
同时切流量时，也不考虑渐近的方法，而是一次性全部切换。因为对于一个partition的流量只能全切，
做渐近意义不大。

跟0.5一样，对于RS重启功换每日合并流程不能中断，需继续上次执行，所以每日合并相关状态记在内部表
 \_\_all_cluster中。RS启动时从内部表恢复状态继续执行。

## 系统设计

### 流程

错峰合并主要工作流程如下：

```nohighlight

          thread start
            |
            v
  +-----> wait <---------------------------------------+
  |         |                                          |
  |         v                                          |
  | +---- all cluster merged                           |
  | |       |                                          |
  | | NO    | YES                                      | NO
  | |       v                                          |
  | |     global_broadcast_version < frozen_version ---+
  | |       |
  | |       | YES
  | |       v
  | |     global_broadcast_version += 1
  | |       |
  | |       v
  | +---> arrange cluster merge order
  |         |
  |         v
  | +---> chose next cluster <----------------------------------------------+
  | |       |                                                               |
  | |       v                                                               |
  | |     cluster.broadcast_version == global_broadcast_version             |
  | |       |                            |                                  |
  | |       | YES                        | NO                               |
  | |       |                            v                                  |
  | |       |                          cluster.merge_stat = MERGING         |
  | |       |                          global_merge_cluster = cluster_id    |
  | |       |                            |                                  |
  | |       |                            v                                  |
  | |       |                          ObLeaderCoordinator::coordinate()    |
  | |       |                            |                                  |
  | |       |                            v                                  |
  | |       +-------- cluster.broadcast_version = global_broadcast_version  |
  | |       |                                                               |
  | |       v                                                               |
  | |     wait <-------------------+                                        |
  | |       |                      |                                        |
  | |       |                      | NO                                     |
  | |       v                      |                                        |
  | |     cluster merged ------> check timeut -----> cluster.merge_stat = MERGE_TIMEOUT
  | |       |
  | |       | YES
  | |       v
  | |     cluster.merge_stat = NORMAL
  | |     cluster.last_merged_version = cluster.broadcast_version
  | |       |
  | | NO    |
  | |       v
  | +---- all cluster merged
  |         |
  |         | YES
  |         v
  +-------global_merge_cluster = 0


```

非错峰合并工作流程类似，但是是所有集群同时合并，没有合并顺序安排，leader改选等，流程更为简单，这里不做描述。

### 合并顺序

每次合并并不以固顺序执行，为了减少RS切换顺序，总是最后合并RS所在集群。
合并顺序为：集群按cluster_id排序(*TODO baihua*: 以后可能过些顺序以后可由配制项指定)，
然后从RS所在集群后面的集群开始循环执行。

如现在有三个集群：cluster1, cluster2, cluster3 (cluster_id分别为: 1, 2, 3)，RS在cluster2上。
那么合并顺序为：cluster3, cluster1, cluster2。

### 系统合并状态

合并状态记录在\_\_all_cluster表中，主要有以下几个：

- 全局状态：
  - global_broadcast_version: 表示正在合并哪个版本
  - global_merge_stat: 全局的全并状态: NORMAL, MERGING, MERGE_TIMEOUT
  - global_merge_cluster: 正在合并的集群，leader协调模块将此集群leader平分到其它集群
- 集群状态：
  - merge_start_time: 本集群开始合并时间
  - last_merged_time: 本集群合并完成时间
  - last_merged_version: 本集群已合到的版本
  - is_pause_merge: 合并是否被暂停

*TODO baihua*: rename:
last_merged_time --> merge_finish_time, last_merged_version --> merged_version, is_pause_merge --> pause_merge ?

### 流量切换

正在合并集群将global_merge_cluster标记为本集群，leader协调模块会将些集群leader切到其它集群。
设置global_merge_cluster后，需主动调用leader协调模块做切换(ObLeaderCoordinator::coordinate())。

*TODO baihua*: add leader coordinator doc link

### 合并完成判断

0.5版本中，每个集群合并完成的标准为：

1. 本集群有完整alive副本 (roottable不能有空洞)
2. roottable中所有属于此cluster的alive副本都合并到broadcast_version.

1.0版本中，我们不需要保证partition table在一个集群没有空洞，但还是需要保证partition safe副本数。
所以集群合并完成的条件改为：

1. 所有partition至少有alive的safe副本数。
2. partition table中本集群alive partition合并到broadcast_version

至于partition是否有足够alive副本由balance模块保证，是否缺少副本并不影响合并进度。

*alive副本*: 是指副本所在CS/OB Server是活的(与RS有心跳)。
*safe副本数*: 是指能保证选出Leader的最少副本数(majorty 副本)。

### 等待唤醒

daily merge线程在轮询系统状态过程中，以及待集群合并完成时都会执行等待操作(流程图中的wait)。
每次等待config.check_merge_interval时长，也可由下以动作唤醒：

- 退出
- 发生major freeze
- 集群所有机器汇报此版本合并完成。(要求OB Server在合并完一个版本后通过RPC通知RS此版本合并完成)

### 异常处理

配制cluster的合并超时时间，如果集群合并时间超过此时间，则将集群标记为MERGE_TIMEOUT，
并开始下个集群的合并。MERGE_TIMEOUT集群合并完成时，会自动标记为完成，不需要人工介入处理。


*TODO baihua*: to be continue

## 兼容性考虑

### SQL功能
### 老版本升级

## DBA考虑

### SQL功能
### 老版本升级
### SQL功能

## 测试考虑
