# Partition分裂 #
test
| header 1 | header 2 |
| -------- | -------- |
| cell 1   | cell 2   |
| cell 3   | cell 4   |
--------------------------------------------------------------------------------------------------------------------------------------------------------
  **编号**   **文档版本**   **修订章节**   **修订原因**                                                                        **修订日期**   **修订人**
                                                                                                                                              
  ---------- -------------- -------------- ----------------------------------------------------------------------------------- -------------- ------------
  1          0.1                           新建文档                                                                            2014/08/22     华庭
                                                                                                                                              

  2          0.2                           修改PartitionTable为2级结构                                                         2014/08/29     华庭
                                                                                                                                              
                                           Partition分裂后索引不用重建                                                                        
                                                                                                                                              
                                           提出一种每日合并全局冻结优化方案                                                                   
                                                                                                                                              
                                           Drop index不用先停读后停写                                                                         
                                                                                                                                              
                                           修改局部索引创建流程                                                                               
                                                                                                                                              

  3          0.3                           修改MetaPartitionTable实现，每个表的一个partition副本在MetaPartitionTable存储一行   **2014/9/9**   华庭
                                                                                                                                              
                                           修改partition迁移流程，不做checkpoint，直接同步日志                                                
                                                                                                                                              
                                           调整局部索引创建，MetaPartitionTable中记录partition副本状态信息                                    
                                                                                                                                              
                                           Partition统计信息可以存储在MetaPartitionTable中                                                    
                                                                                                                                              

                                                                                                                                              
  --------------------------------------------------------------------------------------------------------------------------------------------------------

## 1.  Partition维护功能 ##

      1.  Partition分裂迁移

随着数据库的长时间运行，可能发生各个partition负载和数据量不均衡的情况，解决的一种方案是对热点或大数据量的partition进行分裂并迁移到新的partition
server，使各个partition的数据量和访问量保持均衡。为了提升读写性能和避免分布式事务，将同一个table
space的table按照相同的partition
key进行hash分区，每个partition存储不会跨partition
server，从而保证大部分事务能在同一个partition内执行。基于这种实现方案，partition在分裂和迁移时也需要保证事务的局部性，即按照partition为单位进行分裂和迁移。

1.  PartitionTable

OB使用PartitionTable（partition位置信息内部表）来存储partition的位置信息。PartitionTable分成两级，第一级是RootPartitionTable，第二级是MetaPartitionTable。RootPartitionTable的主键为：meta\_table\_space\_id +
meta\_table\_id + meta\_partition\_id +
server\_addr（标识副本）；MetaPartitionTable的主键为：user\_table\_space\_id +
user\_table\_id + user\_partition\_id + server\_addr。

为了便于PartitionTable的扩展，创建RootPartitionTable表，该表只有一个partition，不能分裂，系统启动时首先会找到这张表的partition位置，加载完成这张表的partition数据后系统才能做接下来的初始化工作。OB包含了许多内部表，这些内部表都属于同一个table
space，每个内部表只有一个partition，不能分裂，内部表的partition的位置信息存储在RootPartitionTable中，每个内部表的每个partition副本存储一行。考虑到MetaPartitionTable未来可能比较大，特意为MetaPartitionTable创建一个table
space，初始只有一个partition，并且允许该partition进行分裂，MetaPartitionTable的partition的位置信息存储在RootPartitionTable中，每个副本存储一行。一般情况下RootPartitionTable中一共只有6行数据。

除内部表partition之外的每个用户表partition的每个副本在MetaPartitionTable中存储一行位置信息，行数据包含如下几列：

  -------------------------------------------------------------------
  User\_table\_space\_id   User table space id，schema中定义
                           
  ------------------------ ------------------------------------------
  User\_table\_id          用户表table\_id，schema中定义
                           

  User\_partition\_id      Hash(partition\_key)%N,N为partition总数
                           

  Server\_addr             副本所在的partition server address
                           

  Is\_leader               是否是partition的leader partition server
                           

  Row\_count               Partition总行数
                           

  Macro\_block\_count      Partition宏块总数
                           

  Size                     Partition的总大小
                           

  Other                    Partition的其它统计信息，比如列统计信息
                           
  -------------------------------------------------------------------

