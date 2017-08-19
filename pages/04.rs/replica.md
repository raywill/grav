# RS支持多类型副本的方案

## 1. 目的
目前（OB 1.2），对于分区的每一个副本，都包含日志，MemTable, SSTable等完整的功能。在三个副本的情况下，实际上只有leader提供服务，另外两个“全功能”的副本具备随时切换为leader的能力，但是在正常情况下不会发生，这样我们比单副本付出了三倍的内存和存储成本。

实际上，作为容灾，我们需要三个paxos选举组成员（只需要日志服务，不需要存储服务），而只需要一个全功能副本能够随时切换为主即可。为此，我们引入一种新的副本类型，暂且把它叫做日志型副本。对于“1：1：1“型三副本的部署，以后可以选择2个全功能副本加一个日志型副本。对于”2：2：1“型两地三中心的部署，可以选择”（全+日）：（全+日）：（全）“的配置方案，从而在可用性和成本之间取到较好的平衡。

基于类似的考虑，我们会引入几种新的副本类型，以支持不同业务在在数据安全，性能伸缩性，可用性，成本等之间进行取舍折中。详见下一节。

同时，为了实现“两地三中心”的方案，我们需要打破一个zone只能有一个副本的限制。同时，我们还要去掉unit同构的限制。同时，负载均衡的时候还需要感知docker容器以保障可用性。

## 2. 副本类型

代码实现上定义ObReplicaType枚举类型来表示多种类型的副本。ObReplicaType中依次使用2 Bits描述Memstore的状态，2 Bits描述SSStore的状态，4Bits描述CLOG的同步状态，ObReplicaType的定义以及Memstore，SSStore，Clog各状态值如下：   
  
|4 Bits|2 Bits|2 Bits|          
|:----:|:----:|:----:|  
|CLOG STATUS|SSSTORE STATUS|MEMSTORE STATUS|              
其中Memstore Status在最低比特位方向，然后依次为SSStore Status，Clog Status在最高比特位方向。


 
> Memstore Status       

|MEMSTORE STATUS|VALUE|       
|:-------:|:----:|     
|WITH_MEMSTORE|0|
|WITHOUT_MEMSTORE|1|

> SSStore Status     

|SSSTORE STATUS|VALUE|       
|:-------:|:----:|     
|WITH_SSSTORE|0 << 2|
|WITHOUT_SSTORE|1 << 2|

> Clog Status

|CLOG STATUS|VALUE|       
|:-------:|:----:|     
|SYNC_CLOG|0 << 4|
|ASYNC_CLOG|1 << 4|


当前支持的副本类型如下：       

* 全能型副本
  也就是目前支持的普通复本，拥有事务日志，MemTable和SSTable等全部完整的数据和功能。它可以随时快速切换为leader对外提供服务。
* 备份型副本
  不实时回放日志的副本，也就是没有MemTable的复本。在必要时，它可以通过回放日志建立MemTable，并切换为leader对外提供服务。
* 日志型副本
  只包含日志的复本，没有MemTable和SSTable。它参与日志投票并对外提供日志服务，可以参与其他复本的恢复，但自己不能变为主提供数据库服务。
* 只读型副本
  包含完整的日志，MemTable和SSTable等，但是它的日志比较特殊。它不作为paxos成员参与日志的投票，而是作为一个观察者实时追赶paxos成员的日志，并在本地回放。这种副本可以在业务对读取数据的一致性要求不高的时候提供只读服务。因其不加入paxos成员组，又不会造成投票成员增加导致事务提交延时的增加。

