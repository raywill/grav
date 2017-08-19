# RS常用分析方法和经验

## 1. 负载均衡

### unit均衡情况

先看有哪些租户。
<pre>
mysql> select tenant_id, tenant_name from __all_tenant where tenant_id>1000;
+-----------+-------------+
| tenant_id | tenant_name |
+-----------+-------------+
|      1001 | tt1         |
|      1002 | tt2         |
|      1003 | tt3         |
|      1004 | tt4         |
|      1005 | tt5         |
|      1006 | tt6         |
+-----------+-------------+
6 rows in set (0.02 sec)
</pre>


然后看unit的分布情况。

<pre>
mysql> select count(*), svr_ip from __all_unit group by svr_ip;
+----------+----------------+
| count(*) | svr_ip         |
+----------+----------------+
|        5 | 10.217.176.113 |
|        5 | 10.217.176.117 |
|        5 | 10.217.176.119 |
|        5 | 10.217.176.120 |
|        5 | 10.217.176.126 |
|        5 | 10.217.176.128 |
|        3 | 10.248.1.66    |
|        4 | 10.248.1.67    |
|        4 | 10.248.1.69    |
|        4 | 10.248.1.90    |
+----------+----------------+
10 rows in set (0.01 sec)
</pre>

### 复本均衡情况

### 副本在本租户内的均衡情况

<pre>
mysql> select tenant_id, unit_id, count(*) from __all_meta_table group by tenant_id, unit_id order by count(*);
+-----------+---------+----------+
| tenant_id | unit_id | count(*) |
+-----------+---------+----------+
|         1 |       1 |     1000 |
|         1 |       3 |     1000 |
|         1 |       2 |     1000 |
|      1002 |    1016 |     5333 |
|      1002 |    1012 |     5333 |
|      1002 |    1013 |     5333 |
|      1002 |    1010 |     5333 |
|      1002 |    1015 |     5333 |
|      1002 |    1018 |     5333 |
|      1002 |    1011 |     5334 |
|      1002 |    1014 |     5334 |
|      1002 |    1017 |     5334 |
|      1003 |    1020 |     8334 |
|      1003 |    1027 |     8666 |
|      1003 |    1024 |     8666 |
|      1003 |    1021 |     8666 |
|      1003 |    1023 |     8667 |
|      1003 |    1019 |     8667 |
|      1003 |    1026 |     8667 |
|      1003 |    1022 |     8667 |
|      1003 |    1025 |     8667 |
|      1004 |    1030 |    14667 |
|      1004 |    1029 |    15921 |
|      1004 |    1032 |    15921 |
|      1004 |    1035 |    15921 |
|      1004 |    1033 |    15921 |
|      1004 |    1036 |    15921 |
|      1004 |    1028 |    15922 |
|      1004 |    1034 |    15922 |
|      1004 |    1031 |    15922 |
|      1001 |    1003 |    45332 |
|      1001 |    1002 |    48508 |
|      1001 |    1008 |    48508 |
|      1001 |    1005 |    48508 |
|      1001 |    1006 |    48509 |
|      1001 |    1009 |    48509 |
|      1001 |    1004 |    48511 |
|      1001 |    1001 |    48511 |
|      1001 |    1007 |    48511 |
|      1005 |    1037 |    65664 |
|      1005 |    1038 |    65885 |
|      1005 |    1039 |    67452 |
+-----------+---------+----------+
42 rows in set (21.76 sec)
</pre>

### 副本在机器间的均衡情况

<pre>
mysql> select svr_ip, count(*) count from __all_meta_table group by svr_ip order by count;
+----------------+--------+
| svr_ip         | count  |
+----------------+--------+
| 10.217.176.113 |  62507 |
| 10.248.1.67    |  62507 |
| 10.217.176.117 |  65432 |
| 10.248.1.66    |  65432 |
| 10.217.176.116 |  73667 |
| 10.217.176.128 |  78429 |
| 10.248.1.69    |  78431 |
| 10.217.176.126 |  79431 |
| 10.217.176.119 |  95586 |
| 10.248.1.90    |  97374 |
| 10.217.176.120 | 144317 |
+----------------+--------+
11 rows in set (22.34 sec)
</pre>

### 副本在机器和unit上的均衡情况