其中rowkey为user\_table\_space\_id +user\_table\_id+
user\_partition\_id +
server\_addr，但客户端一般只需要知道user\_table\_space\_name，user\_table\_name，partition\_key，hash函数和partition总数N，这些值都会在create
table时指定，partition\_id为用户使用公式Hash(partition\_key)%N计算出的值，N为partition总数。应用读写数据时，将user\_table\_name和partition\_key发送给OB的任意一台partition
server，partition
server查询schema获取user\_table\_name所属的user\_table\_space\_id和user\_table\_id，根据partition\_key，hash函数和partition总数计算出partition\_id，然后使用RootPartitionTable定位MetaPartitionTable的partition的leader
partition server ，使用user\_table\_space\_id ，user\_table\_id和
user\_partition\_id，向该leader partition
server查询获得用户表partition的位置信息，并在本地缓存，最后将读写请求发送给leader
partition server进行处理。

MetaPartitionTable比较小，一般不用分区，只有一个partition，并被单个partition
server paxos
group服务。MetaPartitionTable存储的列比较少，每张用户表的一个分区子表（partition）的副本都存储一行数据，所以该表的数据量跟partition副本数量成正比，假设每个partition
1G数据，每个partition存储6个副本，整个集群如果存储1PB数据，MetaPartitionTable总共6百万行数据，同时我们看到MetaPartitionTable大约有10个字段左右，单行数据大约100字节，总共大小600M。所以一般情况下MetaPartitionTable不会发生分裂，仅有一个partition。

如果MetaPartitionTable真的膨胀到比较大时需要分裂，参考下面用户表partition的分裂方式，唯一不同的是，用户表的partition分裂的修改MetaPartitionTable，而MetaPartitionTable的partition分裂修改RootPartitionTable，MetaPartitionTable的partition\_key为user\_table\_space\_id，保证属于同一个table
space的各个partition位置信息一定处于同一个MetaPartitionTable的partition中，避免查询一个table
space的不同表的partition位置信息时需要查询MetaPartitionTable的所有partition。

如果用户表partition发生迁移或者选出新的leader，只需要使用一条update
sql语句将MetaPartitionTable的server信息做相应的调整。用户表partition分裂时，一般不会改变hash函数和partition\_key，只改变partition总数N，比如将N从1024变成2048，那么partition\_id的范围变为0\~2047，其中原来的0\~1023范围内的partition\_id位置信息保持不变，新分裂出的1024\~2047范围内的partition\_id与0\~1023范围内的partition\_id的位置信息一一对应，partition\_id
1024与partition\_id 0的位置信息相同，partition\_id 1025与partition\_id
2的位置信息相同，以此类推。这个过程相当于原有的partition\_id
0分裂成新的partition\_id 0和1024，原有的partition\_id
1分裂成新的partition\_id
1和1025，且分裂出的新partition位置信息保持不变。如果partition发生分裂，只需要将新分裂出的partition位置信息用insert
sql语句写入内部表，每次insert一行，或者批量insert都可以。按照2的幂次方进行分裂（mysql的linear
partition方式）有一个好处，计算partition
id可以使用hash\_value&\~(N-1)，分裂时可以加快每行计算partition id。

属于某个table
space的所有表的partition必须一起分裂，但不用保证这些分裂出的partition位置信息在同一个事务中全部写入MetaPartitionTable，一个table
space包含的表的partition可能成千上万，在同一个事务中写入往往比较困难。这个方案先将新分裂出的partition位置信息加入内部表，但新加入的partition并不影响已有的partition，因为使用了新的partition
id，应用看到的partition总数N还没有修改，所以新加入的partition不会被应用看到，然后使用一个事务修改partition总数N，应用异步更新获取到这个新的partition总数，然后应用可以使用分裂后的partition位置信息。这个分裂过程只是修改了内部表，分裂后的partition共享SSTable和MemTable数据，称这个过程为逻辑分裂。在逻辑分裂后，应用使用旧的分区规则和新的分区规则都可以正确访问到数据，能够实现partition分裂对用户透明，平滑过渡。

同一TableSpace下表的partition(分区)将尽可能在一台机器上(但不强制，即不捆绑在一起)，RS调度需要留意；但数据表及其局部索引是捆绑到一起的。

1.  Partition分裂

Partition分裂类似传统关系数据库的扩库，但传统关系数据库扩库比较麻烦，一般会影响数据的读写服务，OB需要具备动态扩库并且不影响数据读写服务的能力。为了简化partition分裂功能的实现，每次将属于同一个table
space的每个partition分裂成多个partition，一般采用一分为二的方式，OB提供处理partition分裂功能的sql，由DBA根据业务和数据库情况使用该sql对partition进行分裂。

