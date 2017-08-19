#New Root Major Freeze
##新版major freeze与旧版major freeze的对比
###旧版major freeze简介
在旧版的实现中，major freeze以租户为单位，对逐个租户依次执行major freeze，针对每一个租户的所有partition，旧版major freeze通过两阶段提交的方式实现，其中RS为scheduler，paritcipants为当前租户的所有partition，由于RS没有单独的写日志接口，实现中选取partition key最小的partition作为coordinator，并通过coordinator partition的日志标识两阶段提交的进度。执行major freeze的过程中所有的participant partitions都要停事务，因此major freeze过程中停事务的时间与participant partitions的数量正相关，当participant partitions的数量逐渐增大时，会极大影响业务。
###新版major freeze
为了降低major freeze对业务的影响，降低major freeze的执行时间，对major freeze的实现进行了修改：新版major freeze仍然采用两阶段提交实现，其中RS作为两阶段提交的coordinator，paritcipants为集群中的所有observer（后面的在实现细节中可能看到，participant也可能是集群中的部分参与者）。其中coordinator(RS)通过写__all_core_table表的形式记录两阶段提交日志，participants（observer）不写日志，participants的两阶段提交状态只记录到内存中。新版major freeze在执行过程中同样需要停事务（只读事务除外），停事务发生在observer收到RS的prepare freeze请求后，并在observer收到后续的commit freeze或abort freeze请求后解除停事务，以observer作为major freeze两阶段提交的participants，可以大大降低major freeze过程中停事务的时间，进而降低major freeze对业务的影响。
###两版major freeze的不同
* 旧版major freeze以partition作为两阶段提交的participants，major freeze执行成功后，所有的partition都已经产生新的memstore，直接完成了所有partition的升数据版本。
* 新版major freeze以observer作为两阶段提交的participants，major freeze执行成功后，只是observer级别维护的冻结信息发生了改变，observer上的partition并没有生成新的memstore，partition的升数据版本还需要其他机制触发，因此，新版本的major freeze实际上被划分为了冻结observer和partition升版本两个过程，partition升版本的方法为：leader partition感知到自身active memtable的版本与observer的冻结版本的差异，生成新版本memtable并写下升数据版本的日志，follower partition通过回放其leader写下的升版本日志完成。

##新版Root major freeze的实现
###major freeze的状态转换
通过如下的一个四元组来表示某个数据版本的major freeze的状态，我们称之为
frozen_status=（frozen_version,frozen_timestamp,freeze_status,schema_version）
其中frozen_version表示冻结版本号；frozen_timestamp表示这次major freeze的冻结时间戳；freeze_status表示本次冻结的两阶段提交的进度，可选取值有INIT,PREPARED_SUCCEED和COMMIT_SUCCEED；schema_version表示本次冻结的schema_version。
例如frozen_status=（5,10000,COMMIT_SUCCEED,10000）表示数据版本为5的major freeze已经提交成功，并且当时的时间戳为10000，schema_version为10000。不同版本的frozen_status之间的状态转换图如下所示（这里省略frozen_timestamp和schema_version，V表示版本号）：
```nohighlight

+----------+      prepare       
| V,COMMIT |------------>---------------+
+----------+                            |
                                        |
                                        | 
+----------+      prepare       +--------------+      commit          +--------------+
| V+1,INIT |------------------->| V+1,PREPARED |--------------------->| V+1,COMMITED |
+----------+                    +--------------+                      +--------------+
  ^                                     |
  |           abort                     |
  +---------------------<---------------+
```
在新版的major freeze中，每一个observer会维护
###major freeze依赖的内部表
* \_\_all\_core\_table表  
上文中提到，major freeze过程中coordinator（RS）需要写日志记录两阶段提交的进度，由于RS没有专门的写clog日志接口，RS的写日志方式为记录\_\_all\_core\_table内部表。RS在向所有observer发送prepare request前写prepare record，在发送commit request前写commit record，在发送abort record前些abort record。observer在收到prepare request后会停事务，使用普通sql是无法更新\_\_all\_core\_table，这里提供了一套独立的接口给RS用于写major freeze日志时修改
\_\_all\_core\_table
* \_\_all\_zone表
\_\_all\_zone表中的try_frozen_version和frozen_version两个域用于标识major freeze的开始和结束，假设当前集群的冻结版本号为V，则在新开启一轮major freeze前，会设置try_frozen_version为V+1，并在完成这轮major freeze后将frozen_version也设置为V+1，从而标识该轮major freeze结束，另外，在RS切主和集群重启时，也是通过比较try_frozen_version和frozen_version连个值是否相等来判断当前是否存在未完成的major freeze。
* \_\_all\_server表
新版major freeze的participants为observer，在执行major freeze时需要获取observer列表，获取observe列表是通过读取
\_\_all\_server表完成的。RS通过与observer间的心跳来维护\_\_all\_server表中各observer的状态，当observer在一段特定的时间内都没有与RS建立心跳，RS则认为该observer失联，并在\_\_all\_server表中修改该observer的状态，observer与RS失去心跳时，RS是无法区分observer是宕机状态还是RS与observer间发生网络分区的。