<pre>
mysql> select /*+ query_timeout(600000000) */ count(*) count, unit_id, svr_ip, tenant_id from __all_meta_table group by unit_id, svr_ip, tenant_id order by svr_ip, count;
+-------+---------+----------------+-----------+
| count | unit_id | svr_ip         | tenant_id |
+-------+---------+----------------+-----------+
|  5333 |    1010 | 10.217.176.113 |      1002 |
|  8666 |    1021 | 10.217.176.113 |      1003 |
| 48508 |    1002 | 10.217.176.113 |      1001 |
|  5334 |    1011 | 10.217.176.116 |      1002 |
|  8334 |    1020 | 10.217.176.116 |      1003 |
| 14667 |    1030 | 10.217.176.116 |      1004 |
| 45332 |    1003 | 10.217.176.116 |      1001 |
|  1000 |       1 | 10.217.176.117 |         1 |
| 15921 |    1029 | 10.217.176.117 |      1004 |
| 48511 |    1001 | 10.217.176.117 |      1001 |
|  5333 |    1012 | 10.217.176.119 |      1002 |
|  8667 |    1019 | 10.217.176.119 |      1003 |
| 15922 |    1028 | 10.217.176.119 |      1004 |
| 65664 |    1037 | 10.217.176.119 |      1005 |
|  5334 |    1014 | 10.217.176.120 |      1002 |
|  8667 |    1022 | 10.217.176.120 |      1003 |
| 15922 |    1031 | 10.217.176.120 |      1004 |
| 48509 |    1006 | 10.217.176.120 |      1001 |
| 65885 |    1038 | 10.217.176.120 |      1005 |
|  1000 |       2 | 10.217.176.126 |         1 |
|  5333 |    1015 | 10.217.176.126 |      1002 |
|  8666 |    1024 | 10.217.176.126 |      1003 |
| 15921 |    1033 | 10.217.176.126 |      1004 |
| 48511 |    1004 | 10.217.176.126 |      1001 |
|  5333 |    1013 | 10.217.176.128 |      1002 |
|  8667 |    1023 | 10.217.176.128 |      1003 |
| 15921 |    1032 | 10.217.176.128 |      1004 |
| 48508 |    1005 | 10.217.176.128 |      1001 |
|  1000 |       3 | 10.248.1.66    |         1 |
| 15921 |    1035 | 10.248.1.66    |      1004 |
| 48511 |    1007 | 10.248.1.66    |      1001 |
|  5333 |    1016 | 10.248.1.67    |      1002 |
|  8666 |    1027 | 10.248.1.67    |      1003 |
| 48508 |    1008 | 10.248.1.67    |      1001 |
|  5334 |    1017 | 10.248.1.69    |      1002 |
|  8667 |    1026 | 10.248.1.69    |      1003 |
| 15921 |    1036 | 10.248.1.69    |      1004 |
| 48509 |    1009 | 10.248.1.69    |      1001 |
|  5333 |    1018 | 10.248.1.90    |      1002 |
|  8667 |    1025 | 10.248.1.90    |      1003 |
| 15922 |    1034 | 10.248.1.90    |      1004 |
| 67452 |    1039 | 10.248.1.90    |      1005 |
+-------+---------+----------------+-----------+
42 rows in set (24.54 sec)
</pre>

可以看出，之所以120这台机器的副本比较多，是因为1006，1038两个*大* unit 撞到了一起。出现这种情况符合目前的设计。

### Leader在机器间的分布情况

<pre>
mysql> select svr_ip, count(*) from __all_meta_table where role=1 group by svr_ip;
+----------------+----------+
| svr_ip         | count(*) |
+----------------+----------+
| 10.217.176.113 |    27215 |
| 10.217.176.117 |    31636 |
| 10.248.1.66    |    33793 |
| 10.248.1.67    |    35292 |
+----------------+----------+
4 rows in set (21.21 sec)
</pre>

## 2. 宕机处理

### 查看server状态和event

<pre>
mysql> select gmt_modified, svr_ip, svr_port, zone, status from __all_server where status != 'active';
+----------------------------+----------------+----------+-------+----------+
| gmt_modified               | svr_ip         | svr_port | zone  | status   |
+----------------------------+----------------+----------+-------+----------+
| 2016-10-20 22:36:25.297147 | 10.217.176.116 |    41628 | zone1 | inactive |
+----------------------------+----------------+----------+-------+----------+
1 row in set (0.01 sec)
</pre>

