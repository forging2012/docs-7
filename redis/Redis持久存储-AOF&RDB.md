**Redis**中数据存储模式有2种：cache-only,persistence;cache-only即只做为“缓存”服务，不持久数据，数据在服务终止后将消失，此模式下也将不存在“数据恢复”的手段，是一种安全性低/效率高/容易扩展的方式；persistence即为缓存中的数据持久备份到磁盘文件，在服务重启后可以恢复，此模式下数据相对安全。

 

​    持久化数据的方式很多，基于各种考虑面，可能最终导致的设计手段有所差异。针对互联网应用，服务提供者必须具备并发访问/数据安全/故障修复/集群与容错等，我先从如下几个方面引导一下：

1. 单server情况下，一条数据被修改后，是立即持久化？还是按照时间戳“批量 + 增量”持久化？还是全量持久化？
2. 对于密集的变更操作，持久化的性能开支是否会影响到服务输出能力？
3. 数据如何恢复？
4. 集群情况下，如何确保每个节点上持久化数据一致性？

​    Redis中采取了AOF/snapshot/vm三种手段来直接或者间接的持久化数据，其中VM策略已经被弃用。如下为设计思想：

 

**一.AOF**

   **1) AOF：**Append-only file，将“操作 + 数据”以格式化指令的方式追加到操作日志文件的尾部，在append操作返回后(已经写入到文件或者即将写入)，才进行实际的数据变更，“日志文件”保存了历史所有的操作过程；当server需要数据恢复时，可以直接replay此日志文件，即可还原所有的操作过程。AOF相对可靠，它和mysql中bin.log、apache.log、zookeeper中txn-log简直异曲同工。AOF文件内容是字符串，非常容易阅读和解析，它和redis-protocol具有一样的格式约束：

 

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. \##AOF文件格式，其中"##"为注释，非文件实际内容  
2. \##选择DB  
3. *2  
4. $6  
5. SELECT  
6. $1  
7. 0  
8. \##SET k-v  
9. *3  
10. $3  
11. SET  
12. $2  
13. k1  
14. $2  
15. v1  

   我们可以简单的认为AOF就是日志文件，此文件只会记录“变更操作”(例如：set/del等)，如果server中持续的大量变更操作，将会导致AOF文件非常的庞大，意味着server失效后，数据恢复的过程将会很长；事实上，一条数据经过多次变更，将会产生多条AOF记录，其实只要保存当前的状态，历史的操作记录是可以抛弃的；因为AOF持久化模式还伴生了“AOF rewrite”。

​    AOF的特性决定了它相对比较安全，如果你期望数据更少的丢失，那么可以采用AOF模式。如果AOF文件正在被写入时突然server失效，有可能导致文件的最后一次记录是不完整，你可以通过手工或者程序的方式去检测并修正不完整的记录，以便通过aof文件恢复能够正常；同时需要提醒，如果你的redis持久化手段中有aof，那么在server故障失效后再次启动前，需要检测aof文件的完整性。aof文件默认位于src下appendonly.aof:

 

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. $ ./redis-check-aof --fix appendonly.aof  
2. AOF analyzed: size=52, ok_up_to=52, diff=0  
3. AOF is valid  

   

   **2) 配置：**

 

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. \##此选项为aof功能的开关，默认为“no”，可以通过“yes”来开启aof功能  
2. \##只有在“yes”下，aof重写/文件同步等特性才会生效  
3. appendonly no  
4.   
5. \##指定aof文件名称  
6. appendfilename appendonly.aof  
7.   
8. \##指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
9. appendfsync everysec  
10. \##在aof-rewrite期间，appendfsync是否暂缓文件同步，"no"表示“不暂缓”，“yes”表示“暂缓”，默认为“no”  
11. no-appendfsync-on-rewrite no  
12.   
13. \##aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认“64mb”，建议“512mb”  
14. auto-aof-rewrite-min-size 64mb  
15.   
16. \##相对于“上一次”rewrite，本次rewrite触发时aof文件应该增长的百分比。  
17. \##每一次rewrite之后，redis都会记录下此时“新aof”文件的大小(例如A)，那么当aof文件增长到A*(1 + p)之后  
18. \##触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
19. auto-aof-rewrite-percentage 100  

   

​    AOF是文件操作，对于变更操作比较密集的server，那么必将造成磁盘IO的负荷加重；此外linux对文件操作采取了“延迟写入”手段，即并非每次write操作都会触发实际磁盘操作，而是进入了buffer中，当buffer数据达到阀值时触发实际写入(也有其他时机)，这是linux对文件系统的优化，但是这却有可能带来隐患，如果buffer没有刷新到磁盘，此时物理机器失效(比如断电)，那么有可能导致最后一条或者多条aof记录的丢失。通过上述配置文件，可以得知redis提供了3中aof记录同步选项：

