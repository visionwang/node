最近阅读了《深入理解java虚拟机》这本书，感觉写的非常不错。总结一下jvm常用参数，jvm参数对虚拟机优化起着至关重要的作用。jvm参数主要包括三种：堆和栈空间设置、垃圾收集器设置（包括并发gc）以及其他的辅助配置（统计跟踪、逃逸分析等），下面详细介绍下这三种参数的配置。至于栈、堆、垃圾回收、线程、字节码等慢慢会在其他博文中描述。

**堆和栈空间设置：**

堆空间：*heap=young+old*，既：堆空间=年轻代+老年代。其中年轻代：*young=2*survivor+eden*，既：年轻代=两个survivor空间+1个eden空间。堆空间保存了new出对象和数组的实际数据，也是gc最爱回收的。同时对对空间进行了分代管理：
	
 - 年轻代：创建的新对象会被放入年轻代的eden空间，而年轻代gc采用复制算法，复制算法会把内存分为两个区域（即两个survivor空间：from和to）。当进行一次minor gc时（既年轻代的gc），minor gc是串行的，eden空间如果没有被gc root引用的会被回收，而依然存活的会被移动到from空间中，如果from空间在minor gc时对象依旧可以存活，就会对该对象年龄+1，当年龄达到一定数值时会直接放入老年代，没有达到年龄的存活对象会被复制到to中。这时from和eden空间已经被清空，虚拟机会交换from和to的空间，空的from变成to，to的变成from，保证了to是空的，minor gc会不断重复这样的工作，直到to彻底被填满，这时会将对象移动到老年代。
	 
 - 老年代：老年代空间的对象是经过minor gc反复锤炼出来的。老年代使用并行的gc回收期，标记-清除算法，并且产生的是full gc(major gc)。老年代gc虽然是并行的，但full gc会同时对年轻代进行gc，所以大量的full gc会严重耗费jvm的性能，甚至卡死应用。另外可以大对象会直接分配到老年代，避免了在minor gc对两个survivor空间的复制耗时。

永久代：也就是方法栈注意java8中永久代被彻底移除了。永久代包含了Class的元信息和常量池。该区域是gc最不爱回收的。

栈空间：分为本地方法栈和虚拟机栈，本地方法栈是被声明为native的方法存放的空间。虚拟机栈是我们通常说的栈，线程私有，随着线程销毁而销毁，gc是不管这里的。包含了：局部变量表、操作数栈、动态链表、方法出口信息等。我们常用的hotspot把本地方法栈和虚拟机栈合成了一个。

 - -Xmx：最大堆内存，如 `java -Xmx1024m`
 - -Xms：初始化堆内存大小，如 `java -Xmx1024m -Xms1024m`，注意如果这最大和初始化堆大小设置相同的话，可以防止jvm的堆内存动态扩容
 - -Xmn：年轻代空间大小，如`java -Xmx1024m -Xms1024m -Xmn256m`
 - -Xss：线程栈大小，如`java -Xmx1024m -Xms1024m -Xmn256m -Xss128k`，注意jdk1.5之前每个线程栈默认为256k，之后是1m，越多的线程栈空间能换取的线程数越少，反之越少的线程栈空间能换取的线程数越多
 - -Xoss：本地方法栈大小，对hotspot无效
 - -XX:NewRatio：设置年轻代与老年代的比例，如`java -Xmx1024m -Xms1024m -Xss128k -XX:NewRatio=4`
 - -XX:SurvivorRatio：设置survivor空间占年轻代空间的比例，如：`java -Xmx1024m -Xms1024m -Xmn256m -XX:SurvivorRatio=4`，eden:survivor=4:2
 - -XX:MaxPermSize：永久代空间大小，如：`java -Xmx1024m -Xms1024m -Xss128k -XX:MaxPermSize=16m`
 - -XX:MaxTenuringThreshold：年轻代最大gc年龄，如果超过这个阈值会直接接入老年代，如果设置为0，年轻代不经过survivor空间直接进入老年代，如：`java -Xmx1024m -Xms1024m -Xss128k -XX:MaxTenuringThreshold=0`
 - -XX:PretenureSizeThreshold：设置大对象直接进入老年代的阈值，当大对象大小超过该值将会直接在老年代分配。如：`java -Xmx1024m -Xms1024m -XX:PretenureSizeThreshold=5242880 `