要实现Partition的平滑分裂，不能影响到读写服务，我们采用先逻辑分裂后物理分裂的方式，逻辑分裂指将MetaPartitionTable中相应的每条partition位置信息分裂成多条，物理分裂指将partition的数据按照指定的规则进行分裂，基本流程如下：

1.  逻辑分裂：按照上一节分裂修改内部表的方式进行逻辑分裂，将属于同一个table
      space的所有表的partition分裂，修改内部表，最后修改partition总数N。

2.  物理分裂：大版本冻结后，按分裂后partition管理active
      memtable的内存，便于为partition的修改增量创建checkpoint和回收partition占用的内存。每日合并时，将基线数据分裂，使分裂后的partition各自维护不同的SSTable。

逻辑分裂和物理分裂都完成后，整个partition分裂过程才算完成，整个过程对用户透明。物理分裂需要在每日合并中完成，在每日合并过程中，发生分裂的partition按照新的分裂规则，每行数据重新计算partition\_id，从而将属于不同partition的物理数据分离出来，这过程将对partition的数据全部读写一遍。

在做物理分裂的每日合并过程中，不能对正在物理分裂的partition做迁移，只能在每日合并完成后迁移，不然迁移时就需要对基线数据和冻结转储数据的每行计算partition\_id，从而将物理数据按照partition分开，那么每日合并做的物理分裂就有一部分做的无用功，迁移的旧版本数据还需要再次进行每日合并，造成资源浪费。

如果partition逻辑分裂后的每日合并将全部的partition数据进行物理分裂，每日合并的代价可能比较大，可以采用渐进式合并的思想，设定一个规则，每次每日合并将其中的1/10
partition进行物理分裂，增量数据的内存管理可以按照分裂后的partition进行管理。

Partition物理分裂只能针对数据表，对索引表做物理分裂有时是没有意义的，因为索引表附属于数据表，不是根据partition
key进行分区，所以数据表物理分裂后，如果索引表中没有存储partition\_key，那么索引表就无法根据partition\_key按新的规则进行分裂，不能参与物理分裂的那次每日合并，其索引表只能每日合并后重新读取数据表数据来创建。一般情况下，partition
key是主键或唯一索引的一个组成部分，索引表的rowkey为索引列+数据表rowkey，数据表rowkey一般包含了partition\_key，或者索引表冗余存储了partition\_key这种情况下，在发生partition分裂后的每日合并时，索引表也可以参与每日合并，按照同样的分裂规则进行物理分裂。为了便于实现，可以强制保证索引表中冗余存储了partition\_key。

1.  Partition复制与迁移

当发生partition server宕机，该partition
server服务的partition需要复制到新的partition
server；当partition分裂后，由于负载或数据均衡需要将partition迁移。主rs控制Partition复制和迁移流程：

1.  Rs发起partition复制或迁移，通知目的partition server开始从源partition
      server（一般为leader）拉取日志和数据。

2.  目的partition server开始从源partition
      server拉取基线数据和可能的转储数据。

3.  目的partition server向源partition
      server拉取需要迁移的partition上次大版本冻结时的log
      id后的日志，源partition
      server需要遍历日志，将属于该partition的日志过滤出来，目的partition
      server收到日志写入磁盘，不进行回放。

4.  目的partition server持续从源partition
      server拉取基线数据和转储数据，直到拉取完成，同时目的partition
      server拉取的日志已经接近源partition
      server的日志，如果是迁移操作，rs通知leader将源partition
      server剔除paxos
      group并修改MetaPartitionTable，然后leader将目的partition
      server加入paxos group，目的partition
      server开始回放日志，直到接近追上leader的日志。这个过程中目的partition
      server暂时不参与投票，但接受leader同步的日志写入磁盘。

5.  目的partition server接近追上leader日志，通知leader将目的partition
      server加入paxos group的投票列表，目的partition
      server将缺失的log从leader中获取，最后rs将目的partition
      server加入MetaPartitionTable。如果是复制操作，不需要将源partition
      server剔除paxos group。

6.  如果是迁移操作，rs通知源partition
      server删除基线数据和可能的转储数据，以及该partition的增量数据占用的内存。