- always：每一条aof记录都立即同步到文件，这是最安全的方式，也以为更多的磁盘操作和阻塞延迟，是IO开支较大。
- everysec：每秒同步一次，性能和安全都比较中庸的方式，也是redis推荐的方式。如果遇到物理服务器故障，有可能导致最近一秒内aof记录丢失(可能为部分丢失)。
- no：redis并不直接调用文件同步，而是交给操作系统来处理，操作系统可以根据buffer填充情况/通道空闲时间等择机触发同步；这是一种普通的文件操作方式。性能较好，在物理服务器故障时，数据丢失量会因OS配置有关。

​    其实，我们可以选择的太少，everysec是最佳的选择。如果你非常在意每个数据都极其可靠，建议你选择一款“关系性数据库”吧。  
​    AOF文件会不断增大，它的大小直接影响“故障恢复”的时间,而且AOF文件中历史操作是可以丢弃的，AOF rewrite操作就是“压缩”AOF文件的过程，当然redis并没有采用“基于原aof文件”来重写的方式，而是采取了类似snapshot的方式：基于copy-on-write，全量遍历内存中数据，然后逐个序列到aof文件中。因此AOF rewrite能够正确反应当前内存数据的状态，这正是我们所需要的；rewrite过程中，对于新的变更操作将仍然被写入到原AOF文件中，同时这些新的变更操作也会被redis收集起来(buffer，copy-on-write方式下，最极端的可能是所有的key都在此期间被修改，将会耗费2倍内存)，当内存数据被全部写入到新的aof文件之后，收集的新的变更操作也将会一并追加到新的aof文件中，此后将会重命名新的aof文件为appendonly.aof,此后所有的操作都将被写入新的aof文件。如果在rewrite过程中，出现故障，将不会影响原AOF文件的正常工作，只有当rewrite完成之后才会切换文件，因为rewrite过程是比较可靠的。
​    触发rewrite的时机可以通过配置文件来声明，同时redis中可以通过bgrewriteaof指令人工干预。因为rewrite操作/aof记录同步/snapshot都消耗磁盘IO，redis采取了“schedule”策略：无论是“人工干预”还是系统触发，snapshot和rewrite需要逐个被执行。
​    AOF rewrite过程并不阻塞客户端请求。

二.Snapshot

​    Snapshot，快照，这种思想被广泛的应用在多个技术领域，主要目的就是对当前数据的状态进行保存到文件中，有可能触发实际数据的copy过程，在整个时间线（time-line）上，同一个数据会有多个snapshot时间点（snapshot point），每个时间点都会持有相应的数据状态，可以根据需要，从各个snapshot时间点上恢复(重现)当时的数据，就像“照相”一样，每张照片都是某刻的风景的剪影；snapshot的变种以及实现手段很多，比如cop-on-write，fully-copy，内存级别的fuzzy-snapshot等等。 其中copy-on-write和fuzzy-snapshot在基于内存数据的snapshot中广泛使用.

​    1) copy-on-write是基于内存数据“指针”复制实现，在snapshot时，首先把“数据”copy一份(指针引用copy，指向同一个实际数据)，然后将数据序列化到文件中，如果序列化过程中数据变更，那么将导致实际数据的copy。   

 

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. A ---> data  
2. SnapshotA ---> data  
3. 变更时：  
4. data1 == data (实际数据copy)  
5. A ---> data1  
6. SnapshotA ---> data  
7. 结束后:  
8. data = null  

​    2) fuzzy-snapshot:和上述不同的时，此方式不需要引用copy，在snapshot时如果有变更操作，也不会copy数据。比如T1时刻开始snapshot，T2时刻对数据进行了变更，T3时刻snapshot结束，那么snapshot结束时T2时刻的新数据也会被记录在文件中；由此可见这种snapshot并没有正确反应T1时刻的数据状态，而是T1~T3区间(非严格)，就像拍照时，人物的移动导致照片“模糊”一样。

 

