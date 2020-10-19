# mxs-exchange

#### 介绍
mxs-exchange 是10万级内存交易撮合系统。基于[OpenHFT Chronicle-Bytes](https://github.com/OpenHFT/Chronicle-Bytes)，[OpenHFT Chronicle-Wire](https://github.com/OpenHFT/Chronicle-Wire)，[lz4 lz4-java](https://github.com/lz4/lz4-java),RocketMq

#### 软件架构
不同币队分发撮合引擎    
订单匹配引擎    
快速快照模块    
交易，管理和报告API

#### 安装教程

1.  本地安装rocketmq 并自动创建topic,或者创建topic:mxs-local-request-command
2.  运行exchange-match
3.  运行exchange-order

#### 使用说明

1.  通过exchange-match下单及其操作去：http://127.0.0.1:8410/order/match（是ws实现的，只能测试）
2.  同过exchange-order下单，走rocketmq 接口：http://127.0.0.1:8300/order/add    
    参数模型：{"sysUid":1,"sysSiteId":1,"price":7,"num":1,"amount":7,"ifBid":false,"orderType":1,"symbolId":1,"sysUuid":1}
3.  在实际的生产环境，exchange-match的日志级别改为Error,默认是Debug会显示撮合流程。

## 调优

#### jvm选项

推荐使用最新发布的JDK 1.8版本。通过设置相同的Xms和Xmx值来防止JVM调整堆大小以获得更好的性能。简单的JVM配置如下所示：​

```
-server -Xms8g -Xmx8g -Xmn4g
```

如果您不关心exchange-match的启动时间，还有一种更好的选择，就是通过“预触摸”Java堆以确保在JVM初始化期间每个页面都将被分配。那些不关心启动时间的人可以启用它：​ -XX:+AlwaysPreTouch
禁用偏置锁定可能会减少JVM暂停，​ -XX:-UseBiasedLocking
至于垃圾回收，建议使用带JDK 1.8的G1收集器。

```
-XX:+UseG1GC -XX:G1HeapRegionSize=16m   
-XX:G1ReservePercent=25 
-XX:InitiatingHeapOccupancyPercent=30
```
这些GC选项看起来有点激进，但事实证明它在我们的生产环境中具有良好的性能。另外不要把-XX:MaxGCPauseMillis的值设置太小，否则JVM将使用一个小的年轻代来实现这个目标，这将导致非常频繁的minor GC，所以建议使用rolling GC日志文件：

```
-XX:+UseGCLogFileRotation   
-XX:NumberOfGCLogFiles=5 
-XX:GCLogFileSize=30m
```
如果写入GC文件会增加代理的延迟，可以考虑将GC日志文件重定向到内存文件系统：

```
-Xloggc:/dev/shm/mq_gc_%mxs.log
```
####  Linux内核参数
- 这里是列表文本vm.extra_free_kbytes，告诉VM在后台回收（kswapd）启动的阈值与直接回收（通过分配进程）的阈值之间保留额外的可用内存。
- vm.min_free_kbytes，如果将其设置为低于1024KB，将会巧妙的将系统破坏，并且系统在高负载下容易出现死锁。
- vm.max_map_count，限制一个进程可能具有的最大内存映射区域数。RocketMQ将使用mmap加载CommitLog和ConsumeQueue，因此建议将为此参数设置较大的值。（agressiveness --> aggressiveness）
- vm.swappiness，定义内核交换内存页面的积极程度。较高的值会增加攻击性，较低的值会减少交换量。建议将值设置为10来避免交换延迟。
- File descriptor limits，RocketMQ需要为文件（CommitLog和ConsumeQueue）和网络连接打开文件描述符。我们建议设置文件描述符的值为655350。
- Disk scheduler，RocketMQ建议使用I/O截止时间调度器，它试图为请求提供有保证的延迟

#### PS
exchange-match 是一个计算密集型，也是一个IO密集型的服务。每次撮合都是同过撮合算法进行大量的数据匹配与计算，并把撮合产生的数据发送出去。因此对cpu与内存要求相对较高。


#### 服务关闭流程
撮合服务并没有实时的数据落盘，是基于命令实现不定时的罗盘机制，并加以复用。    
1. 停止下单
2. 发生存快照命令
3. 停止服务：url -X POST 127.0.0.1:8762/shutdown（尽量不要直接kill,避免计算和落盘未完成）

但是在实际的运行中，不可避免遇到不可预测和意外发生。内存丢失，未存最后时刻快照，解决方案如下:
1. 直接从数据库中恢复，mq从最新消息点位消费。
2. 从正常的服务器存快照，copy到当前用户目录下snapshoot中，mq消费点位重置到存快照前4小时内任意一刻（建议重置到20-30分钟内）。
3. 从历史快照的前四小时内开始重置mq的消费点位。

## 命令快照切换快照
1. 实时订单状态存储可以在com.github.kinbug.match.core.helper.MatchHelper.java对接
2. 不定时快照存储切换在com.github.kinbug.match.core.snapshoot包中实实现