这个方案中在复制迁移完成的最后切换阶段，与spanner处理directory的迁移复制类似，实现相对简单，异常处理考虑还不够周全，需要进一步理清楚第6步的处理过程。在paxos
group中平滑迁移副本，实现比较复杂，可以参考论文：The SMART way to
migrate replicated stateful
services。<http://research.microsoft.com/pubs/67913/eurosys2006.pdf>。其中的切换时paxos
group的异常处理参考《日志同步与宕机恢复总体设计》。

1.  Partition局部索引创建

云版本支持局部索引，局部索引的使用有一定的限制，局部索引的创建比0.5版本更简单，但整个控制过程更简单，在创建过程中不需要持久化记录任何状态，只需要在rs检查到所有的partition的索引都创建完成后，将索引表的状态设置为可用。局部索引的主要的流程如下：

1.  客户端将create index的sql命令发送给主rs所在的partition server。

2.  Rs更新schema内部表，并根据该索引的数据表partition位置信息，创建与数据表一一对应的partition位置信息，将索引表partition位置信息插入MetaPartitionTable，并设置其status为unavailable，完成内部表修改后响应客户端。

3.  异步更新schema，数据表的partition
      leader获取到schema后，写入数据时同时更新数据表和索引表。

4.  每日合并进行错峰合并，每个集群在合并完成后，检查当前合并完成版本是否有局部索引需要创建，如果有则开启局部索引创建，首先会查询MetaPartitionTable，获取另外的partition副本位置，并查看其它partition副本的局部索引是否创建完成了（其partition
      status为available），如果其它副本已经创建好，则发起复制索引数据到本地，否则开始自己创建，使用一个可调度的并行执行计划来执行索引partition创建，使用sql模块的调度器和执行器来完成，并汇报该partition，将MetaPartitionTable中该partition的status改为available。如果本版本上需要创建的局部索引都创建完成后，设置本集群每日合并完成，并通知主rs，主rs通知还没有进行每日合并的集群进行每日合并，直到所有的集群都完成每日合并。

5.  主rs检查所有集群每日合并是否完成，如果已经完成，扫描schema，找出本版本需要创建索引的表，然后查询MetaPartitionTable，向包含表的partition的server查询其索引是否创建完成了，如果所有的partition副本都创建完成了，将schema中索引表的状态设置为available，该索引表可以被查询使用。

1.1.2节建议索引表冗余存储partition\_key，那么索引表在partition分裂时也能正常参与每日合并，减少实现复杂性。但如果索引表真没有存储partition\_key，在发生partition分裂时也需要重建索引，在partition分裂后的每日合并中，数据表的partition合并完成时，不能立即汇报数据表partition位置信息更新PartitionTable，需要其索引表的partition重新创建完成后，同时更新MetaPartitionTable中数据表和索引表的位置信息。这样实现可以避免数据表partition分裂后，相应的索引表partition没有创建好不可用的问题。

1.  每日合并

云版本仍然采用全集群统一冻结，错峰合并和渐进式合并的方式进行每日合并。统一冻结保证全集群有统一的版本，便于schema等信息的全集群同步；错峰合并最大限度减少每日合并对系统读写服务的影响；渐进式合并保证每次每日合并的数据量均衡，避免多次每日合并的数量变化过大对系统带来过大的冲击。

1.  统一冻结

全集群统一冻结，需要执行一个分布式事务将所有的partition冻结，在这个过程中，每个partition不能接受新的写入操作，并等待正在处理的事务完成，保证整个集群能够创建一个一致完整的check
point。整个冻结过程可能耗时数秒，主rs负责整个冻结过程的控制，基本流程如下：

1.  主rs发起全集群冻结，先冻结内部表，由于所有的内部表只有一个分区，只被一个paxos
      group服务，内部表冻结成功后，会创建新的active
      memtable来写入，冻结过程中只有内部表可以写入新数据。

2.  Rs将开始全集群冻结的信息写入内部表，并通知其它所有的leader冻结。

3.  Leader收到rs冻结请求后，阻塞新的更新，并等待正在执行的事务完成，这个过程中，任何leader不能再发起任何新的分布式事务，已经处于冻结阶段的leader也不能接受其他leader发来的分布式子事务处理请求，由于多个leader的冻结启动时间可能不同，所以在全集群冻结过程中，一些分布式事务可能prepare失败，已经达成commit状态的分布式事务可以继续完成。各个Leader冻结时创建新的active
      memtable，写一条冻结日志并同步到多数派后，通知rs冻结完成，继续阻塞新的更新操作。