可以看到116这台机器永久下线了。

<pre>
mysql> select * from __all_rootservice_event_history;
...
| 2016-10-20 23:36:25.363701 | server       | permanent_offline        | server             | "10.217.176.116:41628" |                     |                        |           |                        |       |        |       |        |       |        |            |
| 2016-10-20 23:50:01.958425 | unit         | migrate_unit             | unit_id            | 1003                   | migrate_from_server | "10.217.176.116:41628" | server    | "10.217.176.119:41630" |       |        |       |        |       |        |            |
| 2016-10-20 23:50:02.005343 | unit         | migrate_unit             | unit_id            | 1011                   | migrate_from_server | "10.217.176.116:41628" | server    | "10.217.176.117:41632" |       |        |       |        |       |        |            |
| 2016-10-20 23:50:02.037975 | unit         | migrate_unit             | unit_id            | 1020                   | migrate_from_server | "10.217.176.116:41628" | server    | "10.217.176.117:41632" |       |        |       |        |       |        |            |
| 2016-10-20 23:50:02.068963 | unit         | migrate_unit             | unit_id            | 1030                   | migrate_from_server | "10.217.176.116:41628" | server    | "10.217.176.113:41626" |       |        |       |        |       |        |            |
</pre>

可以看到永久下线以后，有4个unit需要迁移出去。

### 迁移动作

这4个unit要迁移到哪里去呢？

<pre>
mysql> select * from __all_unit where unit_id in (1011, 1020, 1030, 1003);
+----------------------------+----------------------------+---------+------------------+----------+-------+----------------+----------+---------------------+-----------------------+
| gmt_create                 | gmt_modified               | unit_id | resource_pool_id | group_id | zone  | svr_ip         | svr_port | migrate_from_svr_ip | migrate_from_svr_port |
+----------------------------+----------------------------+---------+------------------+----------+-------+----------------+----------+---------------------+-----------------------+
| 2016-10-20 15:41:27.660879 | 2016-10-20 23:50:01.889166 |    1003 |             1001 |        2 | zone1 | 10.217.176.119 |    41630 | 10.217.176.116      |                 41628 |
| 2016-10-20 16:27:50.454593 | 2016-10-20 23:50:01.972951 |    1011 |             1002 |        1 | zone1 | 10.217.176.117 |    41632 | 10.217.176.116      |                 41628 |
| 2016-10-20 16:54:20.973265 | 2016-10-20 23:50:02.006644 |    1020 |             1003 |        1 | zone1 | 10.217.176.117 |    41632 | 10.217.176.116      |                 41628 |
| 2016-10-20 17:27:11.747269 | 2016-10-20 23:50:02.039476 |    1030 |             1004 |        2 | zone1 | 10.217.176.113 |    41626 | 10.217.176.116      |                 41628 |
+----------------------------+----------------------------+---------+------------------+----------+-------+----------------+----------+---------------------+-----------------------+
4 rows in set (0.00 sec)
</pre>

### 迁移进度

那么，这4个unit中的副本实际迁移到什么程度了呢？

<pre>
mysql> select count(*) count, unit_id, svr_ip, tenant_id from __all_meta_table where unit_id in (1011, 1020, 1030, 1003) group by unit_id, svr_ip, tenant_id;
+-------+---------+----------------+-----------+
| count | unit_id | svr_ip         | tenant_id |
+-------+---------+----------------+-----------+
| 14667 |    1030 | 10.217.176.116 |      1004 |
|  8334 |    1020 | 10.217.176.116 |      1003 |
| 45332 |    1003 | 10.217.176.116 |      1001 |
|  5334 |    1011 | 10.217.176.116 |      1002 |
+-------+---------+----------------+-----------+
4 rows in set (32.30 sec)
</pre>

结果发现，他们实际上还是位于116上。为什么呢？

<pre>
mysql> select Z.count, count(*) from (select X.*, count(*) count from (select tenant_id, table_id, partition_id from __all_meta_table where unit_id in (1011, 1020, 1030, 1003) ) X inner join __all_meta_table Y using (tenant_id, table_id, partition_id) group by X.tenant_id, X.table_id, X.partition_id) Z;
+-------+----------+
| count | count(*) |
+-------+----------+
|     3 |    73667 |
+-------+----------+
1 row in set (50.14 sec)
</pre>

