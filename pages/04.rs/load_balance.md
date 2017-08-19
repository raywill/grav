# OceanBase 1.0 Load Balance

OceanBase 1.0 一期实现中，租户可购买多个资源单位，称为单元(unit)。
负载模块椐此分为两层:

1. partititon在单元内部的均衡
2. 单元在服务器上的负载均衡

本文描述此两层结构的管理及负载均衡实现。

## 名词解释

**UNIT (单元)**: 服务器部分资源集合的抽像，如表示为： 1 CPU + 4GB MEMORY + 200 IOPS。
  单元可在创建租户指定，也可单独购买，通过我们对外的售卖模型将用户购买的资源转化为单元以及单元的数量。

**UNIT CNFIG(单元规格)**: 指此单元按什么样的资源组成，如一个单元的规格可以为：
  1 CPU + 4GB MEMORY + 200 IOPS，也可以是： 2 CPU + 16 GB MEMORY + 400 IOPS。

**RESOURCE POOL(资源池)**: 相同规格的单元集合，可以包含多个zone的多个单元，一期实现中，限制每个zone中
  单元数一样。一个租户可以有多个resource pool，resource pool的单元数，以及是什么规格的unit都可调整。

**TEMPORARY OFFLINE(临时下线)**: 机器lease过期一段时间(分钟级别)后视为临时下线，RS为受影响的partition创建临时副本
，以使得日志形成多数派时间不受影响。

**PERMANENT OFFLINE(永久下线)**: 机器lease过期较久之后(小时级别)，认为机器数据不能恢复，RS为受影响的partition复制副本，保证数据副本数足够。

**NORMAL REPLICA(正常副本)**: 正常的partition副本，包含完整的数据，且日志保持与leader同步。

**TEMP REPLICA(临时副本)**: 正常副本下线后的增加的只记录日志的临时副本，日志与leader同步，不回放memory table，无静态数据。

**TEMP OFFLINE REPLICA(临时下线副本)**: 机器临时下线后，其上面的副本变为临时下线副本，有静态数据，但日志与leader不两步。

## Replica管理

OB Server将所有副本汇报到partition table中，每个partition的副本，需副本所在server自己汇报(leader汇报时
特殊一些，可能会更新其它副本的role)。

RS根据以下方法识别副本partition table中副本类型：

- 正常副本：有静态数据(data_version > 0)，在leader成员列表中
- 临时副本：无静态数据(data_version == 0)，在leader成员列表中
- 临时下线副本：有静态数据(daate_version > 0)，不在leader成员列表中


副本状态变化如下：


```nohighlight


                   migrate_replica(out) && replica gc
   +-----------------------------------------------------------------+
   |                                                                 |
   v                                                                 |
 +----+            create_partition/migrate_replica(in)            +--------------+
 |NONE|----------------------------------------------------------->|normal replica|
 +----+                                                            +--------------+
  ^ | ^                                                              ^ | ^
  | | |   remove_replica                                             | | |
  | | +--------------------------------+                             | | |
  | |                                  |                             | | |
  | |     add_temporary_replica    +---+--------+  add_replica       | | |
  | +----------------------------->|temp replica|--------------------+ | |
  |                                +------------+                      | |
  |                                                                    | |
  |                                                                    | |
  |       replica gc        +--------------------+    remove_member    | |
  +-------------------------|temp offline replica|<--------------------+ |
                            +--------------------+                       |
                               |                                         |
                               |   add_replica                           |
                               +-----------------------------------------+

```

replica gc是指partition manager中回收孤儿partition的过程。


## Server管理

RS通过白名单的方式管理集群的server (记录到\_\_server表)，白名单通过两个方式指定：

1. bootstrap时指定的rootservice列表
2. 通过alter system add server命令增加

OB Server通过心跳向RS汇报自己的资源(cpu, memory, disk ...)，RS椐此进行单元分配。当处Ob Server的
cpu数和内存大小是自动获取的，在测试时可通过node\_cpu\_count 和 memory\_size\_limit分别指定一个固定大小。

另外，可能过： \_\_all\_virtual\_server\_stat 表查server资源分配情况。

## 单元管理

RS通过内部表\_\_all_unit维护单元信息，表结构如下：