###新版major freeze对observer的要求
上文中提到，新版的major freeze包含冻结observer和所有partition升版本两部分，为保证冻结observer完成后，新开启的事务不会被写入partition老版本的memtable上，observer必须满足如下条件中的一个：
* \_\_all\_server表中的所有observer均处于alive状态，这种情况下，集群完成冻结后所有的partition都可以根据observer上的冻结状态的变化完成升版本操作。
* \_\_all\_server表中存在处于非alive状态observer，但如果该非alive的observer上不存在partition，也即该observer是一个空server时，也可以保证major freeze的正确性。

###Root major freeze的基本流程
```nohighlight
 +-----+
 |begin|
 +-----+
   |
   v
 +----------------------------------+
 |__all_zone.try_frozen_vesion += 1 |
 +----------------------------------+
   |
   v
 +--------------------------------------+
 |write __all_core_table prepare record |<--------------------<-------------------+
 +--------------------------------------+                                         |
   |                                                                              |
   v                                                                              |
 +-------------------------------+     FAIL                                       |
 |send prepare request           |-------->--------+                              |
 +-------------------------------+                 |                              |
   |                                               |                              |
   | SUCCESS                                       |                              |           
   v                                               v                              |
 +-------------------------------------+  +-----------------------------------+   |
 |write __all_core_table commit record |  |write __all_core_table abort record|   |
 +-------------------------------------+  +-----------------------------------+   |
   |                                               |                              |
   | SUCCESS                                       |                              |
   v                                               v                              |
 +-------------------------------+        +-------------------------------+       |
 |send commit request            |        |send abort request             |       |
 +-------------------------------+        +-------------------------------+       |
   |                                               |                              |
   | SUCCESS                                       |                              |
   v                                               |                              |
 +------------------------------+                  +--------------->--------------+
 |__all_zone.frozen_vesion += 1 |
 +------------------------------+
   |
   v
 +---+
 |end|
 +---+

```
###major freeze的异常处理
major freeze过程中存在RS切主和集群重启两种异常，这两种异常都会在随后发生一次新RS的上任，新RS上任后需要将之前未完成的major freeze推进完。
* 当前冻结状态为INIT：可依据上述major freeze的基本流程将major freeze推进完成。
* 当前冻结状态为COMMIT_SUCCEED: RS重新向各observer发送commit request直到成功。
* 当前冻结状态为PREPARED\_SUCCEED：此时集群可能处于停事务状态，\_\_all\_zone和\_\_all\_server表的选主结果不能写入到\_\_all\_root\_table表，进而导致major freeze无法继续进行。为处理这种情况，新RS上任时，如果当前的冻结状态为PREPARED_SUCCEED,会首先将冻结状态修改为INIT，并通过心跳冻将冻结状态同步给各observer，恢复集群为可写，并将本次major freeze推进完成。