看起来这些partition的副本数是正常的，都是3个。

<pre>
mysql> select X.* from (select tenant_id, table_id, partition_id from __all_meta_table where unit_id in (1011, 1020, 1030, 1003) ) X inner join __all_meta_table Y using (tenant_id, table_id, partition_id) where Y.role=1 limit 10;
Empty set (45.29 sec)
</pre>

原来，因为这些partition都是无主的。

## 3. 检查partition状态

### schema中有多少partition？

<pre>
mysql> select sum(part_num*sub_part_num) from __all_virtual_core_all_table where table_type in (0,3);
+----------------------------+
| sum(part_num*sub_part_num) |
+----------------------------+
|                         34 |
+----------------------------+
1 row in set (0.01 sec)

mysql> select sum(part_num*sub_part_num) from __all_table where table_type in (0,3);
+----------------------------+
| sum(part_num*sub_part_num) |
+----------------------------+
|                     286452 |
+----------------------------+
1 row in set (0.08 sec)
</pre>

### partition table有多少partition？

<pre>
mysql> select count(*) from (select distinct tenant_id, table_id, partition_id from __all_virtual_core_root_table) X;
+----------+
| count(*) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from (select distinct tenant_id, table_id, partition_id from __all_root_table) X;
+----------+
| count(*) |
+----------+
|      491 |
+----------+
1 row in set (0.04 sec)

mysql> select count(*) from (select distinct tenant_id, table_id, partition_id from __all_meta_table) X;
+----------+
| count(*) |
+----------+
|   303746 |
+----------+
1 row in set (21.28 sec)
</pre>

因为建表的时候有失败，所以partition table里有一些partition所属的table已经删除，但副本尚未删除。所以partition table里的partition数目比schema中记录的要多。

### 有多少无主的partition？

<pre>
mysql> select count(*) from ((select distinct tenant_id, table_id, partition_id from __all_meta_table) except (select distinct tenant_id, table_id, partition_id from __all_meta_table where role=1) ) X;
+----------+
| count(*) |
+----------+
|   175810 |
+----------+
1 row in set (40.42 sec)
</pre>

## 4. 资源分配情况

### server资源使用
<pre>
mysql> select * from __all_virtual_server_stat;
+----------------+----------+-------+-----------+--------------+----------------------+-------------+--------------+----------------------+------------+---------------+-----------------------+----------+--------------------+----------------+--------------+
| svr_ip         | svr_port | zone  | cpu_total | cpu_assigned | cpu_assigned_percent | mem_total   | mem_assigned | mem_assigned_percent | disk_total | disk_assigned | disk_assigned_percent | unit_num | migrating_unit_num | merged_version | leader_count |
+----------------+----------+-------+-----------+--------------+----------------------+-------------+--------------+----------------------+------------+---------------+-----------------------+----------+--------------------+----------------+--------------+
| 10.217.176.113 |    41626 | zone1 |        22 |           20 |                   90 | 54975581389 |   5368709120 |                    9 | 5368709120 |    2684354560 |                    50 |        5 |                  0 |              2 |         NULL |
| 10.217.176.116 |    41628 | zone1 |      NULL |           16 |                 NULL |        NULL |   4294967296 |                 NULL |       NULL |    2147483648 |                  NULL |        4 |                  4 |              0 |         NULL |
| 10.217.176.117 |    41632 | zone1 |        22 |           21 |                   95 | 54975581389 |  20787641712 |                   37 | 5368709120 |    7516192768 |                   140 |        5 |                  0 |              2 |         NULL |
| 10.217.176.119 |    41630 | zone1 |        22 |           20 |                   90 | 54975581389 |   5368709120 |                    9 | 5368709120 |    2684354560 |                    50 |        5 |                  0 |              2 |         NULL |
| 10.217.176.120 |    41636 | zone2 |        22 |           20 |                   90 | 54975581389 |   5368709120 |                    9 | 5368709120 |    2684354560 |                    50 |        5 |                  0 |              0 |         NULL |
| 10.217.176.126 |    41638 | zone2 |        22 |           21 |                   95 | 54975581389 |  20787641712 |                   37 | 5368709120 |    7516192768 |                   140 |        5 |                  0 |              2 |         NULL |
| 10.217.176.128 |    41634 | zone2 |        22 |           20 |                   90 | 54975581389 |   5368709120 |                    9 | 5368709120 |    2684354560 |                    50 |        5 |                  0 |              2 |         NULL |
| 10.248.1.66    |    41646 | zone3 |        22 |           13 |                   59 | 54975581389 |  18640158064 |                   33 | 5368709120 |    6442450944 |                   120 |        3 |                  0 |              2 |         NULL |
| 10.248.1.67    |    41640 | zone3 |        22 |           16 |                   72 | 54975581389 |   4294967296 |                    7 | 5368709120 |    2147483648 |                    40 |        4 |                  0 |              2 |         NULL |
| 10.248.1.69    |    41642 | zone3 |        22 |           16 |                   72 | 54975581389 |   4294967296 |                    7 | 5368709120 |    2147483648 |                    40 |        4 |                  0 |              0 |         NULL |
| 10.248.1.90    |    41644 | zone3 |        22 |           16 |                   72 | 54975581389 |   4294967296 |                    7 | 5368709120 |    2147483648 |                    40 |        4 |                  0 |              2 |         NULL |
+----------------+----------+-------+-----------+--------------+----------------------+-------------+--------------+----------------------+------------+---------------+-----------------------+----------+--------------------+----------------+--------------+
11 rows in set (0.00 sec)
</pre>