```sql
CREATE TABLE `__all_unit` (
  `gmt_create` timestamp(6) NULL DEFAULT CURRENT_TIMESTAMP(6),
  `gmt_modified` timestamp(6) NULL DEFAULT CURRENT_TIMESTAMP(6),
  `unit_id` bigint(20) NOT NULL,
  `resource_pool_id` bigint(20) NOT NULL,
  `group_id` bigint(20) NOT NULL,
  `zone` varchar(128) NOT NULL,
  `ip` varchar(32) NOT NULL,
  `port` bigint(20) NOT NULL,
  `migrate_from_ip` varchar(32) NOT NULL,
  `migrate_from_port` bigint(20) NOT NULL,
  `temporary_unit_ip` varchar(32) NOT NULL,
  `temporary_unit_port` bigint(20) NOT NULL,
  PRIMARY KEY (`unit_id`)
);

```

unit有三个地址:ip, migrate\_from\_ip, temporary\_unit\_ip正常情况下只有ip有效。unit迁移时设置migrate\_from\_ip
ip为源地址，需要生成临时副本时设置temporary\_unit\_ip为临时副本位置。OB Server上租户资源分布根据租户的
unit分布进行。unit的三个地址所在的OB Server都会分配此unit规格的资源。即：当unit发生迁移时，此unit的资源
会在新老机器都分分配。

**UNIT GROUP**: RS将一个resource pool的不同zone的unit编了一个组，一个unit group中任意两个unit不能
在同一zone中。unit group存放一个partition的所有副本。

副本属于哪个unit只在partition table(\_\_all\_root\_table, \_\_all\_meta\_table)中记录，OB Server的
partition meta不记录此信息。RS为维护此信息，会在创建副本之前(包括建表，迁移，增加副本等一切生成
新副本的动作)向partition table中插入一条标记副本(FLAG REPLICA)，标记会在此unit新增副本。标记副本
data_version为-1，由副本创建成功后，汇报partition table时将些记录覆盖。

## 单元均衡

根椐单元的配置与及机器配置，可通过线性规划的方法达到比较理想的单元分布情况。
**一期实现中我们只考虑单元规格中的CPU值和MEMORY值。**

### 单元分配

在创建resource pool时需要在zone中选择机器分配单元。如果机器下线或长时间无心跳，
也要在zone中重新分配不在线机器的单元。

先定义两个限制：

- **机器利用率软限制(resource soft limit)**：表示机器理想负载，当机器分配的资源小于 机器资源 * soft_limit时，
  即使有更空闲的机器，也不会将单元迁到空闲机器。
- **机器利用率硬限制(resource hard limit)**：表示机器最大负载，分配单元不会使机器利用率超过此值。

单元分配按如下方法进行：

1. 租户的任意两个unit不允许分配在一台机器上
1. 以Best Fit原则在未达到软限制的机器中，一个一个的分配租户购买的单元。
   如所有机器都是16个CPU，soft_limit = 100%，机器A已分配11 CPU, 机器B, C已分配0 CPU
  - 如租户申请4 CPU的单元2个，则分配顺序为A, B。最终一个分配在机器A，一个分配在机器B上。
  - 如租户申请2 CPU的单元3个，则分配顺序为A, B, C
1. 如果在未到软限制的机器中分配不出单元，则在最空闲的机器上分配
   (假设机器配置都一样，则在已分配CPU数最少的机器上分配)。
   如是会导致机器利用率超过hard limit，则不能分配，报错。

分配完成后，只需更新\_\_all_unit表，Partition均衡模块会分配或迁移partition到单元中。

### 单元Load均衡

1. 当一些机器超过soft limit，而另一些机器未达到 soft limit 时，尝试迁移部分 unit 到未达到 soft limit机器
2. 当所有机器都都超过soft limit，且机器间cpu分机比例之差超过一定阈值时，则尝试从分配比例高的机器迁移部分
   unit到最低的机器，使其小于此阈值。

单元均衡在各zone间分别进行，不能破坏上面的分配原则。

### 临时下线支持

unit ip随机器状态变化如下 (A, B, C, D 等表示机器ip)：

- (migrate_from_ip = NONE, ip = A, temporary_unit_ip = NONE):
  - A 临时下线，选择机器B存临时副本：(migrate_from_ip = NONE, ip = A, temporary_unit_ip = B)
  - 负载均衡将A迁移到B：(migrate_from_ip = A, ip = B, temporary_unit_ip = NONE)

- (migrate_from_ip = ANY, ip = A, temporary_unit_ip = B):
  - A由临时下线变为永久下线：(migrate_from_ip = A, ip = B, temporary_unit_ip = NONE)
  - A由临时下线改为上线：(migrate_from_ip = ANY, ip = A, temporary_unit_ip = NONE)
  - B临时下线，选择机器C存临时副本：(migrate_from_ip = ANY, ip = A, temporary_unit_ip = C)

