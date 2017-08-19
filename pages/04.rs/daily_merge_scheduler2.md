# RootServer Daily Merge Schedule 【2】

## 背景

OceanBase 1.0数据为分动态数据有基线数据，动态数据会定期合并到基线数据。在OceanBase 1.0之前，
我们一般一天只做一次这样的合并，这个过程叫做每日合并(Daily Merge)。每日合并，由统一冻结后由RS调度
OB Server执行合并，本文描述RS调度每日合并过程。

## 名词解释

* 大版本冻结 (major freeze): 所有OB Server冻结memetable，新开新版本memtable用于写入
* 每日合并 (daily merge): 一般一天只发生一次的基线数据与动态数据合并过程
* 错峰合并 (stagger merge): RS调度每日合并策略，按集群做合并，错开合并压力，保证在合并期间有集群能提供正常服务


## 设计目标

调度各cluster执行错峰合并。

## 设计思路及折衷

Daily Merge Schedule在RS内部为一单独线程，称之为：daily\_merge\_scheduler\_; 轮询当前系统状态。如果发生大版本冻结则发起每日合并流程。
每日合并一般执行错峰合并，但也可所有集群一起合并，由enable_stagger_merge配置项控制。

cluster合并前流量切换与0.5不大一样，不是由client端主动切换而是通过leader改选，将leader切到其它集群。
由于我们只访问partition的leader，leader切换后流量自然发生切换。
同时切流量时，也不考虑渐近的方法，而是一次性全部切换。因为对于一个partition的流量只能全切，
做渐近意义不大。

跟0.5一样，对于RS重启功换每日合并流程不能中断，需继续上次执行，所以每日合并相关状态记在内部表
 \_\_all\_cluster中。RS启动时从内部表恢复状态继续执行。
 
## 代码详解

### 1. ObDailyMergeScheduler
  继承自ObReentrantThread。一个可以反复start/stop的线程。 不同于之前的ObRunnableThread，只能被start、stop一次就会线程退出。daily\_merge的线程会随着RS状态的切换而被反复调用start或者stop;RS切换成主时该线程会start。否则会被stop掉，线程不退出，只是出于等待状态下。

### 2. daily\_merge线程的调度
* init 初始化  
 每个ObServer启动的时候，都会初始化rs\_service, 在rs_service里面会将daily\_merge线程初始化；
* destory 退出   
 ObServer退出的时候，会调用root\_service的destory. 如果obs是主RS， 那么需要先将daily\_merge的线程stop/wait。然后在destory退出；
* start 线程开始工作  
  启动以后，或者切换成主以后，root\_service开始工作的时候，也会start daily\_merge线程；
* wakeup 唤醒
    * 收到mergefreeze冻结命令之后；
    * 收到mergefinish合并完成的命令之后；
		
### 3. 相关内容

(1)渐进合并的两个配置项： GCONF.enable\_merge\_by\_turn  GCONF.zone\_merge\_concurrency
 
(2)ObThreadIdling:线程的等待唤醒机制。  
   合并线程会工作一会休息10分钟，再工作再休息；如果在工作工作中，有人尝试唤醒，这是时候合并线程会记录下来本次唤醒，等本轮工作完成以后，尝试进行wait状态时，发现有人曾经wakeup过，所以就会跳过wait阶段，重新开始工作。