### 租户的资源总量
<pre>
mysql> select tenant_id, count(*) total_unit, sum(max_cpu), sum(min_cpu), sum(max_memory), sum(min_memory), sum(max_iops), sum(min_iops), sum(max_disk_size), sum(max_session_num) from __all_unit_config X inner join __all_unit Y inner join __all_resource_pool Z on Y.resource_pool_id=Z.resource_pool_id and X.unit_config_id = Z.unit_config_id group by tenant_id;
+-----------+------------+--------------+--------------+-----------------+-----------------+---------------+---------------+--------------------+----------------------+
| tenant_id | total_unit | sum(max_cpu) | sum(min_cpu) | sum(max_memory) | sum(min_memory) | sum(max_iops) | sum(min_iops) | sum(max_disk_size) | sum(max_session_num) |
+-----------+------------+--------------+--------------+-----------------+-----------------+---------------+---------------+--------------------+----------------------+
|         1 |          3 |           15 |          7.5 |     49478023248 |     41231686041 |         30000 |         15000 |        16106127360 | 27670116110564327421 |
|      1001 |          9 |           36 |            9 |      9663676416 |      9663676416 |          1152 |          1152 |         4831838208 |                  576 |
|      1002 |          9 |           36 |            9 |      9663676416 |      9663676416 |          1152 |          1152 |         4831838208 |                  576 |
|      1003 |          9 |           36 |            9 |      9663676416 |      9663676416 |          1152 |          1152 |         4831838208 |                  576 |
|      1004 |          9 |           36 |            9 |      9663676416 |      9663676416 |          1152 |          1152 |         4831838208 |                  576 |
|      1005 |          3 |           12 |            3 |      3221225472 |      3221225472 |           384 |           384 |         1610612736 |                  192 |
|      1006 |          3 |           12 |            3 |      3221225472 |      3221225472 |           384 |           384 |         1610612736 |                  192 |
+-----------+------------+--------------+--------------+-----------------+-----------------+---------------+---------------+--------------------+----------------------+
7 rows in set (0.03 sec)
</pre>

## 5. leader coordinator
在开启系统配置项enable_auto_leader_switch时，leader coordinator完成如下几个功能：
### leader在每台server上的分布数量
读取`__all_virtual_server_stat`，获取结果如下：
<pre>
mysql> select svr_ip, leader_count, zone from oceanbase.__all_virtual_server_stat where leader_count > 0 order by zone;
+-------------+--------------+------+
| svr_ip      | leader_count | zone |
+-------------+--------------+------+
| 100.88.8.47 |          557 | z1   |
| 100.88.8.48 |         3614 | z1   |
| 100.88.8.50 |         3614 | z1   |
+-------------+--------------+------+
3 rows in set (0.00 sec)
</pre>
同时，系统表partition table中也可以查询到leader在各server的分布情况，可将结果读出进行比照：