- (migrate_from_ip = A, ip = B, temporary_unit_ip = NONE):  (A向B迁移过程中)
  - A临时下线，什么ip都不变
  - B临时下线，选择机器C存临时副本：(migrate_from_ip = A, ip = B, temporary_unit_ip = C)

temporary_unit_ip选择按上面的unit分配方法进行。

partition的副本中，如果已存在临时副本。此时如果其它副本所在机器又下线，则应该立即复制
静态数据和日志，而不是再创建临副本。为满足此条件，unit manager在发现一个租户在其它zone有临时unit，
而本zone的unit所在机器又下线时，应选择zone做unit迁移，而不是再次创建临时unit。

## Partition均衡

Partition均衡是指租户拥有的单元内，调整partition分布使得单元的负载比较均衡。
Partition均衡方法跟0.5中tablet均衡方法类似，采用简单的贪心法调整partition分布，
使得租户单元负载更为均衡。这里**单元负载**可跟上面一样累加上面的的parition负载得到，
一期我们假设同一租户所有单元都是相同配置，这里采简单实现：**单元负载 = 单元的磁盘使用率**。
即，实际上是按磁盘在做均衡。

**PARTITION GROUP**: 属于同一 table group 的，相同 partition_id 的 partition。

Partition Balance 按以下原则进行：

1. 一个 partition 的所有副本应在一个 unit group 中
1. 同一 partition group 的 partition 放到同一单元内。
1. 同一 table group 的不同 partiton group 应尽量分布到不同单元上。

Partition 均衡通过 partition 分配，复制，迁移完成。OB Server 以 partition 为单位执行迁移复制，
但任务是以 partition group 为单位生成， 一个 partition 同时只允许一个复制或迁移任务在执行。
迁移复制任务保存在 RS 内存中，发生 RS 切换或重启等场景，RS 会重新生成任务。一个迁移/复制任务完成后，
OB Server 会通知 RS，RS 将任务删除并触发其它任务执行。

### Partition分配

表创建时按如下方法分配 partition:

1. 如果存在相同partition group的其它partition，则选与它一样的unit。(保证同partition group的partition在一起)
1. 在zone的unit列表中随机选择一个，并以此为起启地址采用roud robin方法分配。
   (这里没有按partition复制的方法选择，是为了避免遍历租户的所有partition，导致建表过慢。
   同时依赖于partition均衡来使其达到均衡)。

如果建表时指定了primary zone，刚选primary zone为初始化leader所在位置。如未指定则选择租户的
__all_dummy表leader所在zone。

在因机器下线或unit删除等导致partition副本不够情况下，需要选择一个单元在上面复制出一个partition副本。
按如下顺序选择单元：

1. 如果存在同partition group的partiton在此zone的单元上，则选择此单元
1. 选同table group中partition group最少的单元，相同情况下(如都为0)，选负载最低的

### Partition Rebalance

partition均衡以租户为单位执行，一次遍历一个租户所有partition table生成负载均衡任务。

**STANDARD ZONE**: partition所在的unit group以standard zone副本所在的unit group为准。当其它副本unit
group与strandard zone副本所在unit group不一致时，则迁到standard zone副本所在unit group。
当前standard zone的选择方法是选zone name最小(strcmp)的zone。

partition负载均衡主要按以下顺序执行：

1. 保证partition副本足够：生成临时副本，上线临时副本，复制副本等
1. 保证partition group副本在一起：将属于同一partition group副本在起
1. 执行unit迁移
1. unit间负载均衡

具体步骤如下：

1. 临时副本创建: 如果副本所在机器不在线，且其所在unit设置了temporary_unit_ip，则将副本下线，并在
   temporary_unit_ip上增加临时副本
1. 上线临时副本: 如果副本为临时副本，其所在server在线，且与unit.ip相同，则将临时副本上线
1. 上线临时下线副本: 临时下线副本所在机器在线，且本zone中有对应的临时副本，则下线临时副本并
   上线此临时下线副本
1. 如果partition副本数小于应有副本数(schema.replica_num)，则找到没有副本的zone，选择一个unit复制副本。
   unit选择方法为：
  - 如果存在同partition group的partiton在此zone的单元上，则选择此单元
  - 选同table group中partition group最少的单元，相同情况下(如都为0)，选负载最低的