可以用以下表格看出这几种副本的特性。       
<table>
<tr><td>类型</td><td>LOG</td><td>MemTable</td><td>SSTable</td><td>数据安全</td><td>恢复为leader时间</td><td>资源成本</td><td>服务</td><td>名称(简写)</td><td>副本类型值</td>
</tr>
<tr><td>全能型</td><td>有，参与投票(SYNC_CLOG)</td><td>有(WITH_MEMSTORE)</td><td>有(WITH_SSSTORE)</td><td>高</td><td>快</td><td>高</td><td>leader提供读写，follower可非一致性读</td><td>FULL(F)</td><td>0</td>
</tr>
<tr><td>备份型</td><td>有，参与投票(SYNC_CLOG)</td><td>无(WITHOUT_MEMSTORE)</td><td>有(WITH_SSSTORE)</td><td>高</td><td>慢</td><td>中</td><td>不可读写</td><td>BACKUP(B)</td><td>1</td>
</tr>
<tr><td>日志型</td><td>有，参与投票(SYNC_CLOG)</td><td>无(WITHOUT_MEMSTORE)</td><td>无(WITHOUT_SSSTORE)</td><td>低</td><td>不支持</td><td>低</td><td>不可读写</td><td>LOGONLY(L)</td><td>5</td>
</tr>
<tr><td>只读型</td><td>有，但不属于paxos组，只是listner (ASYNC_CLOG)</td><td>有(WITH_MEMSTORE)</td><td>有(WITH_SSSTORE)</td><td>中</td><td>不支持</td><td>高</td><td>可非一致性读</td><td>READONLY(O)</td><td>16</td>
</tr>
</table>

今后，还可能添加其他副本类型。

* 增量型副本
  包含实时回放的日志和MemTable，但是没有SSTable。例如收藏夹的item表的读副本。当然也可以存储sstable，但目前item表已经达到0.45TB，有一定的额外成本；
* 冷备型副本
  包含SSTable和异步日志。可能用于冷备的实现。出于成本的考虑，冷备型副本所在机器内存较小，在冷备上产生Memtable几乎不可能。
  


## 3. 用户接口
建表的时候，目前使用`replica_num`表示副本数。为了兼容，`replica_num`保持含义不变，表示“全功能”副本个数。为了支持新增的3种副本类型，新增三个选项：

  * `logonly_replica_num`表示日志型副本数，不指定的时候，缺省值为0；
  * `backup_replica_num`表示备份型副本数，不指定的时候，缺省值为0；
  * `readonly_replica_num`表示只读型副本数，不指定的时候，缺省值为0。

所有四种副本数的和，是这个表的分区的总副本数。除了`readonly_replica_num`外，其余三个数的和，是paxos成员组大小，他们组成paxos选举成员列表。

`primary_zone`的含义不变，用于表示leader的偏好位置。leader必须是一个全功能副本。引入一个新的可选的参数`locality`，可以指定每个副本的位置分布。这个参数不指定的时候，系统根据一定的策略自动选择副本位置。

### 3.1 Locality描述语言
`locality`属性用来描述一个分区的多个副本的分布策略。`locality`属性具有层次继承关系。如果partition不设定，继承table的值；如果table不指定，继承database的值；如果database不设定，继承tenant的值。

#### 3.1.1 自动策略
`locality`设定为`auto`的时候，系统会根据副本数和租户zone资源等信息，选择一定的策略自动选择分布策略。

#### 3.1.2 详细设定的语法
基本语法结构型如"replicas@location"。由如下一些元素组成。

  1. 副本类型：`F`表示“全功能”副本；`L`表示“日志型”副本；`P`表示leader主副本，它也是一种`F`；`R`表示任意类型副本。全部类型名称见上节中的表格中的`名称`列，可以用全名或者简写。
  2. 位置：位置是系统已知的一组枚举值。位置有两类，一种是zone的名称，如`hz1`, `bj2`等；另一种是地域名称，如`Beijing`, `Hangzhou`。一个zone必然属于某一个地域。此外，还有两个特殊的位置值，`anyzone`表示任意zone，`anyregion`表示任意地域。
  3. 量词：`+`表示1到任意多个；`*`表示0到任意多个；`?`表示0或1个；`{min,max}`表示最少min个，最多不超过max个；`{min,}`表示最少min个；`{,max}`表示最多不超过max个；`{n}`表示严格地n个。

