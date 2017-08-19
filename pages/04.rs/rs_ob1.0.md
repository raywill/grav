# 云版本中的RS

RS是RootServer的缩写，在OB 0.5及以前，是一个单独的server。在云版本中，此部分功能将与UPS, MS, CS一样，合并到OB Server。
这里还是沿用之前的名字，叫做RS模块，是OB Server中以下功能的集合：

- OB Server心跳维护
- 冻结管理以及错峰合并控制
- partition管理
- DDL操作执行
- Bootstrap

## 部署

RS是OB Server中的部分，但一般的实例不启动此模块，只在\_\_all_root_table的partition leader
所在OB Server上启动。当此leader变为follower时，停止上面的RS模块，保证全局只有一个RS模块在工作。
RS模块启动时通知ConfigServer新的RS地址(或修改DNS)。

## 心跳维护

跟0.5一样，RS维护所有server的心跳，并同步更新\_\_all_server表。
RS通过心跳机制通知schema_version, frozen_version 等。RPC为[renew_lease](#renew_lease)，这里，我们新增机器信息。

OB Server renew lease request:

```cpp
		int64_t version;
		int64_t cluster_id;
		ObServer server;
		int64_t inner_port; // mysql listen port.
		ObServerResourceInfo resource; // OB Server, version, CPU, memory, disk info
```

RS response:

```cpp
		int64_t version;
		int64_t lease_expire_time;
		int64_t lease_info_version; // 通过lease_info_version检查是否需要从__all_cluster_stat中更新信息
		int64_t frozen_version;
		int64_t schema_version;
```

privilege_version, config_version, broadcast_version, last_merged_version, is_pause_merging 等信息跟 0.5 中一样，
从\_\_all_cluster_stat中获取。

## 冻结 && 错峰合并

小版本冻结由各OB Server自行处理，大版本冻结需要RS参与作一个协调者 (具体见华庭《partition分裂》中"统一冻结"一节)。
冻结过程是由RS当协调者的分布式事务，RS以冻结\_\_all_root_table表partition的形式记日志。恢复阶段，如果OB Server
只收到[prepare_major_freeze](#prepare_major_freeze)没收到[commit_major_freeze](#commit_major_freeze)或
[abort_major_freeze](#abort_major_freeze)，则通过[query_frozen_stat](#query_frozen_stat)向RS查询是否冻结完成。

RS执行冻结过程如下：

		RS frozen workflow:
		
		try_frozen_version: __all_cluster_stat.name = 'try_frozen_version'
		frozen_version: __all_cluster_stat.name = 'frozen_version'
		
		
		  begin frozen                                   RS module started
		        |                                               |
		        v                                               |
		  update try_frozen_version += 1                        |
		        |                                               |
		        v                                               |
		  try_frozen_version == frozen_version <----------------+
		  (assert: 0 <= try_frozen_version - frozen_version <= 1)
		        |                         |
		        |False                    +---------------------------------------------+
		        v                                                                       |
		  check all partition leader alive.                                             |True
		        |                                                                       |
		        v                                                                       |
		  prepare_major_freeze: send prepare_major_freeze RPC to all partition          |
		        |                                                                       |
		        v                                                                       |
		  write coordinator log:                                                        |
		  call commit_major_freeze api on __all_root_table partition leader             |
		        |                                                                       |
		        v                                                                       |
		  send commit_major_freeze RPC to all partition                                 |
		        |                                                                       |
		        v                                                                       |
		  update frozen_version = try_frozen_version                                    |
		        |                                                                       |
		        v                                                                       |
		  end frozen <------------------------------------------------------------------+

如果发prepare_major_freeze失败或commit_major_freeze \_\_all_root_table partition leader失败，
RS则发abort_major_freeze取消。

此流程涉及4个RPC:

- [prepare_major_freeze](#prepare_major_freeze): (
  partition收到此请求后不允许开新的写事务，停止长时间未提交的事务，同时等待此partition上已提交的分布式事务完成。
  partition写prepare_major_freeze日志。

- [commit_major_freeze](#commit_major_freeze):
  确认冻结RPC，完成partition冻结，允许写入，同时写commit_major_freeze日志。

- [abort_major_freeze](#abort_major_freeze):
  取消冻结RPC，允许写入，同时写abort_major_freeze日志。

- [query_frozen_stat](#query_frozen_stat):
  查询一个版本是否冻结，用于partition恢复阶段，如果只有prepare_major_freeze没有对应的commit或abort。则向
  RS发此RPC，查询版本是否冻结，以决定prepare_major_freeze是提交还是取消。在日志回放完之前，返回UNKNOWN，需要调用者重试。
  (此接口要求RS通过API查询partition状态是否为已回放完日志，以及当前memtable版本)。


错峰合并跟0.5类似，只是在合并之前需要将本集群的leader切到其它集群去。考虑到要尽量将一个tablespace的partition leader
放到一起，leader切换须由RS指定。整个集群合并完成后，恢复集群负载。整个过程以最少leader切换为原则，
合并完成后，不保证各partition leader跟合并之前一样。合并顺序按cluster_id进行，且RS所以集群最后合并。如有三个集群：
cluster 1, cluster 2, cluster 3，RS在 cluster 2上，则合并顺序为：cluster 3, cluster 1, cluster 2。

RS判断一个集群是否合并完成跟0.5中一样：检查此集群所有partition都合并到指定版本。各OB Server通过定时器读取
\_\_all_cluster_stat表，获取整个大集群的merged version，将小于merged version的memtable或转储删除掉。


## partition管理

对于RS来说，partition跟我们0.5中的Tablet类似。partition信息存放在内部表中，分两级内部表存放：

* \_\_all_root_table: root租户所有表partition信息，包括以下表：
	- \_\_all_database,
	- \_\_all_tablespace,
	- \_\_all_table,
	- \_\_all_column,
	- \_\_all_join_info,
	- \_\_all_ddl_operation,
	- \_\_all_tenant,
	- \_\_all_tenant_resource,
	- \_\_all_user,
	- \_\_all_database_privilege,
	- \_\_all_table_privilege,
	- \_\_all_cluster,
	- \_\_all_cluster_stat,
	- \_\_all_sys_param
	- \_\_all_sys_stat,
	- \_\_all_sys_config,
	- \_\_all_sys_config_stat,
	- \_\_all_trigger_event,
	- \_\_all_server
	- \_\_all_meta_table
* \_\_all_meta_table: 普通租户所有表信息partition信息。

以上表中，只有\_\_all_meta_table可分区(按tenant_id, table_id和partition_id分区)。

\_\_all_root_table 类似于我们0.5中root table中的\_\_first_root_table，partition信息存在于RS内存中，
由RS模块启动时，主动向对应的OB Server获取。

#### partition读取

OB Server执行sql时需要取得partition信息，通过PartitionCache模块获取(类似于0.5中的location cache)。
PartitionCache使用kv-cache实现，(这点区别于0.5中的location cache，因为现在hash接口可满足需求，不再需要btree)，
由kv-cache提供各租户间资源隔离与控制。

PartitionCache提供以下接口：

```cpp
		struct PartitionLocationInfo
		{
			ObServer addr_;
			int role_; // leader, follower
		};

		struct PartitionInfo
		{
			int64_t partition_id_;
			ObSEArray<PartitionLocationInfo> location_;
		}

		// 根据table_id, partition_id获取partition地址
		// force_renew表示cache中为过期数据，需要重新读取
		int get(int table_id, int partition_id, PartitionInfo &partition_info, bool force_renew);

		// 取得一个表所有partition地址列表
		int get(int table_id, ObIArray<PartitionInfo> &location_list);
```

PartitionCache实现：直接通过SQL读取对应表的PartitionTable并加入cache中，\_\_all_root_table的partition信息
通过RPC到ConfigServer获取。系统表partition信息不可被淘汰。
如果一个partition在cache存在超过一定时间(10分钟)，则发起异步任务重新获取。(与0.5中Tablet Location维护相同)

### PartitionTable维护

在0.5中，Tablet信息以CS中存储的Tablet为准，通过Tablet汇报方式维护RootTable。
在云版本中，partition信息以OB Server上各partition上的 Member List为准。PartitionTable通过以下方式维护：

1. RS切换时，\_\_all_root_table的partition信息由RS其Member List(leader + follower)直接通过
   [fetch_root_partition](#fetch_root_partition) RPC主动获取。
2. 每日合并时，由OB Server直接修改OB Server更新data_version, checksum等信息
   (\_\_all_root_table partition信息通过[report_root_partition](#report_root_partition) RPC到RS更新)。
   如果partition对的table在schema中不存在，则将partition删除。
3. partition成员变更后，更新PartitionTable，将不在Member List中的partition删除
4. 每个partition选出leader时：由leader操作PartitionTable修改对应partition状态为leader，
   并将不在Member List中的partition从PartationTable中删除。
5. 当partition变为有效的follower(无主选举成功)时，检查其partition是否在PartitionTable存在，不存在则插入

考虑到整体重启，错峰合并以及集群间网络闪断等场景有大量PartitionTable操作，可将上面的4, 5做成异步，放到异步队列里单线程执行。

另外，还需通过下面的timer保证PartitionTable与OB Server中partition一致：

a. RS模块中定时将长时间下线的server对应的partition从PartitionTable中删除。(并触发balance)
b. OB Server定时遍历PartitionTable，将PartitionTable中存在，但OB Server上不存在的partition删除。
c. OB Server定时遍历本地的partition，如果partition不是正在复制，且不是参与者，
   且不在其对的partition Member List中(通过[get_member_list](#get_member_list) RPC向leader获取Member List)，则将其删除。

上面b, c两部分可在一个timer里实现。

典型的场景，PartitionTable维护：

- 建表过程中PartitionTable插入：4, 5 (选完主后，leader和follower自行插入)
- RS切换: 1
- 刚做完Member Change，还没来得及更新PartitionTable就宕机重启：
  4 (PartitionTable删除老Member), 5 (PartitionTable插入新Member), c (老partition删除)
- OB Server永久下线：a (定时清理)
- OB Server下线清空数据(或部分partition丢失)再启动： b (定时清理)
- drop table后partition清理：2

### partition均衡

#### leader管理

RS模块定期遍历所有partition，如果发现同一partition group (tablesapce相同且partition id相同的partition)，
机器分布相同，但leader在不同机器上，则发起改选 ([switch_leader](#switch_leader))。将此partition group所有leader改选到：非错峰合并且当前主最多的集群上。
如果非错峰合并集群partition不在用户期望的集群，也改选到用户期望集群。

#### 副本管理

partition均衡首先是保证partition副本足够，在机器下线一定时间后(safe_lost_time)，发起复制任务。
机器下线后不立即从\_\_all_server中删除， 只是更改其状态为offline，RS发起定时任务检查\_\_all_server表，
当OB Server offline一定时间后再删除。

partition动态均衡跟0.5一样，由RS通过定时任务检查发起，通过partition的迁移达到超卖率，资源的均衡。
迁移时，需要尽量将一个partition group的的partition放到一起。考虑超卖，
机器资源均衡等多因子的均衡算法等我们收费模型确认后再调研补充。
一期可以只实现最简单的均衡算法： 只考虑partition数，实现快速简单。均衡算法在实现时，需要相对独立，
方便替换升级。RS通过心跳机制获取机器CPU，内存，磁盘等信息。静态数据大小，上次合并的动态数据大小，
访问统计等其它信息通过内部表(PartitionTable)获取。

以partiton group为单位生成复制/迁移任务 ([migrate](#migrate))，单个partition只允许一个任务存在。
partition复制/迁移过程见 华庭《partition分裂》。
OB Server复制/迁移完成后，完成Member List变更 ([migrate_member_change](#migrate_member_change)，
同时更新PartitionTable，然后通知RS任务完成 ([migrate_over](#migrate_over))，
RS将任务从任务列表中移除，同时触发下个partition的均衡：如果任务列表中有同个partiton group的任务，
则触发执行，否则触发下个partition group均衡任务。同时，RS会定期遍历任务列表，将超时任务移除。

均衡模块在RS中实现为一个单独线程线程，定期(sleep 一定时间，也可由其它任务唤醒)执行以下任务：

1. leader改选：遍历所有partition group，如果leader不在同台机器，则找出leader最多的机器，将其它partition leader改选到此机器。
   ([switch_leader](#switch_leader))
2. partition副本数保证：遍历所有partition group，如果存在partition副本数少于应有副本数，则发起复制任务，复制到所缺失副本集群
   同partition group所在OB Server。如果一个partition group在一个集群中不存在，则选择复制到其关联tablespace所有pattition gorup最少
   的机器上 (同4.1)。
3. partition group member统一：遍历所有partition group，如果存在同一partition group member不统一，且在复制/迁移任务列表中
   不存在此partition group的任务，则发起迁移任务：将每个集群partition迁到此集群此partition group中table_id最小的
   partition所在机器。
4. OB Server partition平衡：
	- 遍历所有集群
		1. tablespace 中 partition group 均衡：如一个集群中有A, B两台机器，而一个tablespace中有两个partition group: pg1, pg2。
		   如果pg1, pg2都在机器A上，则将pg2迁到机器B，以达到partition group均衡。主要针对场景为partition分裂。
			- 遍历所有tablespace
				- 计算此tablespace中partition group在所有机器上的分布
				- 存在最大partition gorup数(max_pg_num)机器A，与最小partition group数(min_pg_num)机器B。
				  如果max_pg_num - min_pg_num > 2则从机器A中选择partition数最少的partition gorup迁到机器B。
		2. partition数均衡：如集群中有A, B两台机器，对应partition数为1000, 200，则会从A中迁340个partition到B。
		   主要针对场景为新机器加入。
			- 遍历此集群所有partition，算出此集群所有OB Server平均partition数 (avg_partition_num)。
			- 选partition数小于avg_partiton_num * 90%，且avg_partiton_num > 10的OB Server上做为目的端机器 (dest)。 
				- 选partition最多的机器做为源端机器 (src)。
				- 选择src中存在，但dest中不存在，且partition group中partition最少的partition group进行迁移。

1 可以每次balance都执行，2, 3, 4有顺序关系，只有前一类任务达到平衡之后才做后一类任务。

OB Server上执行复制迁移任务流程如下：

		OB Server partition copy/migrate workflow:


								  traverse __all_partition_balance table,
		  RS send copy/migrate    exist copy/migrate task
		  --------+-----------    ---------------------------------------
				  |                   |
				  v                   v
			 check destination partition exist
				  |                   |
				  |not exist          |exist
				  |                   v
				  |               check partition exist in member list-----------------+
				  |                   |                                                |
				  |                   |not exist                                       |           
				  |                   v                                                |
				  |               delete destination partation                         |
				  |                   |                                                |exist
				  v                   v                                                |
			 sync log (as one observer), pull checkpoint, pull static data ....        |
				  |                                                                    |
				  v                                                                    |
			 send migrate_member_change request to leader                              |
				  |                                                                    |
				  v                                                                    |
			 send migrate_over to RS <-------------------------------------------------+
				  |
				  v
			 RS delete copy/migrate task from task list
             and trigger next copy/migrate task execute.



		leader do migrate_member_change:

		     begin
               |
               v
		  is migrate task? ----------YES: delete source partition from member list----+
			   |                                                                      |
			   |NO                                                                    |
			   v                                                                      |
		  MemberList.cout() == MAX_MEMBER_COUNT                                       |
			   |               |                                                      |
			   |False          |True                                                  |
			   |               v                                                      |
			   |         available participant count == MAX_MEMBER_COUNT              |
			   |               |                                  |                   |
			   |               |False                             |True               |
			   |               v                                  v                   |
			   |         elect an unavailable member        abort && return fail      |
			   |         to delete                                                    |
			   |               |                                                      |
			   |               v                                                      |
			   |         delete member <----------------------------------------------+
			   |               |                                                       
			   v               v                                                       
		  add destination partition to member list
               |
               v
		      end   

涉及RPC: 
[migrate](#migrate), [migrate_member_change](#migrate_member_change), [migrate_over](#migrate_over)

## DDL操作执行

跟0.5中一样，RS负责执行DDL操作，执行过程也与之类似。主要分为: 修改schema表，创建partition(create table)，
刷新schema，其中修改schema和刷新schema与0.5一致。OB Server创建partition时，不能依赖其相关的schema，如创建失败，
则RS回滚schema修改。 每日合并时，如果一个partition没有对应的schema则将此partition丢弃。
一期可以在每日合并期间禁止DDL，避免刚创建的partition被丢弃问题。

### create tenant

创建一个普通租户流程如下：

- 获取tenant_id:

```sql
		begin;
		select value from __all_sys_stat where name = 'max_used_tenant_id' for update;
		tenant_id = value + 1;
		update __all_sys_stat set value = tenant_id where name = 'max_used_tenant_id';

		select value from __all_sys_stat where name = 'max_used_database_id' for update;
		database_id = value + 1;
		update __all_sys_stat set value = database_id where name = 'max_used_database_id';

		select value from __all_sys_stat where name = 'max_used_tabespace_id' for update;
		tablespace_id = value + 1;
		update __all_sys_stat set value = database_id where name = 'max_used_tablespace_id';
```

- 插入tenant:

```sql
		insert into __all_tenant (tenant_id, tenant_name, info) value (tenant_id, tenant_name, ...);
		insert into __all_tenant_resource (tenant_id, cpu_reserv, cpu_max, ...) value (tenant_id, ...);
```


- 系统表中插入默认值：

```sql
		insert into __all_tablespace (tenant_id, tablesapce_id, tablespace_name, comment)
			values (tenant_id, tablespace_id, 'default', 'default tablesapce');

		insert into __all_database (tenant_id, database_id, database_name, comment) values (tenant_id, database_id, 'default', 'default database');

		insert into __all_user (tenant_id, user_name, user_id, host, passwd, info, priv_all, priv_alter, priv_create,
				priv_create_user, priv_delete, priv_drop, priv_grant_option, priv_insert, priv_insert,
				priv_update, priv_replace, is_locked)
            		valuese (tenant_id, 'root', user_id, '%', 'password', 'system administrator', 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0);

        insert into __all_database_privilege (tenant_id, user_id, host, database_id, priv_all, priv_alter, priv_create,
				priv_delete, priv_drop, priv_grant_option, priv_insert, priv_update, priv_select, priv_replace)
					values (tenant_id, user_id, '%', databse_id, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1);

		insert into __all_sys_param (tenant_id, name, data_type, value)
                values (tenant_id, 'autocommit', 1, 1),(tenant_id, 'tx_isolation', 6, 'READ-COMMITTED')...;

		insert into __all_ddl_operation (schema_version, tenant_id, database_id, tablespace_id, table_id, operation_type)
			values (time_to_usec(now), tenant_id, 0, 0, 0, DDL_CREATE_TENANT);
```

- 完成修改，并刷新schema ([switch_schema](#switch_schema))

```sql
		commit;
```

整个创建租户操作在一个事务中完成。

### create database

```sql
		begin;
		select value from __all_sys_stat where name = 'max_used_database_id' for update;
		database_id = value + 1;
		update __all_sys_stat set value = database_id where name = 'max_used_database_id';
		insert into __all_database (tenant_id, database_id, database_name, comment) values (tenant_id, data_base_id, 'name', 'commnet');
		insert into __all_ddl_operation (schema_version, tenant_id, database_id, tablespace_id, table_id, operation_type)
			values (time_to_usec(now), tenant_id, database_id, 0, 0, DDL_CREATE_DATABASE);
		commit;
```

### create table

- 获取table_id:

```sql
		begin;
		select value from __all_sys_stat where name = 'max_used_table_id' for update
		table_id = value + 1;
		update __all_sys_stat set value = table_id where name = 'max_used_table_id';
		commit;
```

- 插入schema:

```sql

		begin; (schema_trans)
		insert into __all_table (tenant_id, table_id, table_name, database_id, tablespace_id, ....)
			values (tenant_id, table_id, 'name', database_id, tablespace_id, ...);
		insert into __all_column (tenant_id, table_id, column_id, column_name, ...) values (tenant_id, table_id, column_id, column_name ...);
		insert into __all_ddl_operation (schema_version, tenant_id, database_id, tablespace_id, table_id, operation_type)
			values (time_to_usec(now), tenant_id, database_id, tablespace_id, table_id, DDL_CREATE_TABLE);
```

- PartitionTable中插入默认值。(使得之后的建表操作知道同个partition group的partition所在机器)

```sql
		insert into __all_meta_table (tenant_id, table_id, partition_id, ip, port, data_version, ....)
			values (tenant_id, table_id, partition_id, ip, port, 0, ...), (...);
```

- 创建partition: 为每个partition选择server，并发送[create_partition](#create_partition)RPC请求。 
  当建表涉及的partition较多时，顺序创建partition会导致create table执行时间过长。
  可将请求并发发出去，只要等到一个partition的多数应答成功即认为此partition创建成功。(1.0中可先不做此优化)

- 完成修改并刷新schema。

```sql
		commit; (schema_trans)
```

除增加获取table_id外，整个操作在事务中完成，可随时回滚。如果创建不成功，最坏的情况是导致table_id增长1，
和留下一些垃圾partition。垃圾partition由上面的的partition维护机制删除 (每日合并时删除schema不存在的partition)。

对于建表操作，需要等到所有partition选出主后，用户才能执行DML语句。有两种做法：

1. 建表操作是异步的，client收到建表成功的结果后，需等待一定时间才能执行DML
2. 同步建表，由server端等到所有partition都选出主后再返回给client.

方法1 不符合用户的直觉，与mysql行为也不一致。我们应该按方法2执行，由OB Server在返回给client端之前，
遍历所有partition直到所有主都选出后再返回给client。


### drop table

```sql

		begin;
		delete from __all_table where tenant_id = tenant_id and table_id = table_id;
		delete from __all_column where tenant_id = tenant_id and table_id = table_id;
		insert into __all_ddl_operation (schema_version, tenant_id, database_id, tablespace_id, table_id, operation_type)
			values (time_to_usec(now), tenant_id, database_id, tablespace_id, table_id, DDL_DROP_TABLE);
		commit;
```

table相关的partition由partition维护机制自行删除。

### drop database

```sql

		begin;
		for table_id in (select table_id from __all_table where tenant_id = tenant_id and database_id = database_id) {
			delete from __all_table where tenant_id = tenant_id and table_id = table_id;
			delete from __all_column where tenant_id = tenant_id and table_id = table_id;
		}
		delete from __all_database where tenant_id = tenant_id and database_id = database_id;
		insert into __all_ddl_operation (schema_version, tenant_id, database_id, tablespace_id, table_id, operation_type)
			values (time_to_usec(now), tenant_id, database_id, 0, 0, DDL_DROP_DATABASE);
		commit;

```

drop database会将相关的table一并drop，避免在系统表中留下垃圾数据被用户看到。

### drop tenant

```sql
		begin;
		delete from __all_tenant where tenant_id = tenant_id;
		delete from __all_tenant_resource where tenant_id = tenant_id;
		delete from __all_table where tenant_id = tenant_id;
		delete from __all_column where tenatn_id = tenant_id;
		delete from __all_user where tenant_id = tenant_id;
		delete from __all_database where tenant_id = tenant_id;
		delete from __all_database_privilege where tenant_id = tenant_id;
		delete from __all_tablespace where tenant_id = tenant_id;
		delete from __all_sys_stat where tenant_id = tenant_id;
		delete from __all_system_param where tenant_id = tenant_id;
		insert into __all_ddl_operation (schema_version, tenant_id, database_id, tablespace_id, table_id, operation_type)
			values (time_to_usec(now), tenant_id, 0, 0, 0, DDL_DROP_TENANT);
		commit;
```

## inner sql client

因为我们所有server都有执行sql的能力，我们内部用的sql client都是类似我们0.5中direct_execute的方式在当前线程执行sql。
执行时直接调用direct_execute api，调用之前初始化session时设置不检查权限。
root租户提供root用户，可以登录并执行创建tenant，修改系统变量等，供外部使用。

内部所有sql都在本机，当前线程执行，对于不在本机的partition，由内部的转发机制将执行计划发到目的server执行。

## Bootstrap

云版本中自举过程与0.5类似：创建系统表，并插入初始数据。Bootstrap命令通过ob_admin由[bootstrap](#bootstrap) RPC发住任意一台
OB Server，命令中指定rs list (__all_root_table partition list)。

收到命令的Ob Server执行流程如下：

- 检查自己以及rs list中server上都没有在RS的lease中，且上面不存在partition，RPC: [is_empty_server](#is_empty_server)。
  (检查RS是否存在且server是否为空，以防止多次bootstrap)
- 在rs list的server中创建\_\_all_root_table的partition
- 等待主被选出，发送bootstrap命令到RS ([execute_bootstrap](#execute_bootstrap))

RS收到bootstrap命令，执行以下流程：

- 创建核心表(硬编码schema的表)partition：为 \_\_all_database, \_\_all_tablespace, \_\_all_table, \_\_all_column
  \_\_all_join_info, \_\_all_ddl_operation 选择OB Server(就是rs list，因为他们都属于同一tablesapce)，
  发RPC创建，并等待所有partition先出主。如果选出的主与\_\_all_root_table不在同个server则发起改选，
  使他们主都在同一机器。
- 创建其它系统表schema：将以下表依次将schema插入 \_\_all_table,\_\_all_column，并刷新schema。
	- \_\_all_tenant
	- \_\_all_tenant_resource,
	- \_\_all_user,
	- \_\_all_database_privilege,
	- \_\_all_table_privilege,
	- \_\_all_cluster,
	- \_\_all_cluster_stat,
	- \_\_all_sys_param
	- \_\_all_sys_stat,
	- \_\_all_sys_config,
	- \_\_all_sys_config_stat,
	- \_\_all_trigger_event,
	- \_\_all_server
	- \_\_all_meta_table
- 为以上系统表选择partition (除\_\_all_meta_table非第一个partition外，都选rs list)，发RPC创建partition并等待选主完成。
- 插入虚拟表schema，如现在的 \_\_updateserver_session_info, \_\_kvcache_stat等，1.0中我们需要哪些虚拟表，实现中再确定。
- 系统表中插入初始化数据：

```sql
		insert into __all_tablespace (tenant_id, tablesapce_id, tablespace_name, comment) values (1, 1, 'system', 'system tablesapce');
		insert into __all_database (tenatn_id, database_id, database_name, comment) values (1, 1, 'system', 'system database');
		insert into __all_user (tenant_id, user_name, user_id, host, passwd, info, priv_all, priv_alter, priv_create,
				priv_create_user, priv_delete, priv_drop, priv_grant_option, priv_insert, priv_insert,
				priv_update, priv_replace, is_locked)
            		valuese (1, 'root', 1, '%', 'password', 'inner user', 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0);

        insert into __all_database_privilege (tenant_id, user_id, host, database_id, priv_all, priv_alter, priv_create,
				priv_delete, priv_drop, priv_grant_option, priv_insert, priv_update, priv_select, priv_replace)
					values (1, 1, '%', 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1);

		insert into __all_sys_param (tenant_id, name, data_type, value) values (1, 'autocommit', 1, 1),(1, 'tx_isolation', 6, 'READ-COMMITTED')...;

		insert into __all_sys_stat (name, data_type, value) values ('max_used_user_id', 1, 1),
				('max_used_table_id', 1, 3000), ('max_used_database_id', 1, 1), ('max_used_tablespace_id', 1, 1);
```

- 通知刷新schema, 权限等，bootstrap完成。

整个bootstrap过程不是一个事务，如果失败，则需要重新部署整个大集群(清空将所有server)。


大集群重启：

- 各partition自行选主，并在 \_\_all_root_table 上启动RS模块，将RS地址更新到ConfigServer/DNS

- OB Server取得RS地址后，注册并获取core table schema ([fetch_schema](#fetch_schema))

- 读\_\_all_table, \_\_all_column 加载系统表schema。因为已拿到\_\_all_table, \_\_all_column表schema且
  内部sql执行不检查权限，因此，可以直接读这些表。

- 读取\_\_all_tenant表，获取租户列表。

- 读取\_\_all_database, \_\_all_tablespace 加载所有database和tablespace。(此时schema加载完成)

- 读取\_\_all_user，以及\_\_all_database_privilege等权限表，加载权限信息。

- 读所有租户\_\_all_sys_param 读取系统变量值

RS模块启动后，轮询[get_partition_stat](#get_partition_stat)所有server上的partition，确认有多少partition恢复完成(可写)。
并以内部表，所有表为维度在日志中打印是否可服务信息。

## RPC list

*对外提供的：* (return都会包含ResultCode，所以不做描述)

### renew_lease

心跳RPC

- param:
	- int64_t cluster_id;
	- ObServer server;
	- int64_t inner_port;
	- ObServerResourceInfo server_info; // 包含ob version, cpu info, memory info, disk info...
- return:
	- int64_t lease_expire_time;
	- int64_t lease_info_version; // 通过lease_info_version检查是否需要从\_\_all_cluster_stat中更新信息
	- int64_t frozen_version;
	- int64_t schema_version;

### report_root_partition

汇报\_\_all_root_table表partition信息，用于每日合并汇报此partition data_version等。

- param:
	- ObServer self;
	- PartitionReplica replica; // 包含 table_id, partition_id, data_version, data_checksum, column_checksum, role, ...
- return: NONE

### migrate_over

迁移完成后，OB Server发送此命令告诉RS结果。see: [migrate](#migrate)

- param:
	- int64_t table_id;
	- int64_t partition_id;
	- ObServer src;
	- ObServer dest;
	- bool keep_src;
	- int64_t result; // 迁移命令执行结果
- return: NONE

### fetch_schema

获取schema，主要用于server启动过程中向RS取核心表schema。

- param:
	- bool only_core_table; // 是否只取核心表，保留取全部schema的功能是为以后扩展 (一些tools可能需要全部schema)
- result:
	- ObSchemaManager schema;

### query_frozen_stat

获取\_\_all_root_table partition memtable冻结状态，用于两阶段冻结的恢复。

- param:
	- int64_t frozen_version;
- param:
	- int64_t frozen_stat; // 可能的值：FROZEN, NOT_FROZEN, UNKNOWN。
	                            在rs正在进行两阶段冻结以及\_\_all_root_table partition正在回放日志阶段返回UNKNOWN.

### create_tenant
- param:
	- ObTenantInfo tenant_info; // 包含tenant name，租户的属性，资源限制等信息
- return:
	- int64_t tenant_id;

### create_database
- param:
	- int64_t tenant_id;
	- ObDatabaseInfo db_info; // 包含database name，database属性等
- return:
	- int64_t database_id;

### create_table
- param:
	- bool if_not_exist;
	- TableSchema schema; // 包含 tenant_id, database_id, tablespace_id
- return:
	- int64_t table_id;

drop tenant, drop database, drop table, ...等DDL相关RPC不在此一一罗列。

### execute_bootstrap
RS端执行bootstrap流程。

- param:
	- ObIArry< std::pair<int64_t /*cluster_id*/, ObServer> > rs_list;
	  如果 rs_list.count()与配置的集群数不等，则返回失败。
- return: NONE

*依赖的外部RPC：*

### create_partition

bootstrap和create table阶段创建partition。指定cluster_id是为了避免bootstrap阶段创建partition时，
cluster_id与OB Server不匹配。

执行此RPC时，其相关table的schema可能还不存在，所以在执行过程中不能依赖其schema.

- param: 
	- int64_t cluster_id;
	- int64_t table_id;
	- int64_t partition_id;
	- ObIArray<ObServer> member_list;
- return: NONE

### fetch_root_partition

RS主动获取\_\_all_root_tablepartition信息，由RS在启动阶段向\_\_all_root_table表partition的member list获取。
0.5中也有类似过程，通过发force report要求强制汇报，然后再通过汇报获取信息。这里，我们直接做成了同步。

see: [report_root_partition](#report_root_partition)

- param: NONE
- return:
	- PartitionReplica replica;

### migrate

复制/迁移partition.

- param:
	- int64_t table_id;
	- int64_t partition_id;
	- ObServer src; // 源地址
	- ObServer dest; // 目的地址
	- bool keep_src; // 是否保留源partition (复制：True, 迁移：False)
- return: NONE // 调度起了迁移任务立即返回，不用等迁移任务完成，迁移任务完成时，发migrate_over

### migrate_member_change

OB Server复制/迁移的最后阶段向leader请求进行成员变更。如果是迁移(keep_src == False)，则先将
src从成员中踢掉，再加入dest成员 (即两次成员变更)。如果是复制(keep_src == True)，在成员数少于最大成员数的情况下，
直接加入，否则选踢掉一个死掉的(不在leader lease中)成员，再加入新成员。

- param:
	- int64_t table_id;
	- int64_t partition_id;
	- ObServer src;
	- ObServer dest;
	- bool keep_src;
- return: NONE

### prepare_major_freeze

准备冻结阶段，停止一个server上所有leader的写入，并写对应日志。(如已处于prepare_major_freeze状态，则直接返回成功)

- param:
	- int64_t frozen_version;
	- ObIArray<Partition> partition_list; // Partition 中包含table_id, partition_id
- return: NONE

### commit_major_freeze

确认冻结：创建新memtable，允许写入，并写对应日志。(如没处于prepare_major_freeze状态，则直接返加成功)

- param:
	- int64_t frozen_version;
	- ObIArray<Partition> partition_list;
- return: NONE

### abort_major_freeze

取消冻结：允许写入，并写对应日志。(如没处于prepare_major_freeze状态，则直接返加成功)

- param:
	- int64_t frozen_version;
	- ObIArray<Partition> partition_list;
- return: NONE

### get_member_list

向partition leader获取其成员列表。

- param:
	- int64_t table_id;
	- int64_t partition_id;
- return:
	- ObIArray<ObServer> member_list;

### switch_leader

对一个partition发起有主选举，将leader改选到其它server。

- param:
	- int64_t table_id;
	- int64_t partition_id;
	- ObServer new_leader_addr;
- return: NONE

### switch_schema

RS主动通知名Server刷新schema。

- param:
	- int64_t schema_version;
- return: NONE

### bootstrap

ob_admin发起的执行bootstrap的RPC。参数与返回值与 [execute_bootstrap](#execute_bootstrap) 一致。

### is_empty_server

bootstrap过程中检查server是否为空，为避免多次bootstrap。收到此RPC，
做以下检查：1.是否在RS lease中(即：是否注册上RS)，2.server中是否存在partition。

- param: NONE
- return:
	- bool is_empty;

### get_partition_stat

查询一个机器上的partition是否恢复完成。

- param:
	- ObIArray<Partition> partition_ist;
	- ObIArray<PartitionStat> stat_list; // PartitionStat包含table_id, partition_id, role以及partition_stat。
	                                       // partition_stat表示日志是否恢复完成，是否可写等状态
