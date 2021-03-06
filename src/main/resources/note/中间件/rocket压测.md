一次RocketMQ小规模集群的压测
（备注：以下压测过程以及压测结果，都是根据我们之前真实的RocketMQ压测报告总结而来，非常的有代表性，大家完全可以结合我
们之前说的机器配置来参考一下）

（1）RocketMQ的TPS和消息延时

我们让两个Producer不停的往RocketMQ集群发送消息，每个Producer所在机器启动了80个线程，相当于每台机器有80个线程并发的
往RocketMQ集群写入消息。
而
RocketMQ集群是1主2从组成的一个dledger模式的高可用集群，只有一个Master Broker会接收消息的写入。

然后有2个Cosumer不停的从RocketMQ集群消费数据。

每条数据的大小是500个字节，这个非常关键，大家一定要牢记这个数字，因为这个数字是跟后续的网卡流量有关的。

我们发现，一条消息从Producer生产出来到经过RocketMQ的Broker存储下来，再到被Consumer消费，基本上这个时间跨度不会超
过1秒钟，这些这个性能是正常而且可以接受的。

同时在RocketMQ的管理工作台中可以看到，Master Broker的TPS（也就是每秒处理消息的数量），可以稳定的达到7万左右，也就是
每秒可以稳定处理7万消息。

（2）cpu负载情况

其次我们检查了一下Broker机器上的CPU负载，可以通过top、uptime等命令来查看

比如执行top命令就可以看到cpu load和cpu使用率，这就代表了cpu的负载情况。

在你执行了top命令之后，往往可以看到如下一行信息：

load average：12.03，12.05，12.08

类似上面那行信息代表的是cpu在1分钟、5分钟和15分钟内的cpu负载情况

比如我们一台机器是24核的，那么上面的12意思就是有12个核在使用中。换言之就是还有12个核其实还没使用，cpu还是有很大余力
的。

这个cpu负载其实是比较好的，因为并没有让cpu负载达到极限。

（3）内存使用率

使用free命令就可以查看到内存的使用率，根据当时的测试结果，机器上48G的内存，仅仅使用了一部分，还剩下很大一部分内存都是
空闲可用的，或者是被RocketMQ用来进行磁盘数据缓存了。

所以内存负载是很低的。

（4）JVM GC频率

使用jstat命令就可以查看RocketMQ的JVM的GC频率，基本上新生代每隔几十秒会垃圾回收一次，每次回收过后存活的对象很少，几
乎不进入老年代

因此测试过程中，Full GC几乎一次都没有。

（5）磁盘IO负载

接着可以检查一下磁盘IO的负载情况。

首先可以用top命令查看一下IO等待占用CPU时间的百分比，你执行top命令之后，会看到一行类似下面的东西：

Cpu(s): 0.3% us, 0.3% sy, 0.0% ni, 76.7% id, 13.2% wa, 0.0% hi, 0.0% si。


在这里的13.2% wa，说的就是磁盘IO等待在CPU执行时间中的百分比

如果这个比例太高，说明CPU执行的时候大部分时间都在等待执行IO，也就说明IO负载很高，导致大量的IO等待。

这个当时我们压测的时候，是在40%左右，说明IO等待时间占用CPU执行时间的比例在40%左右，这是相对高一些，但还是可以接受
的，只不过如果继续让这个比例提高上去，就很不靠谱了，因为说明磁盘IO负载可能过高了。

（6）网卡流量

使用如下命令可以查看服务器的网卡流量：

sar -n DEV 1 2

通过这个命令就可以看到每秒钟网卡读写数据量了。当时我们的服务器使用的是千兆网卡，千兆网卡的理论上限是每秒传输128M数
据，但是一般实际最大值是每秒传输100M数据。

因此当时我们发现的一个问题就是，在RocketMQ处理到每秒7万消息的时候，每条消息500字节左右的大小的情况下，每秒网卡传输
数据量已经达到100M了，就是已经达到了网卡的一个极限值了。

因为一个Master Broker服务器，每秒不光是通过网络接收你写入的数据，还要把数据同步给两个Slave Broker，还有别的一些网络通
信开销。

因此实际压测发现，每条消息500字节，每秒7万消息的时候，服务器的网卡就几乎打满了，无法承载更多的消息了。