其中，`P`类型副本可以用来代替`primary_zone`的功能，并且有更灵活的语义，比如指定leader位于某个地域而不要求zone。特别需要指出的是，**设定的时候无需给定所有全部副本的位置**，只需要列出用户认为必须满足的分布约束条件，未指定的条件系统自动决定(相当于locality值最后有一个默认规则“R*@anyzone”)。下面举一些例子说明用法。

  * "F,L@hz1, P,L,L@hz2, F@sz1"，表示hz1这个zone一个`F`一个`L`，hz2这个zone里一个`P`两个`L`；
  * “F{1,2}@Beijing, L{2}@sz1, R*@Shanghai”，表示Beijing区域部署1到2个`F`副本，sz1这个zone部署2个`L`副本，其余副本部署在Shanghai。

#### 3.1.3 使用策略名称
`locality`的引入给了用户和管理员很大的控制空间，以便满足不同业务对数据局部性和可用性多变的需求。但是使用上面的语法设定起来很繁琐，系统会内置很多常用规则可以满足日常的需求。比如对于`replica_num`为3，`logonly_replica_num`为2，租户zone为3的情况，系统自动选用“两地三中心”的部署方案，即"F,L@region1, F,L@region1, F@zone3"。对于这样特定的自动分配策略，系统会选择最优的策略，用户为了明确起见，还可以用策略名称作为`locality`的值。比如“3zones-in-2regions”。

## 4. Server management
机器注册的时候，需要汇报自己所属的docker host的IP。如果不在docker容器中，
host ip等于自身ip。

zone的属性里需要记录地域region，比如hz1这个zone位于Hangzhou。

## 5. Partition table
replica副本类型可能需要修改或重新定义。PartitionTableOperator在指定`table_id`, `partition_id`从partition table获取副本信息时，会给每一个副本指定一个副本类型，当前的副本类型有如下几种：

  * `OB_NORMAL_REPLICA`：在`leader member_list`中并且`data_version`不为0的副本，通常是一个普通副本。
  * `OB_TEMP_OFFLINE_REPLICA`：不在`leader member_list`中但`data_version`不为0的副本，通常是迁移时加副本成功但成员变更失败的副本。
  * `OB_TEMP_REPLICA`：临时副本(用`data_version==0`标识)，临时副本概念目前已删除。
  * `OB_FLAG_REPLICA`：flag表示，创建partition前插入partition table的一个标识(用`data_version == -1`表示)。   

引入日志型副本后需要增加一种副本类型(例如OB\_CLOG\_ONLY\_REPLICA)，并在partition table中给予日志型副本一个标识，考虑在\_\_all\_root\_table表和\_\_all\_meta\_table表中分别加入一列replica\_type，用以标识副本状态，\_\_all\_meta\_table已经支持ddl操作，\_\_all\_root\_table目前不支持ddl操作，需要通过修改\_\_all\_core\_table表给\_\_all\_root\_table加列。原来临时副本相关逻辑不再使用，都会删除。
 
#### 5.1 其他系统表及二级分区表
`__all_table`里需要添加`logonly_replica_num`和`locality`列。同样地，二级分区记录分区元信息的`__all_part`, `__all_phy_part`和`__all_part_ops`等也需要添加这两个属性。

## 6. Bootstrap

在多副本的实现中，内部表默认的replica_num改成5，并且全部是全功能副本。

具体修改时，bootstrap的阶段不需要做改动，等到负载均衡阶段，检测内部表的副本数，如果副本数小于5，则尝试进行增加副本操作，复制时需要满足两个条件：

  1. 同一个replica在同一个物理机上面不能有两个副本；
  2. 同一个replica在一个zone上面的副本数不能超过2；可以将这个规则作为另外一种`locality` 的策略，即`F{,2}@anyzone`，内部表都采用这种策略。