1. partition group member统一：遍历所有partition group，如果存在同一partition group member不统一，
   且任务列表中不存在此partition group的任务，则发起迁移任务：将partition迁到此zone partition group
   中table_id最小的partition所在机器。(partition group中table_id最小的partition称做primary partition)
1. 执行unit迁移：如果副本所在server与其unit所在server不一致，
   则将此unit迁到单元所在server。如所有partition地址与单元地址一致，则将\_\_all_unit表中
   migrate_from_ip, migrate_from_port清空。
1. 迁移到同一unit group: 如果partition副本与standard zone处于不同unit group，则将其迁移致该unit group
1. standard zone执行partition group分散：如有unit1, unit2，一个table group中有两个
   partition group: pg1, pg2，如果pg1, pg2都在unit1上，则将pg2迁到unit2，以达到partition group分散的目的。
   主要针对场景为新的资源(unit)加入
  - 计算此table group中partition group在单元上的分布
  - 存在最大partition gorup数 > 平均 partition group数 + 1，则从其中选择partition group迁到
    partition group数最少(相同数目下选负载最低)的unit上。
1. standard zone执行单元负载平均(unit使用磁盘平均)：如zone中有unit1, unit2，对应磁盘使用比例为 80%, 20%，平均磁盘使用率为
   50%，配置的不均衡忍耐度(unit_load_tolerance_percent)为10%。
   则会从unit1中选择部分partition group迁到unit2，unit磁盘使用率降低到 60% 以下。
   在此过程中不能破坏partition group数量的均衡。
  - 随机打散partition group
  - 遍历partition group，如副本所在unit磁盘使用率 > 平均磁盘使用率 + unit_load_tolerance_percent，
    且此unit上此table group的partition group数 > 平均partition group数。则选此unit做为源unit。
  - 选择源unit所在zone中所有unit，中partition gorup数 < 源unit partition group数，且磁盘使用率最低的做为目的unit。
  - 如果从源unit迁移此partition group到目的unit不会导致目的unit磁盘使用率 > 平均磁盘使用率 + unit_load_tolerance_percent。
    则从源unit迁移此partition group到目的端unit。

复制迁移任务的源端选择时，会尽量选同一zone中的副本做为源。

## 任务调度

RS在调度负载均衡任务时，遵从以下限制：

1. 一个partition同时只有一个任务在执行
1. 集群内所有迁移复制任务数不超过concurrent_data_copy_task_limit
1. 同时从一个server复制数据的任务数不超过server_copy_in_task_limit
1. 同时往一个server复制数据的任务数不超过server_copy_out_task_limit

任务执行超过rebalance_task_timeout未返回的，则认为任务执行超时，结束任务。

RS生成的等待被调度以及正在行的任务可以通过__all_virtual_rebalance_task_stat表查看。

OB Server如果执行迁移任务发现此OB Server因不可自行恢复的错误(磁盘不可写入，日志满等)导致迁入不成功，则返回特定错误号。
RS收到此错误号后，将server的迁入block一段时间，负载均衡对block的server做如下调整：

1. 正在迁往此server的unit迁一个其它目的地
1. 不再发起向这个server的任何迁移复制任务
1. 因server block导致的partition group副本不在一台机器的partition group，将partition group迁到非block的机器。
   如一个zone的某partition group内有2个副本，此partition group正在从A迁到B，第二个partition迁移时server被block，
   则将第一个副本再迁由B迁回A，保证partition group副本在一起。
1. 如上，如果因server block导致同个partition的多个副本不在一个unit group，则将standard zone的副本迁到没有
   server被block的unit group上。

## 人工干预

1. 关复制: (不做临时副本，副本补全等操作)

```sql
alter system set enable_rereplication = 'false';
```

1. 关迁移: (不做load均衡)

```sql
alter system set enable_rebalancing = 'false';
```

1. 手动发起副本迁移复制:

```sql
alter system move replica partition_id = 'partition_idx%partition_cnt@table_id' source = 'ip:port' destination = 'ip:port';
alter system copy replica partition_id = 'partition_idx%partition_cnt@table_id' source = 'ip:port' destination = 'ip:port';
```

1. 手动发起unit迁移: 手动改unit的ip地址，再 reload unit:

```sql
update __all_unit set migrate_from_ip = ip, migrate_from_port = port where unit_id = xxx;
update __all_unit set ip = dest_ip, port = dest_port where unit_id = xxx;
alter system reload unit;
```

