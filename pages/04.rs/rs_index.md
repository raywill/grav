# Index

## 背景

OceanBase按照唯一主键来组织表的数据，当用户需要通过其他维度来检索数据时，就需要全表扫描了。为了加快
检索速度，OceanBase实现了索引功能。

## 名词解释

* 索引：是对数据库表中一列或多列的值进行排序的一种结构, 可用于加快检索速度
* 局部索引：在partition为单位建的索引
* 全局索引：以表为单位建的索引
* 唯一索引：索引列必须是唯一的

## 设计目标

支持创建索引与删除索引，保证索引表跟数据表一致，保证走索引读数据跟走数据表读数据的一致性。
一期只支持局部索引，后续会考虑实现全局索引。

## 设计思路及折衷

### rootserver生成索引表的schema

OceanBase 0.5.2里，索引表的schema是在mergeserver通过数据表的schema和索引列生成的，假如mergeserver1
接收到创建索引的请求并构建好索引表的schema，mergeserver2接受到修改数据表被索引列的schema的请求并发
给rootserver执行成功了，此时mergeserver1再把索引表的schema发给rootserver创建索引则会失败。由于所有
ddl操作都在rootserver执行，rootserver的schema总是最新的,构建索引表的schema由rootserver来执行就能规
避上述问题。

### 索引表数据由存储层在每日合并时构建

OceanBase的索引表并不是立即生效的，必须等到索引表的数据构建好了，索引表才开始生效。Oceanase 0.5.2
中实现的索引是全局索引，数据表的tablet跟索引表的tablet并不是一一对应的，因此构建索引表的数据需要
rootserver进行统筹控制。Oceanbase 1.0一期只支持局部索引，数据表的partition跟索引表的partition是一
一对应的，因此索引表的数据直接由存储层在每日合并时构建，不需要rootserver介入。rootserver检测每日
合并是否完成时同时将在上一次冻结前新创建的索引表的状态设置为Available之后，索引表生效。

### 删除索引表的策略

为了保证读索引表跟读数据表的一致性，在OceanBase 0.5.2中，删除索引表需要先停索引表的读，等一段时间，
然后再停索引表的写。如果索引表同时停读停写，有可能会导致读索引表和读数据表读出来的结果是不一致的。
考虑这样的场景，mergeserver1将数据表table1的索引表index1删掉，但mergeserver2上的schema由于版本比较
老仍认为索引表index1存在，此时一个客户端在mergeserver2上先往table1插入几行数据然后再走索引读新插入
的数据，新插入的数据是读不到的，但不走索引新插入的数据是能读到的。OceanBase 1.0中，读写都只在
partition的leader上进行，正常情况下同时停读停写是不会有问题的，但假如leader发生切换了，而新的主的
schema比老的leader的schema老，上述问题还是有可能出现。

一种方案是执行写事务时，将leader的schema version也通过日志同步到followers，新的主被选出来时如果本地
的schema version比日志里的schema version低，则需要刷新schema到该版本才能开始提供服务。这种方案引入
了同步schema version的开销。

另一种方案是索引表的停读写操作等到major freeze之后才执行，由于执行major freeze需要保证所有observer的
schema都同步到最新版本，上述问题就能规避掉，但在major freeze之前，被删除的的索引表还需要写memtable，
如果写入量大的话，这会浪费大量内存，另外就是用户要创建同名索引的话，也会出错，这可以在删除索引时将
索引表的名字改成一个内部的名字来规避。

删除索引的方案未定，下面删除索引的流程基于方案一实现。

## 系统设计

### 索引表的主键和索引表的状态

OceanBase 1.0中的索引包含以下几种：

```cpp
    enum ObIndexType
    {
      INDEX_TYPE_IS_NOT = 0,         //is not index table
      INDEX_TYPE_NORMAL_LOCAL = 1,
      INDEX_TYPE_UNIQUE_LOCAL = 2,
      INDEX_TYPE_NORMAL_GLOBAL = 3,
      INDEX_TYPE_UNIQUE_GLOBAL = 4,
      INDEX_TYPE_MAX = 5,
    };
```