副本复制时，需要先保证对应机器上有系统租户的unit，如果没有，则需要先创建；最终会出现整个集群中有5个系统租户的unit，内部表partition的5个副本分别位于这些unit上。

## 7. Major freeze
当前的major freeze采用两阶段提交的方式实现，两阶段提交的协调者由RS充当，集群中的若干observer作为两阶段提交的参与者。如果存在日志型副本升级成全功能副本的机制，major freeze无需进行改到；如果日志型副本不会升级为全功能副本，major freeze可能进行存在如下优化：

* 发起major freeze时，如果集群中存在inactive的observer，RS会检查inactive的observer是否为空server(空server指不包含任何partition副本)，major freeze可以忽略掉空server而仍然保证正确性，类似的思路可以延伸到日志型副本的情形下，由于日志型副本不会成为leader，如果inactive的observer上仅包含日志型副本，major freeze可以将这类observer忽略掉，进而可以提高major freeze的成功率。

注：这个优化比较trivial，实现时可以不做。

## 8. Daily merge
合并过程中，我们的调度策略和集群合并顺序，是不需要修改的。还是批量的以zone为单元进行调度，主RS所在的集群最后一个合并。 
 
需要进行的修改：

 * 参与合并的副本  
 单个obs在合并过程中，需要检测副本的类型，如果只是单纯的日志副本的话，则是不需要进行数据合并，直接修改数据版本号即可。这块检查需要严格一些，以后还有可能出现其他类型的副本。必须只能是类型为“F”的全功能副本才能进行合并，否则有可能出错数据错误。  

 * 切主和预热  
 在切主和预热的过程中，需要考虑：主不能切换到非全功能副本上面，非全功能的副本也不需要进行预热。

 * 日志副本版本的提升  
 1.2版本中，每个partition的数据版本上升之后，都需要更新partition表。但是现在多了日志副本，这种副本是不需要真正进行数据合并的，因此这种日志型副本的data_version我们设置成-1，每日合并时不进行版本的提升。

 * 检测单个zone合并完成  
 1.2版本中，会通过完成合并的partition\_cnt的来判断是否合并完成。现在增加了特殊副本以后，只能通过检测全副本的合并状态来判断集群是否合并完成。只有完成合并的全副本的partition\_cnt > percentage && data\_size > percentage，才能判断某个zone是否合并完成。

## 9. Leader coordinate
Leader Coordiante用来控制切主。引入的日志型副本由于没有完成的数据不能提供服务，因此在进行leader coordinate时需要避免将leader切到日志型副本上。因此有以下两点需要注意:

* leader coordinate通过`partition_table_operator`获取各partition的副本后，需要在leader coordinate这边将日志型副本过滤掉，日志型副本的相关信息不应该被作为后续切主的依据，目前leader coordinate已经提供过滤某种类型副本的功能。

* leader coordinate通过RPC向各partition的选举模块获取candidates时，需要选举模块保证返回的candidates中不能包含日志型副本。

## 10. Load balance

#### 副本复制

L 副本的复制可以做得很简单: 在目标 UNIT 上增加一条副本记录，从 MASTER 追赶日志，然后走副本
上线流程，不需要复制 SSTABLE。

#### 副本迁移

当 UNIT 资源不均衡时，L 副本也需要参与迁移。迁移的目的是让 UNIT 资源消耗在租户内均衡，迁移的方法
同 L 副本复制。迁移后，原副本下线。

L 副本和 F 副本之间的数据关系:

 * L 副本始终只会是 L 副本，不会通过复制 SSTABLE 的方式变成 F 副本。
 * F 副本不会退化成 L 副本
 * 可能需要支持从`F`副本复制日志产生新的`L`副本

L 副本迁移策略有如下选择：

 * L 副本获取到资源后立即上线。
 * L 补齐 Major Freeze 后的日志，走成员变更的途径上线，替换原 L 副本：

   * L 副本从源 L 副本获取到足够日志后上线。
   * L 副本从 LEADER 获取到足够日志后上线。

