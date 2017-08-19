# Obtest Case Design

## root major freeze

major freeze构造以下场景需要在代码里面增加多个debug sync point

### 正常情况

* 一个租户：连续发起5次major freeze
* 五个租户：连续发起5次major freeze

### prepare失败，abort成功，然后全部重做后成功

* 系统租户
* 普通租户

### prepare成功，commit失败，经过重试后成功

* 系统租户all_core_table切主, 老rs退出，新rs继续做
* 系统租户all_core_table commit超时, 重试commit
* 系统租户all_core_table commit成功，其他partition commit失败，重试commit其他partition

### prepare失败, abort失败，重试abort成功，然后全部重做成功

### major freeze过程中rs发生切换，新的rs继续完成未完成的major freeze

* 发起major freeze更新all_zone表成功，更新all_global_stat失败，然后rs发生切换
* 系统租户prepare过程中rs切换, partition table相关partition未prepare
* 系统租户prepare后rs切换，partition table相关partition已经prepare(跟上面的重复）
* 系统租户commit过程中rs切换，all_core_table已经commit
* 系统租户完成后普通租户freeze过程中rs切换
* 所有租户都freeze完成，在更新all_zone表前rs切换
* 所有租户都freeze完成，更新完all_zone表后更新all_global_stat失败

### 两个rs同时做major freeze

## unit manager

### basic

* 创建unit_config, resource_pool, tenant
* unit_config被多个resource pool引用，resource_pool被多个tenant引用(报错）
* 删除unit_config(包含被引用时）, resource_pool（包含被引用时), tenant

### 创建resource pool

* 没超过resource_soft_limit
* 超过resource_soft_limit，没超过resource_hard_limit
* 超过resource_hard_limit，报错
* 改大resource_hard_limit，成功
* 创建unit_num为2的resource_pool, 分布到不同的observer
* 创建unit_num超过server数量的resource_pool,报错

### 更改unit_config, 更改resource_pool

* 改大cpu到server能hold的值，成功; 改大cpu到server不能hold，报错; 改小cpu，成功
* 改大max_memory到server能hold的值，成功; 改大max_memory到server不能hold，报错;
  改小min_memory到比已用的memory还小的值，报错; 改小min_memory使得至少剩余10%可用，成功
* 同时更改cpu和memory
* 修改resource pool unit_num, unit_config, zone_list

### unit 容灾

* server临时下线
* server由临时下线变为上线
* server由临时下线变为永久下线
* unit已经有临时unit，另一个zone的unit的server临时下线，直接迁移
* server永久下线触发了迁移，迁移过程中目的server临时下线
* server临时下线，临时unit所在的server被block
* server永久下线触发了迁移，迁移过程中目的server被block
* unit正在迁移，迁移过程中目的server被block

### unit 负载均衡

* 手动迁移unit导致unit负载不均衡
* 增加一台新的server
* 删除一台server
* 增加一些容灾操作观察unit分布是否均衡
