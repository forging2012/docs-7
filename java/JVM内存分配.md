**一.运行时数据区:**

 

1. ![img](http://dl.iteye.com/upload/attachment/0082/7007/cc30f034-cc0b-3d69-acee-1413ae24142d.jpg)程序计数器:它是一块较小的内存空间,主要作用是当前线程所执行的字节码的行号指示器.由于java虚拟机的多线程是通过轮流切换并分配处理器执行时间的方式来实现的(协作式/抢占式?!),即任何时刻,任一CPU只会正在处理一个线程的指令;为了确保线程切换后能够正确恢复执行的位置,每个线程都有一个独立的程序计数器,每个计数器为线程私有.如果线程正在执行java方法,那么此计数器记录的是正在执行的字节码指令地址;如果执行的是Native方法,那么它(计数器)的值则为undefined.
2. 虚拟机栈:线程私有,每个线程在执行java方法时都会被创建虚拟机栈,它用来存储方法执行期间的局部变量/操作数栈/动态连接表等.直到方法退出.局部变量表存放了编译期可知的各种基本类型(boolean,int,char...)以及对象引用类型(reference).如果线程请求的栈深度大于JVM所允许的深度,将会抛出statckOverflowError.
3. 本地方法栈:对于native方法调用期间,jvm为线程创建的空间.在hotspot中,本地方法栈和虚拟机栈合二为一.
4. 堆:JVM规范中阐述,所有的对象都将被存储在堆中;堆有可以根据对象的生命周期和收集策略,分为新生代和旧生代;其中新生代中为了便于GC和性能优化,再次分为Eden,from/to交换区三个小区.
5. 方法区:通常称为持久带,此区为线程共享区域;用于存储VM加载的类信息/常量/静态变量等,它作为堆的逻辑部分,但在GC和空间使用上和堆有着很大的不同.方法区一般很少会被GC,但是方法区中的常量池以及类引用的卸载,仍然会被GC;当方法区空间不足时,仍会抛出OOM异常,如JSP预编译,cglib生成类.
6. 运行时常量池:作为方法区的一部分,用于编译期生成的各种字面量和符号引用,它具有动态性,即在运行时仍然可以讲常量添加到常量池中,例如String.intern()方法.常量池会被GC.
7. 直接内存(Directed Menory):通常发生在NIO中为了提升性能,将生命周期较长的buffer声明为直接内存,直接内存不会受到heap区域大小的限制(--Xms,--Xmx参数对其无效),直接内存的大小受到本机物理内存和OS-swap区域大小的限制,如果无法申请到直接内存,JVM也将抛出OOM异常. DirectByteBuffer = ByteBuffer.allocateDirect();

​    JVM采取了"直接内存指针"方式来访问对象,即本地变量表中维护的对象reference直接指向对象数据的内存地址,在此对象数据中包含了此对象类型的引用(Class),此方式对于操作对象更加快捷.

 

**二.JVM常用调优/调试参数:**

 

| -Xms<size>                               | 例如:-Xms2G,设置Heap的初始大小                    |
| ---------------------------------------- | ---------------------------------------- |
| -Xmx<size>                               | 设置Heap的最大尺寸,建议为最大物理内存的1/3(建议此值不超过12G)    |
| -Xss<size>                               | 方法栈大小                                    |
| -Xmn<size>                               | 新生代大小                                    |
| -XX:NewSize=<value>-XX:MaxNewSize=<value>-XX:SurvivorRatio=<value> | 新生代设置.单位为byteSurvivorRatio = Eden/survivor;例如此值为3,则表示,Eden : from : to = 3:1:1;默认值为8 |
| -XX:PermSize=<value>-XX:MaxPermSize=<value> | 方法区大小,max值默认为64M                         |
| -XX:[+ \| -]UseConcMarkSweepGC           | 打开或关闭基于ParNew + CMS + SerialOld的收集器组合.(-XX:+UseParNewGC) |
| -XX:[+ \| -]UseParallelGC                | 在server模式下的默认值(+),表示使用Parallel Scavenge + Serial Old收集器组合 |
| -XX:[+ \| -]UseParallelOldGC             | 默认关闭,+表示打开/使用Parallel Scavenge + Parallel Old收集器组合 |
| -XX:PretenureSizeThreshold=<value>       | 直接晋升为旧生代的对象大小.大于此值的将会直接被分配到旧生代,单位byte    |
| -XX:MaxTenuringThreshold=<value>         | 晋升到旧生代的对象年龄(已经或者即将被回收的次数);每个对象被minor GC一次,它的年龄+1,如果对象的年龄达到此值,将会进入旧生代. |
| -XX:[+ \| -]UseAdaptiveSizePolicy        | 默认开启;是否动态调整java中堆中各个区域大小以及进入旧生代的年龄;此参数可以方便我们对参数调优,找到最终适合配置的参数. |
| -XX:[+ \| -]HandlePromotionFailure       | JDK1.6默认开启,是否支持内存分配失败担保策略;在发生Minor GC时，虚拟机会检测之前每次晋升到老年代的平均大小是否大于老年代的剩余空间大小，如果大于，则改为直接进行一次Full GC。如果小于，则查看HandlePromotionFailure设置是否允许担保失败；如果允许，那只会进行Minor GC；如果不允许，则也要改为进行一次Full GC。 |
| -XX:ParallelGCThreads=<value>            | 并行GC时所使用的线程个数.建议保持默认值(和CPU个数有换算关系).      |
| -XX:GCTimeRatio=<value>                  | GC占JVM服务总时间比.默认为99,即允许1%的GC时间消耗.此参数只在Parallel Scavenge收集器下生效 |
| -XX:CMSInitiatingOccupancyFraction=<value> | 设置CMS收集器在旧生代空间适用占比达到此值时,触发GC.            |
| -XX:[+ \| -]UseCMSCompactAtFullCollection | 默认开启,表示在CMS收集器进行一次Full gc后是否进行一次内存碎片整理,[原因:CMS回收器会带来内存碎片] |
| -XX:CMSFullGCSBeforeCompaction=<value>   | 进行多少次FullGC之后,进行一次内存碎片整理.[原因:CMS回收器会带来内存碎片] |

 

**参考样例：**

JAVA_OPTS = "-verbose:gc 

-Xms3G -Xmx=3G 

-XX:MaxPermSize=256M 

-XX:SurvivorRatio=3 

-XX:MaxNewSize=1G 

-XX:+UseConcMarkSweepGC 

-XX:MaxTenuringThreshold=5 

-XX:CMSInitiatingOccupancyFraction=70 

-XX:+UseCMSCompactAtFullCollection

-XX:+PrintGCDetails 

-XX:+PrintGCDateStamps 

-Xloggc:../logs/server-gc.log.$(date +%Y%m%d%H%M) 

-XX:+UseGCLogFileRotation 

-XX:NumberOfGCLogFiles=1 

-XX:GCLogFileSize=512M"