说明：理论上，新的 L 副本只需要获取到运行资源后，就可以立即上线参与新事务的选举。
为了防止 F/L 副本宕机, Major Freeze 后产生的事务日志无法形成多数派，L 可以尽量补齐这些日志，
走成员变更的途径上线。

#### 负载均衡策略的约束条件

 * 位置感知：RS 需要感知 locality 限制，根据 locatlity 指定的参数来调度 L 副本。
 * 副本限制：一个 zone 内可以有多个副本，副本配置由 locality 指定。默认情况下系统内全部为 F 副本
 * docker：同一个partition的多个副本不放在同一个docker host物理机器上。

#### 租户内 Replica 均衡算法

租户内负载均衡算法的目的，是在确定有限的资源总量（unit规格和数量）的前提下，平衡租户拥有的多个unit间的负载。通过这种均衡，消除资源瓶颈和潜在的热点，通过降低unit的负载可以提升性能。同时，还可以提升租户的资源利用率，降低租户的使用成本。

###### 负载度量
一个unit的负载，等于这个unit上所有副本的负载之和。replica均衡的目标是通过迁移replica的手段，使得一个租户的所有unit的负载达到均衡。

考虑以下单一资源指标：

  1. CPU使用率
  2. IOPS
  3. sstable磁盘使用率
  4. 日志写入量
  5. 内存使用率
  6. 网络使用率

一个unit的负载定义为资源使用率的加权平均。

> LOAD_unit = SUM_resource( Weight_resource * ((SUM_replica (Usage_resource)) / Capacity_resource) )

其中，resource为上面的单一资源，他们的权重之和为1。

> SUM_resource ( Weight_resource ) = 1

###### 权重
一个资源的使用率越高，它越关键，其对负载贡献的权重也越高。比如，CPU和MEMORY的利用率为1/2和1/3，则他们的权重W之比为3:2。

每轮负载均衡开始的时候，计算这个租户整体上各种资源的利用率，得到资源权重比并确定权重Weight_resource。

###### 失衡度量
度量各unit负载的标准差，即