(3）\_\_all\_meta\_table/\_\_all\_root\_table 中的两列：   

* is\_original\_leader:在错峰合并的时候，如果当前partition为主，那么会将这个标记设置成true，等待合并完成以后，在将该partition重新切成主；  

* is\_previous\_leader:该列已经不再生效。用来在系统出现无主时，查看谁原来是主，以及这个partition的member list有哪些；但是这个方法有问题，已经修改成用一个选成主的时间戳来判断谁是最后一个主，这样也会简化一些RPC请求，比如，在切主的时候，新的主需要扫描该partition的所有副本，设置其他副本的role为follow,然后在设置自己的role为主。改成时间戳以后，就只需要设置自己的时间戳，就能查出谁是最后的主；


### 4. 流程

1. schedule_daily_merge流程

```

          thread start
            |
            v
  +-----> wait <-------------------------------------------------+
  |         |                                                    |
  |         v                                                    |
  |    +---update_local_index_status                             |
  |   | No   | YES                                               |
  |   |      v                                                   |
  |   |   merge_status= MERGE_STATUS_IDLE ==>reset:TIMEOUT/ERROR |
  |   |      |                                                   |
  |   +----- v                                                   |
  |          |                                                   |
  |          |                                                   |
  |          v                                                   |
  | +---- all cluster merged（majority）                          |
  | |       |                                                    |
  | | NO    | YES                                                | NO
  | |       v                                                    |
  | |     global_broadcast_version < frozen_version ------------+
  | |       |
  | |       | YES
  | |       v
  | |      backup_leader_pos
  | |       |
  | |       v
  | |     global_broadcast_version += 1 && MERGE_STATUS_MERGING
  | |       |
  | |       v
  | +---> arrange cluster merge order(主RS最后)
  |         |
  |         v
  | +---> chose next cluster <----------------------------------------------+
  | |       |                                                               |
  | |       v                                                               |
  | |     cluster.broadcast_version == global_broadcast_version             |
  | |       |                            |                                  |
  | |       | YES                        | NO                               |
  | |       |                            v                                  |
  | |       |         +-------------stagger_merge -----+                    |
  | |       |         | YES                            |  NO                |
  | |       |         v                                |                    |
  | |       |        warm_up          +----------------|                    |
  | |       |         |               |                                     |
  | |       |         v               v                                     |
  | |       |       cluster.is_merging = 1                                  |
  | |       |       cluster.merge_stat = MERGING                            |
  | |       |                          |                                    |
  | |       |                          v                                    |
  | |       |                    ObLeaderCoordinator::coordinate()          |
  | |       |                            |                                  |
  | |       |                            v                                  |
  | |       |                       start_zone_merge                        |
  | |       +-------- cluster.broadcast_version = global_broadcast_version  |
  | |       |             && merge_start_time = now()                       |
  | |       |             && merge_stat = MERGE_STATUS_MERGING              |
  | |       |                                                               |
  | |       v                                                               |
  | |     wait <----------------------+                                     |
  | |       |                         |                                     |
  | |       |                         | NO                                  |
  | |       v                         |                                     |
  | |     cluster merged(majority) ------> check timeut -----> cluster.merge_stat = MERGE_TIMEOUT
  | |       |
  | |       | YES
  | |       v
  | |     is_merging = 0
  | |     cluster.merge_stat = MERGE_STATUS_IDLE/MERGE_STATUS_MOST_MERGED
  | |     last_merged_time = now()
  | |     cluster.last_merged_version = cluster.broadcast_version
  | |       |
  | | NO    |
  | |       v
  | +---- all cluster merged （majority）
  |         |
  |         | YES
  |         v
  +-------global_merge_cluster = global.broadcast_version && merge_status = MERGE_STATUS_INDEXING


```
       		
## 相关类

```  
* ObIZoneOp  
	封装了多个__all_zone需要进行的操作，按照操作的内容分为三个子类：  
	 * ObSetZoneMergingOp：设置某个zone的状态为合并中；
	 * ObSetFinishZoneMergeOp：设置某个zone合并完成；
	 * ObSetZoneMergeTimeOutOp：设置某个zone合并超时
 利用ObZoneItemTransUpdater封装类来修改__all_zone表；修改对应的项目；
 
 * ObZoneInfo
 	保存每个zone的状态信息，也是__all_zone表中信息在内存中的表示；
 	enum MergeStatus
    {
    MERGE_STATUS_IDLE,
    MERGE_STATUS_MERGING,
    MERGE_STATUS_TIMEOUT,
    MERGE_STATUS_ERROR,
    MERGE_STATUS_INDEXING,
    MERGE_STATUS_MOST_MERGED,
    MERGE_STATUS_MAX
    };
 	
 * ObZoneInfoItem 
 	ObZoneInfo的组成模块，ObZoneInfoItem有一个update接口，可以传入ObZoneItemTransUpdater，来操作__all_zone表；
  
```  


##相关问题

* 为什么要有add op和直接调用两个接口，应该可以统一;  可以统一。
* warm\_up\_start\_time只有全局值，预热的过程，好像没有分zone进行。而是整个集群一起开搞，是不是增加了负担；  
* 为什么全局状态中没有all\_merged\_version;  暂时没有需求。
* 为什么要搞一个majority merged的概念；包括两个概念：（1）配置项，（2）majority。 防止某些特殊的zone合并失败而拖累整个集群，导致合并不能推进。