<pre>
__all_root_table表中leader数量信息如下：

mysql> select count(*), svr_ip, zone from oceanbase.__all_root_table where role = 1 group by svr_ip order by zone;
+----------+-------------+------+
| count(*) | svr_ip      | zone |
+----------+-------------+------+
|      550 | 100.88.8.47 | z1   |
|        1 | 100.88.8.48 | z1   |
|        1 | 100.88.8.65 | z3   |
+----------+-------------+------+
3 rows in set (0.02 sec)

__all_meta_table表中leader数量信息如下：

mysql> select count(*), svr_ip, zone from oceanbase.__all_meta_table where role = 1 group by svr_ip order by zone;
+----------+-------------+------+
| count(*) | svr_ip      | zone |
+----------+-------------+------+
|        5 | 100.88.8.47 | z1   |
|     3613 | 100.88.8.48 | z1   |
|     3613 | 100.88.8.65 | z3   |
+----------+-------------+------+
3 rows in set (0.16 sec)
</pre>

将`__all_root_table` 和 `__all_meta_table` 表中数据相加后与 `__all_virtual_server_stat` 中的值进行比照。   
注： `__all_core_table` 和 `__all_root_table` 表的leader信息没有存储在 `__all_root_table` 或 `__all_meta_table` 中，因此， `__all_virtual_server_stat` 中leader的数量总是比 `__all_root_table` 和 `__all_meta_table` 表中leader的数量多2。

### partition的leader切到primary zone

<pre>
首先查询partition所述的primary zone:
OceanBase (root@oceanbase)> select table_name, table_id, primary_zone from oceanbase.__all_table where table_name = 'wenduo_test' and tenant_id = 1;
+-------------+---------------+--------------+
| table_name  | table_id      | primary_zone |
+-------------+---------------+--------------+
| wenduo_test | 1099511677777 |              |
+-------------+---------------+--------------+
如果__all_table表中primary zone为空，查询该partition所在database的primary zone：
OceanBase (root@oceanbase)> select database_name, primary_zone  from oceanbase.__all_database where tenant_id = 1 and database_name = 'oceanbase';
+---------------+--------------+
| database_name | primary_zone |
+---------------+--------------+
| oceanbase     | NULL         |
+---------------+--------------+
1 row in set (0.01 sec)
如果__all_database表中primary zone为空，查询该partition所在的tenant的primary zone：
OceanBase (root@oceanbase)> select tenant_id, tenant_name, primary_zone  from oceanbase.__all_tenant where tenant_name = 'sys';
+-----------+-------------+--------------+
| tenant_id | tenant_name | primary_zone |
+-----------+-------------+--------------+
|         1 | sys         | zone1        |
+-----------+-------------+--------------+
1 row in set (0.00 sec)
获取到这个partition的primary zone为zone1。
查询table partition中该partition的leader所在zone：
OceanBase (root@oceanbase)> select tenant_id, table_id, partition_id, svr_ip, zone, role from oceanbase.__all_meta_table where tenant_id = 1 and table_id = 1099511677777 and role = 1;
+-----------+---------------+--------------+--------------+-------+------+
| tenant_id | table_id      | partition_id | svr_ip       | zone  | role |
+-----------+---------------+--------------+--------------+-------+------+
|         1 | 1099511677777 |            0 | 10.125.224.5 | zone1 |    1 |
+-----------+---------------+--------------+--------------+-------+------+
1 row in set (0.01 sec)
</pre>

### 同一partition group的partition的leader切到同一台observer