4.  Rs收到所有leader的冻结成功消息后，修改内部表，标识执行冻结的分布式事务的状态为commit，rs通知所有的leader可以开始处理阻塞的更新操作。

使用内部表同步全集群执行冻结的分布式事务状态，实现简单，异常处理相对简单，在整个过程中，如果rs宕机或重新选主，可以通过内部表恢复事务状态。Leader重新选举或等待rs的响应超时，leader可以查询该内部表获取执行冻结的分布式事务状态。

一种优化方案，全局冻结时，不停止任何写服务，开启一个分布式事务来让所有partition
server协商出一个最小的已提交的global\_transaction\_timestamp，这个global\_transaction\_timestamp就是全局冻结版本号，每个partition
server持久化这个冻结版本号和所在的log文件，创建新的memtable，启动一个后台线程将冻结的memtable中global\_transaction\_timestamp之后数据拷贝到新的memtable中，memtable的数据迁移不是那么好实现。每日合并读取数据只能读取到global\_transaction\_timestamp。

1.  错峰合并

Rs发起全集群冻结完成后，接着会控制全集群的错峰合并，错峰合并的基本策略与0.5版本类似，采用内部表来记录各个集群的合并状态，只是在一个集群合并开始前，需要将该集群的partition
leader全部发起重新选主，选择其它集群的partition
server为partition的leader。

Partition
leader在有主选举的情况下，可以高效实现选主，旧leader指定新的leader并通知所有follower，新leader同步了旧leader的所有日志后开始提供服务。每个partition
server每合并完成一个partition，通过sql更新MetaPartitionTable的相应信息，如果主rs发现合并集群的所有partition
server都已经合并到最新版本，切换到下一个集群进行每日合并，直到所有的集群都合并到最新版本。

1.  渐进式合并

渐进式合并保留0.5版本的实现方式，保证带有过期条件的表，每次每日合并过期部分数据，如果数据表带有局部索引，需要每日合并过程中将过期的行的索引行删除。另外一种情况也需要使用渐进式合并，partition分裂后每日合并，由于要保证一个table
space的所有partition都参与分裂，分裂后的每日合并需要将partition进行物理分裂，对系统的冲击比较大，渐进式合并保证每次每日合并只物理分裂部分partition，在一段时间后完成所有partition的物理分裂。但需要注意的是数据表在每日合并中物理分裂后，如果带有过期条件，不需要将过期的行的索引行删除，其对应的索引表在每日合并完成后会进行重建。

1.  DDL操作

云版本的DDL操作执行时间都比较长，需要使用异步的方式进行处理。Create
table/create
index操作，更新相应内部表（schema表和MetaPartitionTable表）并标识partition不可用，完成后响应客户端。然后OB内部异步完成schema的同步，在后台完成各个partition的创建和选主，partition
server每完成一个partition的创建，更新MetaPartitionTable将该partition设置为可用。

Drop table/drop
index与0.5的处理方式类似，修改内部表状态完成后响应客户端，
rs后台扫描已经删除的table，并将其partition位置信息从内部表中删除，partition
sever在每日合并过程中删除已经drop的表的partition。

传统数据库用表锁来互斥drop table，drop
index等ddl操作与dml操作，在所有dml完成后，ddl操作获取表独占锁，然后做相应的drop
table操作。由于OB是分布式数据库，接受sql的partition
server可能没有所需的所有的partition数据，需要在接受请求的partition
server上构建执行计划，然后将执行计划发送给对应partition
server执行，在整个分布式系统的Partition
server要么都使用局部索引，要么都不使用局部索引，所以需要全局协调，create
table，create index只有所有partition都处理完成后，表才可用。Drop
table，drop
index不采用渐进的停止方式，0.5版本全局索引先停读后停写实现比较麻烦，容易有某台机器没有获取到最新的schema而发生数据一致性问题。所以，1.0版本还是以局部索引整体停读写来考虑，所有的多机事务都需要考虑协调者（生成execution\_plan）和参与者（执行execution\_plan）之间schema不一致的情况。如果协调者的schema版本更新，那么参与者执行时要获取到相应的schema后才能执行；如果协调者的schema版本更旧，参与者返回一个错误码，协调者识别这些特定错误码并更新schema后重新生成执行计划。

test