​    废话半天，redis中snapshot最终将内存数据序列化到dump.rdb文件中(参考：[redis存储与数据结构](http://shift-alt-ctrl.iteye.com/blog/1874693))，此文件的作用仍然是“故障恢复”。Redis采用了copy-on-write的思路，将内存中数据(K-V,expire,数据结构类型等)以结构化的方式序列化到rdb文件中。snapshot和AOF只不过在一些策略上不同，其实工作原理非常类似。snapshot的主要目的，就是“数据备份”，比如每隔12小时备份一次数据，并把snapshot生成的rdb文件保存起来，我们可以根据rdb文件将数据恢复到任意时间点上。此外rdb文件也是作为master-slave中数据同步的一种手段，一个slave加入集群，那么master(也可以是其他“领导者”，--slaveof指定)将会立即执行一次“例外”的snapshot，并把snapshot期间的新变更操作收集起来(buffer)，snapshot结束后，将rdb文件直接转发给slave，同时把收集的新变更操作依次同步给slave。(SYNC指令)

   

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. \##snapshot触发的时机，save <seconds> <changes>  
2. \##如下为900秒后，至少有一个变更操作，才会snapshot  
3. \##对于此值的设置，需要谨慎，评估系统的变更操作密集程度  
4. \##可以通过“save “””来关闭snapshot功能  
5. save 900 1  
6. \##当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等  
7. stop-writes-on-bgsave-error yes  
8. \##是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间  
9. rdbcompression yes  
10. \##rdb文件名称  
11. dbfilename dump.rdb  
12. \##rdb/AOF文件存储的目录  
13. dir ./  

   save 900 1       （15分钟变更一次）

   save 300 10     （5分钟变更10次）

   save 60 10000  （1分钟变更1万次）

   这个默认的时间&次数设置还是比较科学的。

​    snapshot触发的时机，是有“间隔时间”和“变更次数”共同决定，同时符合2个条件才会触发snapshot,否则“变更次数”会被继续累加到下一个“间隔时间”上。snapshot过程中并不阻塞客户端请求，copy-on-write方式也意味着极端情况下可能会导致2倍内存的使用量，同时snapshot和aof rewrite一样文件操作也是很快速的。snapshot首先将数据写入临时文件，当成功结束后，将临时文件重名为dump.rdb（参考配置）

​    snapshot和aof的方式相比，它没有“everysec”文件同步，即在对磁盘文件的操作只会在snapshot时发生，而不是像AOF那样“每次变更”都会触发“文件同步”，从这方面说，snapshot是有优势的。此外snapshot的文件尺寸要比aof小，而且更加结构化，非常便于解析引擎快速解析和内存实施（比AOF要快）。但是因为snapshot的触发实际和“间隔时间”有密切关系，因此server失效后会导致上一个时间点之后的数据未能序列化到rdb文件，这和aof相比，安全性上稍弱。

​    可以使用**bgsave**指令手动触发snapshot，即使配置文件中关闭了snapshot(save "")，此指令仍然能够正确生成dump.rdb文件，你可以根据需要将此rdb文件转存到其他地方。

​    SAVE指令会阻塞Redis服务进程，直到RDB文件创建完毕，在阻塞期间，服务器将不能处理任何请求。BGSAVE命令会派生一个子进程，然后又子进程负责创建RDB文件，服务器进程（父进程）可以继续处理客户端请求。

 

**三.总结：**

​    AOF和snapshot各有优缺点，这是有它们各自的特点所决定：

​    1) AOF更加安全，可以将数据更加及时的同步到文件中，但是AOF需要较多的磁盘IO开支，AOF文件尺寸较大，文件内容恢复数度相对较慢。

​    2) snapshot，安全性较差，它是“正常时期”数据备份以及master-slave数据同步的最佳手段，文件尺寸较小，恢复数度较快。

​    可以通过配置文件来指定它们中的一种，或者同时使用它们(不建议同时使用)，或者全部禁用，在架构良好的环境中，master通常使用AOF，slave使用snapshot，主要原因是master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务(目前slave只能提供读取服务)，它的主要目的就是快速响应客户端read请求；但是如果你的redis运行在网络稳定性差/物理环境糟糕情况下，建议你master和slave均采取AOF，这个在master和slave角色切换时，可以减少“人工数据备份”/“人工引导数据恢复”的时间成本；如果你的环境一切非常良好，且服务需要接收密集性的write操作，那么建议master采取snapshot，而slave采用AOF。

​    1) 如果master失效，那么slave可以人工的方式“提升”为master：

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. redis 127.0.0.1:6379> config set appendonly yes  
2. OK  
3. redis 127.0.0.1:6379> BGREWRITEAOF  
4. Background append only file rewriting started  
5. redis 127.0.0.1:6379> SLAVEOF no one  
6. OK  

​    当然可以通过slaveof指令为当前server指定master。提升master的过程，首先开启slave的AOF功能，然后立即执行bgrewriteaof，然后将此slave提升为master。

​    如果此slave与master之间数据存在很大差异，那么需要把master中的aof文件copy一份到slave中，并重启slave：首先开启slave的appendonly，不执行bgrewriteaof，然后提升master，停止slave，将master的aof文件copy到slave中，然后重启slave。