<pre>
首先获取系统内的table group：
OceanBase (root@oceanbase)> select tenant_id, tablegroup_id, tablegroup_name from oceanbase.__all_tablegroup;
+-----------+------------------+-----------------+
| tenant_id | tablegroup_id    | tablegroup_name |
+-----------+------------------+-----------------+
|         1 |    1099511627777 | oceanbase       |
|      1001 | 1100611139403777 | oceanbase       |
+-----------+------------------+-----------------+
2 rows in set (0.05 sec)
获取tenant_id = 1, tablegroup_id = 1099511627777的table：
OceanBase (root@oceanbase)> select table_id, table_name, tablegroup_id from oceanbase.__all_table where tenant_id= 1 and tablegroup_id = 1099511627777 limit 10;
+---------------+--------------------------+---------------+
| table_id      | table_name               | tablegroup_id |
+---------------+--------------------------+---------------+
| 1099511627877 | __all_meta_table         | 1099511627777 |
| 1099511627878 | __all_user               | 1099511627777 |
| 1099511627879 | __all_user_history       | 1099511627777 |
| 1099511627880 | __all_database           | 1099511627777 |
| 1099511627881 | __all_database_history   | 1099511627777 |
| 1099511627882 | __all_tablegroup         | 1099511627777 |
| 1099511627883 | __all_tablegroup_history | 1099511627777 |
| 1099511627884 | __all_tenant             | 1099511627777 |
| 1099511627885 | __all_tenant_history     | 1099511627777 |
| 1099511627886 | __all_table_privilege    | 1099511627777 |
+---------------+--------------------------+---------------+
10 rows in set (0.02 sec)
查询这些table的partition_id = 0的partition group的leader：
OceanBase (root@oceanbase)> select svr_ip, svr_port, table_id, partition_id, role from oceanbase.__all_root_table where partition_id = 0 and table_id in (1099511627877, 1099511627878, 1099511627879, 1099511627880, 1099511627881, 1099511627882, 1099511627883, 1099511627884, 1099511627885, 1099511627886) and role = 1;
+--------------+----------+---------------+--------------+------+
| svr_ip       | svr_port | table_id      | partition_id | role |
+--------------+----------+---------------+--------------+------+
| 10.125.224.5 |    56100 | 1099511627877 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627878 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627879 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627880 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627881 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627882 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627883 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627884 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627885 |            0 |    1 |
| 10.125.224.5 |    56100 | 1099511627886 |            0 |    1 |
+--------------+----------+---------------+--------------+------+
10 rows in set (0.02 sec)
</pre>

### 主 Rootserver 的位置

主Rootserver的位置，就是__all_core_table partition leader 的位置, 简易查询方法:

<pre>
mysql> select svr_ip, svr_port, id, zone, inner_port, with_rootserver, status from __all_server;
+---------------+----------+----+------+------------+-----------------+--------+
| svr_ip        | svr_port | id | zone | inner_port | with_rootserver | status |
+---------------+----------+----+------+------------+-----------------+--------+
| 100.81.252.10 |    10233 |  1 | z1   |      10268 |               1 | active |
| 100.81.252.11 |    10234 |  4 | z1   |      10269 |               0 | active |
| 100.81.252.13 |    10238 |  6 | z3   |      10273 |               0 | active |
| 100.81.252.21 |    10235 |  2 | z2   |      10270 |               0 | active |
| 100.81.252.22 |    10236 |  5 | z2   |      10271 |               0 | active |
| 100.81.252.25 |    10237 |  3 | z3   |      10272 |               0 | active |
+---------------+----------+----+------+------------+-----------------+--------+
6 rows in set (0.01 sec)
</pre>

## 6.容灾case分析
### 问：当有obs宕机时，这台机器上的partition什么时候从partition表中删除：

RS会有一个ObLostReplicaChecker检查线程： 不断的迭代所有的partition（ObFullPartitionTableIterator）
如果确定是lost_replica的话，则会将其删除；调用pt_operator_->remove();

第一步：确定是 lost_server:

* 如果不在`server_manager`中的话，则认为是`lost_server`  
*  暂时忽略状态为deleting状态的OBS， 对于其他在`server_manager`中，最后一次心跳的时间距离现在已经超过
`GCONF.replica_safe_remove_time`的OBS，认为已经是下线的机器, 为`lost_server`。
(该配置项的默认值为：2个小时)

第二步：确定是lost_replica:  
需要先确定OBS，如果是已经`lost_server`的话，并且：如果满足`replica.modify_time_us_ + GCONF.replica_safe_remove_time` 小于当前时间的话：  
  
* 当前replica不在成员列表中的话，确定是lost_replica  
* 该replica对应的表已经不在schema里面了，确定是`lost_replica`
 
 

### 问：已经宕机的OBS，什么时候从rs的servermanager里面去掉？
用户主动发起delete-server: 

