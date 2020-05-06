- ### 问题描述：

 &nbsp;&nbsp;新上线的项目发现项目的YoungGC时间过长，因为YoungGC的时候，整个Java进程都是Stop the world 的状态，并且YoungGC的频率很高，所以对服务的稳定性有着极大的危害。如下图：

![gc.png](https://upload-images.jianshu.io/upload_images/22682521-7b3faceb9bfa0f7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### 最终结论

&nbsp;&nbsp;因为公司基础架构组有个数据库(PostgreSQL)健康检测机制，会在服务内对数据库进行心跳检测，但是心跳检测采用的是短链接，检测心跳的频率频率2s/次，如果分库的话，会成倍数增长，并且每个数据库连接都实现了finalize()方法，实现了此类方法的对象，在垃圾回收的时候需要做特殊处理，此类对象过多的时候，导致了GC时间比较长

---

- ### 排查步骤

看到问题现象之后，有以下几个怀疑点：

1. 因为新生代垃圾收集算法采用的复制算法，因为新生代存活对象过多，导致GC时间长

2. String对象过多，StringTable的Hash槽不够用了，导致YoungGC的时候，扫描StringTable花费了大量的时间

3.  因为程序书写不当进入SafePoint的时间过长，导致GC时间过长

4. 清理各类引用的时间过长

于是开始了排查过长，首先用jstat -gcutil 命令观察GC前后的情况，发现并没有很多的存活对象，survivor增长趋势正常，故此排除因新生代存活对象过多导致GC时间长的问题，于是又加上了如下参数，观察GC情况：

##### -XX:+PrintStringTableStatistics

##### -XX:+PrintReferenceGC

##### -XX:+PrintGCApplicationStoppedTime

##### -XX:+PrintSafepointStatistics

加上之后发现有异常的耗时信息，如下：

![gc.png](https://upload-images.jianshu.io/upload_images/22682521-37ec683d57c42309.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 发现清理FinalReference的时候，清理时间过长

###### 找到原因之后，就需要分析一下，FinalReference对象是哪里来的，对应到Java代码的关系是什么？

经过查询资料验证，发现当java对象自己实现了finalize()方法的时候，这类对象就会进入FinalReference队列，等待JVM进行垃圾回收的时候，会调用这个对象的finalize()方法，并判断调用完此方法之后，这个对象是否变可达，如果不可达，下次GC的时候会回收掉，主要的使用场景是一些连接资源用来防止使用者忘记关闭/释放连接，在此方法做兜底，或者有特殊需要定制的对象，想自己控制自己的存活时间，可以在此方法内，让自己变得可达，避免垃圾回收。

观测其源码，如下：

![Finalizer.png](https://upload-images.jianshu.io/upload_images/22682521-5431bd2b773ecb25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Finalizer.png](https://upload-images.jianshu.io/upload_images/22682521-c48aac3f43591816.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到其内部是使用ReferenceQueue来存放所有实现了finalize()方法的对象，但是对象注册进来，是JVM自己的操作，只有JVM才能调用此方法，注册时机为：如果一个类重写了void finalize()方法，并且方法体不为空，类加载时jvm会给这个类加上标记，表示这是一个finalizer类(为和Finalizer类区分，以下都叫f类)。当我们创建一个对象时，会先为分配对象空间，然后调用构造方法。如果创建的是f类对象，默认会在调用构造方法返回之前调用register方法，参数就是当前对象。

找到了问题产生的根源，那如何找到在我们程序里，何处产生了这些对象呢？于是进行内存dump，dump出来之后，分析如图：

![dump.png](https://upload-images.jianshu.io/upload_images/22682521-6b4be1cfe11d7c92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![dump.png](https://upload-images.jianshu.io/upload_images/22682521-aa07e3eb4d772f50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

观测发现，内存中存在大量的PgConnection和SocksSocketImpl对象，跟踪其源码，看看是否实现了finalize()方法，如图：

![PgConnection.png](https://upload-images.jianshu.io/upload_images/22682521-19090f9986c70c51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![SocksSocketImpl.png](https://upload-images.jianshu.io/upload_images/22682521-e53bd44d808ccf08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发现确实是这两个类重写了finalize()方法。创建PgConnection的时候，会创建Socket对象.

我们的项目里都是用连接池，为什么会有这么多pgConnection对象，于是去基础架构组询问，得到的回答是，我们项目的基础包里有一个数据库的心跳设置，2秒一次并且是短链接。回顾之前的gc日志，由于新项目并发量不高，所以youngGC周期间隔较长，平均一小时一次，所以产生大量的f类，下一次youngGC的时候，需要清除这些类，所以时间过长。

---

- ### 解决方案

&nbsp;&nbsp; 先加入此-XX:+ParallelRefProcEnabled参数，并行清理Reference，然后等基础架构组更改为使用长连接做心跳

----