​    2) 如果slave失效，这个非常好办，直接重启就行，slave重启后，会首先向master发送“SYNC”指令，那么master将会立即生成一份snapshot文件，并传输给slave，slave根据此snapshot文件恢复内存数据。

​    3) 在redis中master和slave的角色切换非常简单，可以通过指令将slaveof提升为master，反之亦然；事实上slave也可以接受write操作，只不过这些write操作，在和master进行“SYNC”之后，将会消失；建议将slave作为read-only模式。

​    4) 在redis中AOF和snapshot可以同时使用，但是在数据恢复时AOF会被优先采用，因为AOF比snapshot中数据更加完整。

 

​    在数据恢复时，如果开启了AOF，那么服务器端将优先使用AOF文件恢复数据，否则才会使用snapshot的RDB文件来恢复数据。

 

四、M-S模式下数据复制

​    M-S模式下，数据replicated机制也不复杂，不过在redis2.8前后版本稍有不同。旧版本：

​    1）slave向master发送“SYNC”指令

​    2）master使用BGSAVE，在后台为此slave生成一个RDB文件，同时使用一个专用的buffer来保存此期间新的变更操作。（BGSAVE基于copy-on-write）

​    3）master开启子进程，将RDB文件通过网络发送给slave，并将buffer中的新变更记录随后补充发送给slave（直到slave数据全部重放完毕后销毁）。

​    4）此后，slave接收RDB文件，并在接收时对数据进行内存重放。当数据恢复完毕后，即与master建立同步复制机制。

 

​    这种机制，我们称为“全量同步”，即使slave只是简单的网络闪断，它与master之间的数据同步，也是全量的，由此可见，性能稍差。

 

​    在redis 2.8之后，redis增加了“增量同步”机制，有点类似于mysql的binlog同步机制（GTID），使用“PSYNC”指令；这个指令兼顾“全量复制”、“增量复制”，即如果slave落后太多，则执行全量复制，如果只是短暂离开，则使用增量复制。这也要求redis在数据存储、同步时，需要有一个offset来标记数据游标的位置、backlog缓冲区、服务器运行ID。

​    1）master每次向某个slave同步N个字节的数据之后，将与此slave的复制offset值加N。此offset用来标记下一次replication的起始位置。（如果master有多个slave，每个slave均有各自的offset值，这个很好理解）

​    2）当slave收到master的N个字节数据后，将本地记录的复制offset值加N。当slave与master链接断开后重练时，此offset用来告知master需要replicaiton的起始位置。

​    3）backlog缓冲区，默认为1M，是一个FIFO队列，全局共享；在master上发生的数据变更，都会写入到backlog缓冲区，同时也会同步给slave。

 

​    当从服务器与master建立连接时，会通过PSYNC指令携带本地的offset值（也包括master ID），如果offset的数据仍然在backlog缓冲区中，那么只需要执行增量同步即可，即将backlog中的数据直接同步给slave即可；如果slave指示的offset数据已经从backlog中移除（这也意味着slave离线的时间过长了），此时将指示slave进行全量复制（SYNC）。

 

​    所以此时，我们需要评估backlog的大小，以尽可能的避免slave闪断后而全量复制，这个值的大小取决于redis的write操作的密度、slave网络稳定性不良带来的延迟时长。

 

​    我们还谈到服务器运行ID，每个redis实例在初始化时都会为自己创建一个server ID，而且此ID一旦创建则不会再改变，而且全局不会重复。在slave与master首次建立连接时，slave会保存master ID，此后，slave与master失去连接后，重连时（包括重新slaveof）将会把offset、master ID发送给master，此后master会判断，slave指示的masterID是否与自己的ID相同，如果相同，则进行增量复制机制（参见上述过程），如果不同，这意味着在slave断开期间，发生了failover、或者slave是迁移到新的master上，此时master将指示slave进行全量复制。

 

​    master与slave之间除了数据同步交互，可以判断链接的活性；此外还开启了心跳检测机制，slave会每隔一秒向master发送“replication ack <offset>”心跳检测指令，来判定链接的活性以及与master的数据间隔的步差；如果心跳检测失败，则会导致slave重建链接。

 

​    对于master而言，可以通过设置：

Java代码  [![收藏代码](http://shift-alt-ctrl.iteye.com/images/icon_star.png)]()

1. min-slaves-to-write 1  
2. min-slaves-max-lag 10  
3. \##单位秒  

 

​    当master持有的slaves个数少于“min-slaves-to-write”时，master将切换为readonly模式，即拒绝客户端的write操作。同样，当“min-slaves-to-write”个数的slaves写入延迟都大于“min-slaves-max-lag”时，master也切换为readonly。