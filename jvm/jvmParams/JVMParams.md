## 常用的JVM参数

内容会持续补充

#### 堆内存相关

|序号|参数|解释|
|:---:|:---:|:---:|
|1|`-Xms`|JVM启动时申请的初始Heap值, eg:-Xms1G 堆内存初始值为1G|
|2|`-XX:InitialHeapSize`|JVM启动时申请的初始Heap值，eg:-XX:InitialHeapSize=1024m， 堆内存初始值为1G |
|3|`-XX:MaxHeapFreeRation`|缩小堆内存的时机，默认当堆中空闲内存大于70%时，会减小堆内存，最小到`-Xms`指定值； eg：-XX:MaxHeapFreeRation=70，|堆内存空闲率到70%，触发堆内存缩小
|4|`-Xmx`|JVM可申请的最大Heap值，eg：-Xmx2g 堆最大值为2G |
|5|`-XX:MaxHeapSize`|JVM可申请的最大Heap值，-XX:MaxHeapSize=2048m 堆最大值为2G|
|6|`-XX:MinHeapFreeRatio`|扩大堆内存的时机，默认当堆内存1使用率大于40%时，会扩大堆内存，最大到`-Xmx`指定值， eg：-XX:MinHeapFreeRatio=50，堆区内存使用到达50%触发堆区扩容|
|7|`-Xmn`|Java Heap Young区大小， eg：-Xmn512m，新生代大小为512MB|
|8|`-XX:SurvivorRation`|survivor区和Eden区大小比率,eg:-XX:SurvivorRation=8, s0:s1:eden = 1:1：8|
|9|`-XX:NewRatio`|新生代和老年代的占比，eg：-XX:NewRatio=4 新生代：老年代=1：4|
|10|`-XX:PretenureSizeThreshold`|指定大对象的界限，超过这个值，对象会直接在老年代分配。eg：-XX:PretenureSizeThreshold=1024 对象大小超过1024子节，直接在老年代分配内存|


#### 栈相关
|序号|参数|解释|
|:---:|:---:|:---:|
|1|`-Xss`|指定单个虚拟机栈的大小, eg：-Xss1m 设置虚拟机栈大小为1MB|
|2|`-XX:ThreadStackSize`|指定单个虚拟机栈的大小, eg：-XX:ThreadStackSize=1m 设置虚拟机栈大小为1MB|

**栈大小太小会导致栈深度不够抛出`StackOverflowException`异常，栈太大又会导致线程可以创建的线程数减少。**
**另外，操作系统用对单个线程可以创建的栈数量又限制，线程并不能无限生成**

#### 方法区相关

|序号|参数|解释|
|:---:|:---:|:---:|
|1|`-XX:PermSize`|方法区初始容量， eg：-XX:PermSize=64m， 方法区初始大小为64M|
|2|`-XX:MaxPermSize`|方法区初始容量， eg：-XX:PermSize=512m， 方法区最大为512M|



#### 垃圾收集器

|序号|参数|解释|
|:---:|:---:|:---:|
|1|`-XX:+UseSerialGC`|新生代开启使用Serial垃圾收集器，老年代需要使用SerialOld|
|2|`-XX:-UseSerialGC`|新生代关闭使用Serial垃圾收集器|
|3|`-XX:+UseParNewGC`|新生代开启使用ParNew垃圾收集器， 老年代需要使用功能CMS|
|4|`-XX:-UseParNewGC`|新生代关闭使用ParNew垃圾收集器|
|5|`-XX:+UseConcMarkSweepGC`|老年代开启使用CMS垃圾收集器|
|6|`-XX:-UseConcMarkSweepGC`|老年代关闭使用CMS垃圾收集器|
|7|`-XX:+UseG1GC`|开启使用G1垃圾收集器|
|8|`-XX:-UseG1GC`|关闭使用G1垃圾收集器|



#### GC策略
|序号|参数|解释|
|:---:|:---:|:---:|
|1|`-XX:+PrintGCDetails`|打印详细GC日志|
|2|`-XX:+PrintGCApplicationStoppedTime`|打印暂停时间|
|3|`-XX:InitialTenuringThreshol`|进入老年代最小的GC年龄， 每经历一次Minor GC，年龄+1|
|4|`-Xloggc:`|设置GC日志文件路径|
|5|`-XX:+PrintHeapAtGC`|GC完成后打印JVM堆中各个区域使用情况|
|6|`-XX:+PrintTenuringDistribution`|打印存活实例的年龄信息|
|7|`-XX:+HeapDumpBeforeFullGC`|FULL GC前生成dump文件|
|8|`-XX:+HeapDumpAfterFullGC`|FULL GC后生成dump文件|
|9|`-XX:HeapDumpPath`|设置Dump保存的路径|


#### 其他
|序号|参数|解释|
|:---:|:---:|:---:|
|1|`-XX:+PrintFlagsInitial`|查看所有的JVM参数，可以用来模糊查找|

