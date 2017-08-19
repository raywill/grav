# OceanBase-1.0 复制迁移 设计文档
标签（空格分隔）： markdown OceanBase 复制迁移 设计文档 

------
版本修订历史
0.1 希羽 2014/12/24：新建markdown格式文档

------
### 目 录
1 [背景](#background)
2 [名词解释](#glossary)
3 [设计目标](#design_goal)
4 [设计思路及折衷](#design_idea)
5 [系统设计](#system_design)
    5.1 [基本介绍](#summary)
    5.2 [系统架构图及说明](#achitecture_and_notes)
    5.3 [与外部系统的接口](#outer_API)
    5.4 [详细介绍](#detail)
    5.5 [异常处理](#exception)
6 [模块](#module1) 
    6.1 [基本介绍](#module1_summary)
    6.2 [系统架构图及说明](#module1_achitecture_and_notes)
    6.3 [与外部系统的接口](#module1_outer_API)
    6.4 [详细介绍](#module1_detail)
    6.5 [异常处理](#module1_exception)
7 [兼容性考虑](#compatibility)
    7.1 [SQL功能](#SQL)
    7.2 [老版本升级](#upgrade)
8  [DBA考虑](#design_for_DBA)
    8.1 [使用限制](#limitation)
    8.2 [配置项说明](#usage)
    8.3 [错误信息](#error)
9 [测试考虑（可选）](#design_for_TEST)
10 [设计评审意见](#design_feedback)
11 [附件及参考资料](#design_reference)

------

<span id="background">
### 1 背景 
</span>
当扫描RootTable中的partition时：

* 如果某个partition的副本数小于阀值，选取某台包含partition副本的ObServer为迁移源，另外一台符合要求的ObServer为迁移目的地，生成partition迁移任务。迁移目的地需要符合一些条件，比如，不包含待迁移partition，服务的partition个数小于平均个数减去可容忍个数（默认值为10），正在进行的迁移任务不超过阀值，等等。

* 如果某台ObServer包含的某个表格的partition个数超过平均个数与可容忍个数（默认值为10）之和，以这台ObServer为迁移源，并选择一台符合要求的ObServer，生成partition迁移任务。

<span id="glossary"></span>
### 2 名词解释 

* 基准数据：又称静态数据，某个时间点的数据快照，按partition划分以后存储在本次存储介质上。

* 增量数据：又称动态数据，在某个时间点以后更新数据，存储在内存中，以Btree和Hash进行索引，另外存储其更新日志。

* TableSpace: 表空间，存储一个或是多个表的数据的物理位置。

* Partition: 逻辑子表，对于处在同一个TableSpace的表数据，根据相同的partition规则横向划分为多个子表。

* ObServer: 又称PartitionServer，将chunkserver/mergeserver/updateserver融合后的server，承担若干个Tablet的增删查改服务。

* SSTable: Tablet的物理形式，基准数据的存储方式，是一种按主键(rowkey，也称为primary key)排序的文件格式。

<span id="design_goal"></span>
###  3 设计目标 
实现对partition的复制迁移功能。

<span id="design_idea"></span>
### 4 设计思路及折衷
RootServer发起复制迁移任务，将任务加入目的ObServer的任务队列中。当任务被调度后，执行完成后，通知RootServer任务执行状态，并将任务从队列中移除。

RootServer发起的复制迁移任务对象是某个指定的分区。分区的内容包括静态数据和动态数据。其中，静态数据从某个指定的分区从某源ObServer复制迁移到某目标ObServer上，包括基准数据和可能的转储数据。动词态数据从此分区的ObServer Leader上同步静态数据之后更新日志。 

为了保证系统的稳定性，源ObServer不能是此分区的Leader。如果是， 则需要先进行Leader的切换，再从当前非Leader复制指定的分区。

复制迁移过程中，如果源或目的ObServer宕机，RootServer宕机等异常，尽可能地做断点续传功能。

<span id="system_design"></span>
### 5 系统设计

<span id="summary"> </span>
#### 5.1 基本介绍 
本小节介绍复制迁移功能在ObServer上的流程逻辑，以及主要的接口。

<span id="achitecture_and_notes"> </span>
#### 5.2 系统架构图及说明
RootServer(rs)控制 Partition(pa) 复制和迁移的主要逻辑：

－step1. rs发起复制迁移任务，通知目的节点ObServer2(obs2)从源节点ObServer1(obs1)复制分区pa的数据

－ step2. obs2从obs1拉pa内的基准数据及可能的转储数据

－ step3. obs2从获取与pa相关联的日志点，记录为sync_log_id

－ step4. rs通知leader执行副本下线流程，将obs1的pa从paxos组中删除（只针对迁移），更新元分区表信息，同时执行副本上线流程，将obs2的pa作为leader的paxos参与者参与日志同步，并参与日志同步投票。

－step5. obs2向leader主动拉从sync_log_id开始的日志。

－step6. 如果step5追赶上日志，则流程完成

其中，有几点需要解释下：

1. step4中，leader被动发老日志与主动发新日志并发进行，如果新日志因落后太多而无法落入副本的滑动窗口，则直接丢弃。因为老日志发送速度快，迟早新日志可以放入窗口内。在此期间日志可以参与投票。

2. 在step4中，因为目前paxos的实现存在缺陷，暂时定为先下线再上线的逻辑。在副本较少时，这里存在一个可能单点的风险点。

3. 对于迁移，在step5完全追赶上leader的日志后，无需要主动清理obs1中的多余的数据及memtable中的缓存，在partition的核对删除中会完成这个工作。


更详细的流程可以参考下面的调用序列图:
![c813f45fc5686d817c1ceed3c6b1eadd](http://gitlab.alibaba-inc.com/uploads/xiyu.lh/mysql/b278516d9c/c813f45fc5686d817c1ceed3c6b1eadd.png)

<span id="outer_API"></span>
#### 5.3 与外部系统的接口 

复制迁移的核心逻辑在存储层，需要与rs及obs的日志模块交互。

* ObServer与RootServer交互的接口

－接收rs发的复制迁移消息

－通知rs任务的执行结果

* 存储模块与日志模块交互的接口

－通知leader将obs中pa下线及上线

－从leader中拉指定pa中某位点后的日志

<span id="detail"> </span>
#### 5.4 详细介绍

下面详细描述复制迁移的控制逻辑，发送的数据等。

1. 控制逻辑

* 发起
rs触发复制迁移的逻辑由rs模块实现，在此不讨论。
* 接收
obs接收到rs发起的复制迁移消息后，将消息保存在一个消息队列中，然后给rs发收到响应。
* 处理
obs内由另外的调度线程分配消除队列中的消息给工作线程来实现具体的任务，结果为成功或失败。
* 完成
obs中的工作线程完成任务后，通知rs的最终结果，并从消息队列中移除此任务。

2. 分区pa相关的数据

分区pa在obs2的格式为一个个的宏块，需要其中实际需要的将这些宏块依次序列化发给obs1，然后obs1再反序列化将数据重新写入本地pa文件的宏块中。

* PartitionMeta 此部分包含当前partition所索引到的宏块信息。此部分在obs2端几乎被重写，可能只需要发送部分必要信息。
* SSTable 将实际的数据按行排序存储，包括header/data/index。
* MacroBlockMeta 此部分包含当前宏块中使用到的： 

－ compressor
存储宏块数据所采用的压缩方法名称。

－ schema
当前宏块所存储的表中的列id。

－ endkey
宏块数据按行排序存储，endkey标记当前宏块中索引到的endkey。

－ 其它
宏块中行数，checksum, data/index/endkey的offset等。


<span id="exception"></span>
#### 5.5 异常处理
* 复制迁移的断点续传
复制与迁移的过程中，最耗时的阶段为从obs1复制pa到obs2。如果这个过程任何一方机器宕机，rs会认为此轮复制迁移未完成，在下一轮中重新发起复制迁移任务。
如果rs新发起的复制迁移任务还是从复制pa到obs2，为了节省资源，避免再次重传pa的数据，可以在obs2上记录上一轮迁移的任务以及状态。例如，在任务开始时，记录已经复制的sstable相关的space_id/table_id/partition_id/macroblock_id/macroblock_offset，等整个复制迁移完成后再删除。在obs2上记录的上一轮迁移任务及状态暂时不计划持久化，因此如果obs2宕机，则不会续传。
对于obs1宕机后rs发起新的一轮复制迁移任务，如果不是从继续复制迁移pa到obs2，则对于上一轮中obs2上已经复制遗留的sstable文件，不用特殊处理，会在每日合并时删除。

* 分区表分裂与复制迁移 
复制与迁移过程中，如果有分区表发生分裂，例如pa分裂为pa1和pa2， 对复制迁移任务无任何影响。因为分裂是针对后续的某个冻结的大版本而言，会在每日合并时发起。

* 每日合并与复制迁移 
复制与迁移过程中，如果obs1正在对分区进行每日合并，则中断任务，由rs发起下一轮复制迁移。因为每日合并完成后，可能将正在复制迁移的pa视为老版本而将其回收。

* obs2在复制迁移中宕机恢复日志
只需要在整个pa复制迁移成功才记一个日志到obs1中。其它情况无须关心日志。此时日志中仍然可能存在宏块相关的日志，但如果宕机，因为没有完成标记，日志不会被应用。

### 6 模块 <span id="module1"></span>

#### 6.1 基本介绍<span id="module1_summary"></span>
#### 6.2 系统架构图及说明 <span id="module1_achitecture_and_notes"> </span>
#### 6.3 与外部系统的接口<span id="module1_outer_API"></span>
#### 6.4 详细介绍<span id="module1_detail"> </span>
#### 6.5 异常处理<span id="module1_exception"></span>

###  7 兼容性考虑 <span id="compatibility"></span>
#### 7.1 SQL功能<span id="SQL"></span>
#### 7.2 老版本升级<span id="upgrade"></span>

### 8  DBA考虑<span id="design_for_DBA"> </span>
#### 8.1 SQL功能<span id="limitation"></span>
#### 8.2 老版本升级<span id="upgrade"></span>
#### 8.3 SQL功能<span id="error"></span>

### 9 测试考虑（可选）<span id="design_for_TEST"> 
### 10 设计评审意见<span id="design_feedback"> </span>
### 11 附件及参考资料<span id="design_reference"> </span>