**垃圾收集器设置：**
在jdk1.6中提供的gc年轻代分为：Serial、Parallel Scavenge、ParNew，而老年代分为：Serial、Parallel、CMS。在jdk1.7中加入了G1。其中Serial为串行gc，是jvm参数指定-client时使用的默认gc。Parallel和ParNew为串行gc，jvm参数指定-server时使用的默认gc为Parallel Scavenge，同样CMS为老年代并行gc，G1为jdk1.7实验性gc，为java9默认的gc。
串行gc在垃圾回收时会stop the world，既停止当前用户线程，然后进行垃圾回收，这样做的目的是防止用户线程继续运行产生内存碎片。而串行gc在垃圾回收时通常不会stop the world，而CMS gc会进行多次垃圾回收（期间会进行一次短暂的stop the world）或者压缩来减少内存碎片。
另外Parallel Scavenge和ParNew的区别在于Parallel Scavenge更关注与吞吐量，既吞吐量配置参数可控。
G1垃圾回收器将内存从分代转换为分块，将内存分配为多块大小相等的heap，每个heap有独立的eden、survivor、old空间，在内存逻辑上都是连续的。采用并行标记压缩方式。

 - -XX:+UseSerialGC：在新生代和老年代使用串休gc
 - -XX:+UseParNewGc：在新生代使用并行gc
 - -XX:+UseParallelOldGC：在老年代使用并行gc
 - -XX:ParallelGCThread：设置Parallel gc的垃圾回收线程数，通常与cpu数量相同
 - -XX:MaxGCPauseMillis：设置最大垃圾收集停顿时间，垃圾回收器会尽量控制回收的时间在该值范围内
 - -XX:GCPauseIntervalMillis：设置停顿时间间隔
 - -XX:GCTimeRatio：设置吞吐量大小，0~100之间的整数。若该值为n，那么jvm将会花费不超过1/(1+n)的时间用于垃圾回收。
 - -XX:+UseAdaptiveSizePolicy：开启自适应gc策略，jvm会根据运行时吞吐量等信息自动调整eden、old等空间大小以及晋升老年代年龄
 - -XX:+UseConcMarkSweepGC：新生代使用ParNew，老年代使用CMS和Serial，其中老年代的Serial用于作为CMS失败时调用的备选gc
 - -XX:+ParallelCMSThreads：CMS线程数量
 - -XX:CMSInitiatingOccupancyFraction：设置老年代空间被使用多少后触发CMS gc，默认为68%
 - -XX:CMSFullGCsBeforeCompaction：设置多少次CMS回收后，进行一次内存压缩
 - -XX:+CMSClassUnloadingEnabled：在类卸载后进行CMS回收
 - -XX:+CMSParallelRemarkEnabled：启用并行重标记
 - -XX:CMSInitiatingPermOccupancyFraction：当永久代空间被使用多少后触发CMS gc，百分比（在使用时CMSClassUnloadingEnabled必须被配置）
 - UseCMSInitiatingOccupancyOnly：只有当gc达到配置的阈值时才进行回收
 - XX:+CMSIncrementalMode：使用增量模式，适合单CPU
 - XX:+UserG1GC：使用G1回收器，与G1相关的虚拟机参数都只能在jdk1.7以上使用 
 - XX:+UnlockExperimentalVMOptions：允许使用实验性参数
**辅助设置：**
 - -XX:+PrintGC：输出GC垃圾回收日志
 - -verbose:gc：与-XX:+PrintGC相同
 - -XX:+PrintGCDetail：输出详细的GC垃圾回收日志
 - -XX:+PrintGCTimeStamps：输出GC回收的时间戳
 - -XX:+PrintGCApplicationStoppedTIme：输出GC垃圾回收时所占用的停顿时间
 - -XX:+PrintGCApplicationConcurrentTime：输出GC并行回收时所占用的时间
 - -XX:+PrintHeapAtGC：输出GC前后详细的堆信息
 - -Xloggc:filename：把GC日志输出到filename指定的文件
 - -XX:+PrintClassHistogram：输出类信息
 - -XX:+PrintTLAB：输出TLAB空间使用情况
 - -XX:+PrintTenuringDistribution：输出每次minor GC后新的存活对象的年龄阈值