> imbalance-metric = sqrt( (1/n * SUM_i( (LOAD_i - AVGLOAD)^2 ) )

###### 均衡算法
和目前一样，从load高的unit选一些副本迁移到load底的unit。选择的标准是，迁移以后，是否可以降低imbalance-metric。

作为一种启发式规则，选择迁移的复本时，根据本轮资源权重，可以确定优先选择某种种类型的复本。比如权重最高的是IOPS，则选择的顺序依次是只读型、leader、全能型、备份型、日志型。因为我们可以统计每一个副本的资源使用情况，可以利用这些数据进行副本的选择。

#### Unit 均衡算法
历史库水位较低，Unit 资源一般比较充足，unit均衡的策略不受影响(目前是静态均衡，暂时不变)。迁移unit的时候，所有类型的副本都需要进行迁移。

#### 消除unit同构限制
之前的负载均衡策略中，为了把replica在租户的unit中进行均衡，需要满足一个约束：如果一个unit里有partition p0, p2的副本，那么必须有另一个zone的一个unit里也有p0, p2的副本。这是因为同一个partition的不同副本需要有相同的内存资源限制，否则可能造成备机回放日志失败。为了实现这点，以前的replica均衡实际是partition均衡，因为不同zone中的partition在unit中的分布是同构的。实现中，先对一个standard zone进行副本均衡，然后其他zone的副本和standard zone同构。

因为我们将要实现minor freeze了，日志回放对内存要求没有强制要求。这样，就可以打破上面的限制，各个zone分别以各自的partition group为单位进行负载均衡，无需考虑partition的复本在各unit中的分布同构。

## 11. 实现
### ObReplicaType
代码实现上定义ObReplicaType枚举类型来表示多种类型的副本。ObReplicaType中依次使用2 Bits描述Memtable的状态，2 Bits描述SSStore的状态，4Bits描述CLOG的同步状态，ObReplicaType的定义以及Memtable，SSStore，Clog各状态值如下：   
  
|4 Bits|2 Bits|2 Bits|          
|:----:|:----:|:----:|  
|CLOG STATUS|SSSTORE STATUS|MEMTABLE STATUS|              
其中Memtable Status在最低比特位方向，然后依次为SSStore Status，Clog Status在最高比特位方向。


 
> Memtable Status       

|MEMTABLE STATUS|VALUE|       
|:-------:|:----:|     
|WITH_MEMTABLE|0|
|WITHOUT_MEMTABLE|1|

> SSStore Status     

|SSSTORE STATUS|VALUE|       
|:-------:|:----:|     
|WITH_SSSTORE|0 << 2|
|WITHOUT_SSTORE|1 << 2|

> Clog Status

|CLOG STATUS|VALUE|       
|:-------:|:----:|     
|SYNC_CLOG|0 << 4|
|ASYNC_CLOG|1 << 4|


### 模块接口 @文铎
ObServer需要给RS提供副本操作的接口，因为新增副本类型，这些接口需要修改。

1. 新建：ObCreatePartitionArg及ObCreatePartitionBatchArg
2. 复制：ObAddReplicaArg
3. 删除：ObRemoveReplicaArg
4. 迁移：ObMigrateReplicaArg


### 内部表 @蓉暄
需要修改以添加属性列`locality`, `readonly_replica_num`等的内部表有：

1. `__all_tenant`
2. `__all_database`
3. `__all_table`
4. `__all_phy_part`

### replica统计信息收集 @竹翁
增加load balance算法所需要的资源使用信息。
#### 实现expontentially moving average
https://aone.alibaba-inc.com/code/D40508

#### replica级别资源使用率指标
1. https://aone.alibaba-inc.com/code/D43531
2. `__all_virtual_partition_info`

http://obfarm.alibaba.net/wiki/#!1/rs/resource_manager.md
### 添加虚拟表 @all

反映资源负载均衡指标当前状态的虚拟表：
1. `__all_vritual_rebalance_tenant_stat`
2. `__all_vritual_rebalance_unit_stat`
3. `__all_vritual_rebalance_replica_stat`

### observer支持L副本 @元启
ObPartition新增replica\_type域， replica\_type持久化在ObBaseStorageInfo中。新建replica（建表或迁移)时需要指定replica\_type,
宕机重启后可以从ObBaseStorageInfo中恢复replica\_type。

L副本的特点:

1. 不能成为leader，不能提供读写服务.
2. 有sstore和memstore，只不过ssstore和memstore都是空的。不需要回放事务日志，但是需要回放major freeze日志; 正常触发merge，merge完成后sstore还是空的； merge完成后正常汇报给RS，RS不做checksum校验。
3. 回放点正常处理: merge完成正常更新回放点到ObBaseStorageInfo; 宕机重启按正常流程确定回放点, clog正常push到滑动窗口，恢复之后滑动窗口和log\_index都会重建好; 回收日志的逻辑也不受影响。全局merge完truncate log_index逻辑不受影响。
3. 迁入L副本时，不用拷贝静态数据， 而是直接新建一个空的sstable。

L副本不需要特殊处理的地方:

1. 核对删除，garbage clean不受影响。
2. 因为L副本是Paxos组成员，所以成员变更正常执行

L副本转成F副本:

1. L副本转成F副本，类似于rebuild F副本，不同的地方在于拷贝静态数据时不比较版本号，而是强制拷贝，并且拷贝完静态数据后，replica\_type更新为F副本。恢复memstore等流程都不用修改。
2. 如果静态数据拷贝失败，则回滚成L副本，如果静态数据拷贝成功，按F副本rebuild失败处理。