* 将机器上的unit资源迁走；
* 将`__all_server`表中的状态改成`OB_SERVER_ADMIN_DELETING`;

如果迁移完了，就会将该OBS从 `__all_server`表中删除


### 问： with_partition这个标志是怎么删除的
负载均衡线程里面，每轮都会去尝试：`try_notify_empty_server_checker`。这个函数里面会检查该OBS上的副本数，如果副本数为空的话，会启动一下`empty_server_checker`:
ObEmptyServerCheckRound 分为几个步骤：最后两个步骤是：

(1)  
(2)  
(3) EmtpyServer需要先wait_pt_sync_finish;  
(4)检查所有的partition的成员列表，只有某个server不在任何一个成员列表
中的时候，才会将他设置成`empty_server`. 修改`__all_server`表中的`with_partition`列，设置为true;
包括内存状态也会修改；  

这个标记最大的影响就是不能进行每日合并；

已经下线的obs怎么才能不在成员列表中呢？
 (1) 永久下线，并且触发复制，新上线的repilca，会发起成员变更，将下线的OBS去掉；
（2） rootserver的负载均衡会将永久下线的OBS从所有的成员列表中去掉；

对于`__all_server`表中的status域，临时下线和永久下线都是"inactive"， 必须要配合`gmt_modified`域一起来使用，判断是否是永久下线。


### 问：`server_permanent_offline_time` 是做什么用的？
一台机器下线后多久才认为是永久下线；  默认是一小时；
(1) RS在加载`__all_server`表的信息到内存中时，如果obs的状态不是active，则判断如果下线时间超过`server_permanent_offline_time`的话，则在内存中认为是永久下线的机器(`OB_HEARTBEAT_PERMANENT_OFFLINE`)，否则认为是`OB_HEARTBEAT_LEASE_EXPIRED`;
 
(2)RS会有一个检查心跳的线程（ObHeartbeatChecker），会定期的检查server的状态，间隔时间是100毫秒；
根据last_hb_time来设置obs的状态；三种有效状态：  
`OB_HEARTBEAT_ALIVE`,  
`OB_HEARTBEAT_LEASE_EXPIRED`,  
`OB_HEARTBEAT_PERMANENT_OFFLINE`,  
如果有状态变化的话，会往`rootservice_event`中写入变更信息；唤醒负载均衡和每日合并；    
如果机器正处于block_migrate的状态中的话，会判断如果时间已经超过阻塞时间`GCONF.migration_disable_time`的话，解锁`block_migrate`状态，并且唤醒负载均衡和每日合并；  
同时，只要有变更的话，都会生成一个异步任务（ObAllServerTask），去更新`__all_server`表；将最新的内存状态写入`__all_server`表中，如果是主RS，还会帮别人清理掉`with_rootservice`这一列的值；  
如果是下线的OBS会直接把负载均衡任务扔掉； 


###问：为什么看上去每日合并的调度序列不满足RS所在的集群最后一个合并

RS重启的时候，会加载`__all_zone`表中的`merge_list`;`merge_list`是本次合并调度的序列。每日合并线程开始工作的时候，需要生成本次合并的调度序列，主RS所在的zone最后一个进行合并。但是在代码实现的时候，只要zone的个数和名字不发生变化，就不会重新生成合并序列；因此，就算有新主上线，他也不会生成新的合并调度序列；（BUG！）  
在测试环境中，主是变来变去的，每个主都会援用就的合并序列，因此轮状合并就会出现主不过是最后一个被合并的；


###问： obs临时下线是否会出发迁移unit；何时出发迁移
临时下线的OBS不会发出迁移，需要等永久下线的时间到；

###问：负载均衡中soft_limit和hard_limit是怎么生效的？
在负载均衡选择unit的时候，必须要求：迁移unit以后，目的机器的机器资源小于`hard_limit`,相当于硬性资源限制；
对于若干满足`hard_limit`的OBS，如何选择呢？  
如果满足资源占比小于`soft_limit`，则选择`best_fit`算法，选择最合适的一个；  
如果资源占比不少于`soft_limit`, 则选择`least_load`算法，选择资源占比最少的一个；  
在unit均衡的时候，一般是将超过soft_limit的机器上的资源迁往少于`soft_limit`的机器；  
系统租户的最小内存占用为系统可用内存的25%， 最大为30%  
