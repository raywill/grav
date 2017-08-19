```
* SETP_0.1:alter_system  
OBS接受到用户发起的bootstarp命令，语法格式为:ALTER SYSTEM BOOTSTRAP ZONE='ZONE1' SERVER='10.232.144.13:2209', ZONE='zone2' SERVER='10.244.4.14:2203'   
 
* SETP_0.2:boot_strap_rpc  
SQL端处理命令，向接受alter system命令的obsesrver发起bootstrap RPC命令，传入参数 rs_list; 
RPC_S(PR5 bootstrap, OB_BOOTSTRAP, (ObServerInfoList));

* SETP_1:obs收到bootstrap的rpc命令，开始prepare_bootstrap操作：
  * STEP_1.1:check_bootstrap_rs_list
   	 检查rs_list里面指定的SERVER地址，不能出现两个SERVER位于同一个ZONE中的情况；因为每个ZONE里面只有一个副本；
  * STEP_1.2:check_is_all_server_empty
    RPC检查rs_list，必须所有rs都为空obs；  
    RPC_S(PR5 is_empty_server, OB_IS_EMPTY_SERVER, Bool);

  * STEP_1.3:create_partition  
    创建__all_core_table的partition,发送异步RPC到RS,创建partition. 为了加快选主进度，会直接指定rs_list中第一个obs为主partition；  
    RPC_AP(PR5 create_partition_batch, OB_CREATE_PARTITION_BATCH, (ObCreatePartitionBatchArg), ObCreatePartitionBatchRes);

  * STEP_1.4:wait_elect_master_partition  
    确认本机中该partition的主已经选出；  
    相关类
  		* ObPartitionTableOperator 负责操作partition相关的内部表；包括两个partition地址：
  		* ObInMemoryPartitionTable ：保存___all_core_table的地址
  		* ObPersistentPartitionTable: 提供访问patition表的接口；
  		* ObPartitionTableProxy： 访问partition表的接口，包括__all_core_talbe, __all_root_table. __all_meta_table;
  		
  * STEP_1.5:prepare_bootstrap
    至此，prepare的工作已经做完了。prepare的工作主要就是检查所有server的状态为空，创建出__all_core_table，并选出主partition来，主partition所在的RS就是主RS。下一步就是往主RS发送execute_bootstrap的RPC命令了。
    RPC_S(PR5 execute_bootstrap, OB_EXECUTE_BOOTSTRAP, (ObServerInfoList));
    
* STEP_2：execute_bootstrap
  主RS收到execute_bootstrap的命令，开始执行操作
  
  * STEP_2.1:check_is_already_bootstrap  
   检查当前schema版本，是否为OB_CORE_SCHEMA_VERSION     相关类：ObMultiVersionSchemaService， ObSchemaGetterGuard 要配合一起使用；
 
  * STEP_2.2:check_bootstrap_rs_list
    检查RS_LIST，不能属于同一个zone
 
  * STEP_2.3:add_rs_list
    将rs_list加入到ddl_server中的server_manager中；
  
  * STEP_2.4:wait_all_rs_online
    在一个leases周期内，循环检查，直到所有的rs状态都是active；RS四种状态：alive/lease_expired/permanent_offline/max
  
  * STEP_2.5:init_global_stat
    初始化core_schema_version.这里会往__all_core_table里面写入三行数据。inner sql会发给__all_core_table所在的主；
    INSERT INTO __all_core_table (table_name, row_id, column_name, column_value, gmt_create, gmt_modified) VALUES ('__all_global_stat', 0, 'core_schema_version', '1', usec_to_time(1472534133389119), usec_to_time(1472534133389119));
    INSERT INTO __all_core_table (table_name, row_id, column_name, column_value, gmt_create, gmt_modified) VALUES ('__all_global_stat', 0, 'try_frozen_version', '1', usec_to_time(1472534133389119), usec_to_time(1472534133389119));
    INSERT INTO __all_core_table (table_name, row_id, column_name, column_value, gmt_create, gmt_modified) VALUES ('__all_global_stat', 0, 'frozen_version', '1', usec_to_time(1472534133389119), usec_to_time(1472534133389119));
    
    相关类：
      * ObGlobalStatProxy， 负责往core_table中写入global_stat相关的内容，比如：core_schema_version/frozen_version/try_forzen_version;  	  * ObCoreTableProxy: 专门负责操作__all_core_table的类；  	  * ObDMLSqlSplicer：sql语句拼接操作类；        * ObDMLExecHelper：负责执行dml语句；  
      
  * STEP_2.6:construct_schema
    生成所有表的schema.表的类型包括：核心表，系统表，虚拟表，视图。这些表的schema是通过脚本文件直接生成的；
 
  * STEP_2.7:broadcast_sys_schema
    将第六步生成的schema RPC 广播给所有的RS; 收到命令的RS会用这个schema来初始化schame_cache;
    RPC_S(PR5 broadcast_sys_schema, OB_BROADCAST_SYS_SCHEMA，common::ObSArray<share::schema::ObTableSchema>));
 
  * STEP_2.8:create_all_partitions
    收集核心表和系统表的信息，统一调用 ObPartitionCreator.execute接口，统一创建所有partition.然后等待这些partition的主选出
    RPC_AP(PR5 create_partition_batch, OB_CREATE_PARTITION_BATCH, (ObCreatePartitionBatchArg), ObCreatePartitionBatchRes);
    设计点：所有table_group的表会在同一个OBS上面。__all_meta 表中的一个标记副本（flag_replica），版本号为-1，在创建完partition没有回报的时候，也知道同一个table_group的表在哪里；创建成功并汇报后，这个标记副本就会被更新。
 
  * STEP_2.9:create_all_schema
    将所有表的schema写入到内部表中，其中区分核心表和非核心表。核心表的schame写入到__all_core_table中，非核心表的schema写入到__all_table和__all_column表里面；将日志计入__all_ddl_operation中。
    core_schema_version的作用：核心表发生变更时的schema号。 __all_ddl_operation发生变更的时候， 不能简单的直接增量刷__all_ddl_operation, 而是应该刷__all_core_table; 

  * STEP_2.10:create_sys_unit_config
    始化系统unit config信息；产生一条内部SQL：
    INSERT INTO __all_unit_config (unit_config_id, name, max_cpu, min_cpu, max_memory, min_memory, max_disk_size, max_iops, min_iops, max_session_num, gmt_modified) VALUES (1, X'7379735F756E69745F636F6E666967', 5.000000, 2.500000, 16492674416, 13743895347, 5368709120,
10000, 5000, 9223372036854775807, now(6)) ON DUPLICATE KEY UPDATE name = X'7379735F756E69745F636F6E666967', max_cpu = 5.000000, min_cpu = 2.500000, max_memory = 16492674416, min_memory = 13743895347, max_disk_size = 5368709120, max_iops = 10000, min_iops = 5000, max_session_num = 9223372036854775807, gmt_modified = now(6)")
  
  * STEP_2.11:create_sys_resource_pool
    创建系统resource pool;产生一条内部SQL：
    INSERT INTO __all_resource_pool (resource_pool_id, name, unit_count, unit_config_id, zone_list, tenant_id, gmt_modified) VALUES (1, X'7379735F706F6F6C', 1, 1, 'zone1', -1, now(6)) ON DUPLICATE KEY UPDATE name = X'7379735F706F6F6C', unit_count = 1, unit_config_id = 1, zone_list = 'zone1', tenant_id = -1, gmt_modified = now(6)")
  
  * STEP_2.12:create_sys_tenant
    创建sys租户；同时初始化该tenant的一些属性，产生多条内部SQL:
    REPLACE INTO __all_tenant (tenant_id, tenant_name, replica_num, locked, zone_list, primary_zone, info, collation_type, read_only, rewrite_merge_version, gmt_modified) VALUES (1, X'737973', 1, 0, 'zone1', 'zone1', 'system tenant', 45, 0, 0, now(6))")
    往__all_tablegroup表中插入sys tablegroup的行；
    创建租户的五大database： oceanbase/mysql/information_schema/recyclebin/test
    权限信息写入了__ALL_DATABASE_PRIVILEGE; 系统用户拥有全部的权限；
    新建该tenant的root用户； 插入数据到__all_user/__all_user_history/__all_ddl_operation
    初始化__al_sys_variable表；把所有系统变量写入表中；
    初始化__all_sys_stat表.如果是系统租户的话，写入：ob_max_used_tenant_id.ob_max_used_unit_config_id_ ob_max_used_server_id_、ob_max_used_unit_id_等。
    如果是普通用户的话，写入：ob_max_used_database_id_、ob_max_used_tablegroup_id、ob_max_used_sequence_id等等
    初始化__all_charset, 字符集表
    初始化__all_collation，collation表
    初始化__all_privilege，权限信息表
 
  * STEP_2.13:init_server_id
    更新__all_sys_stat表中最大的server_id;产生一条内部SQL：
    UPDATE __all_sys_stat SET VALUE = '1', gmt_modified = now(6) WHERE ZONE = '' AND NAME = 'ob_max_used_server_id' AND TENANT_ID = 1"
 
  * STEP_2.14:init_all_zone_table
    初始化__all_zone表。 会产生一些列的插入操作；比如：
    INSERT INTO __all_zone (zone, name, value, info, gmt_modified) VALUES('', 'merge_status', 0, 'IDLE', now(6))，不指定zone的语句用来修改>全局的状态信息；
    比如：
    INSERT INTO __all_zone (zone, name, value, info, gmt_modified) VALUES('zone1', 'status', 2, 'ACTIVE', now(6)) 指定zone的语句用来修改特定zone的状态信息；

  * STEP_2.15:wait_all_rs_in_service
    所有内部表操作结束，等待RS的状态都切换成in service;
  
  * STEP_2.16:execute_bootstrap
    BOOTSTRAP阶段的工作已经做完。接下来就是RS的一些任务线程的启动；

```