NORMAL表示普通索引，UNIQUE表示唯一索引，LOCAL表示局部索引，GLOBAL表示全局索引。一期我们只做局部索引。
普通索引的主键由索引列和数据表的主键构成，由于普通索引并不要求索引列的唯一性，所以需要加上数据表的
主键保证索引表主键的唯一性。唯一索引的主键只由索引列构成。举个例子：表A(c1, c2, c3, c4), 其中c1是
主键，如果以(c2, c3)为索引列建普通索引，则索引表的主键为(c2, c3, c1), 如果以(c2, c3)为索引列建唯一
索引，则索引表的主键为(c2, c3)。

Oceanase 1.0中索引表的状态包含以下几种：

```cpp
    enum ObIndexStatus
    {
      INDEX_STATUS_UNAVAILABLE = 0,
      INDEX_STATUS_AVAILABLE = 1,
      INDEX_STATUS_UNIQUE_INELIGIBLE = 2,
      INDEX_STATUS_INDEX_ERROR = 3,
      INDEX_STATUS_MAX = 4,
    };
```

索引表刚被创建时，索引表的状态被置为INDEX_STATUS_UNAVAILABLE，表示此时索引表不可读，但写数据表时
也要在一个事务内同时写索引表的数据；
索引表的数据构建好后，索引表的状态被置为INDEX_STATUS_AVAILABLE, 表明此时索引表可写可读；
如果在构建唯一索引的数据时发现不满足索引列的唯一性，索引表的状态置为INDEX_STATUS_UNIQUE_INELIGIBLE;
如果在构建索引表的数据时出现其他错误，索引表的状态被置为INDEX_STATUS_ERROR;
当索引表的状态为INDEX_STATUS_UNIQUE_INELIGIBLE或者INDEX_STATUS_ERROR时，则需要dba介入把索引删掉。

### 创建索引表

Oceanbase 1.0中创建索引表是异步的，创建索引表的主要涉及下面两个阶段：

* 阶段一：

```nohighlight

            rs receive create index rpc request
                           |
                           v
            generate index table schema,
            status = INDEX_STATUS_UNAVAILABLE,
            create_mem_version = memtable_version
                           |
                           v
             insert index table schema to schema
                           |
                           v
                  response to client

```

* 阶段二：

```nohighlight

    Rootserver:

            rs do major freeze, memtable version will increase by 1
                    |
                    v
            rs start daily merge
                    |
                    v
            all zones are merged
                    |
                    v
        update index table status whose create_mem_version == memtable_version - 1
        and status == INDEX_STATUS_UNAVAILABLE, set status = INDEX_STATUS_AVAILABLE
                    |
                    v
            rs finish daily merge


    ObServer:
           observer start daily merge
                    |
     yes            v
  +--------all partitions are merged  <--------------------------------------------------+
  |                 | no                                                                 |
  |                 v                                                                    |
  |          merge partition                                                             |
  |                 |                                                                    |
  |                 v                                                                 no |
  |   partition has index table schema whose's create_mem_version == mem_version - 1 ----+
  |                 |                                                                    |
  |                 v                    succeed                                         |
  |       build index table partition ---------------------------------------------------+
  |                 | fail                                                               |
  |                 v                                                                    |
  |         call rpc set_index_error ----------------------------------------------------+
  |
  |
  +-------> observer finish daily merge

```

PS: （个人感觉索引创建后续可考虑做成同步的，即将索引表的schema插到schema里后，由rs触发从数据表
的静态数据构建索引表的partition，从数据表的memtable构建索引表的动态数据，成功之后索引立即生效
，不需要等到每日合并后才生效。这样用户体验会好些。）

### 删除索引表

```nohighlight

              rs receive drop_index rpc request
                         |
                         v
            rs drop index schema from schema
                         |
                         v
                response to client

```

每日合并时，被删除的索引表的partition会因为在schema中不存在而被删掉。

## 兼容性考虑

### SQL功能
### 老版本升级

## DBA考虑

### SQL功能
### 老版本升级
### SQL功能

## 测试考虑
