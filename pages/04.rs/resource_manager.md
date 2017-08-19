本文档是一些思考备忘，非正式设计文档。....yzf....Fri, 28 Oct 2016....10:37....

# 资源管理器
RS的一个主要职能是作为多租户分布式数据库的物理和逻辑资源管理。

## 资源

1. observer物理资源
2. unit虚拟机资源
3. zone可用性域（机房）
4. region地理区域
5. docker容器

## 资源使用指标
资源负载均衡的指标，边做边思考，我的一些想法和大家探讨一下。资源利用率resource usage rate，是一个宽泛的说法，对于每种资源，他的含义不同。但是，一般地，计算它需要获得两个量：使用量，和容量。

  rate = ussage / capacity(or limit)
  
目前考虑以如下六个独立的资源使用率指标作为资源负载均衡的指标因子。我们需要在partition级别准确地测量，在unit级别精确地限制这六个指标。

### 1. CPU
CPU的使用是系统内共享的，没有占用量指标，使用cpu利用率来衡量。

#### 1.1 使用率
使用每秒cpu时间来衡量：
<pre>
  cpu_usage_rate = cpu_utime_rate + cpu_stime_rate （1）
</pre>

其中：
1. cpu_utime_rate为每秒内用户态cpu耗时
2. cpu_stime_rate为每秒内内核态cpu耗时

CPU的容量为核数，比如8核，则容量为8.

### 2. DISK
磁盘资源有占用量和IOPS两个角度。

#### 2.1 占用量
<pre>
   disk_usage_rate = disk_bytes / disk_limit （2）
</pre>

#### 2.2 IOPS
<pre>
   iops_usage_rate = sstable_read_rate + sstable_write_rate + log_write_rate （3）
</pre>

TODO：需要测得observer IOPS并汇报给RS的基础设施。

### 3. MEMORY

#### 3.1 占用量
<pre>
   memory_usage_rate = memtable_bytes / memory_limit （4）
</pre>

#### 3.2 吞吐量
内存的吞吐量忽略不考虑，目前想不到精确度量和限制的基础设施。

### 4. NET
网络考虑吞吐量和RPC处理能力两个角度。

#### 4.1 IOPS
因为RPC框架的packet per seconds是有上限的。
<pre>
   net_packet_rate = net_in_rate + net_out_rate （5）
</pre>

TODO: 需要测得OB RPC框架packet per second容量。

#### 4.2 吞吐量
考虑网卡吞吐率。
<pre>
   net_throughput_usage = net_in_bytes_rate + net_out_bytes_rate （6）
</pre>

容量为网卡带宽。
