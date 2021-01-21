# JavaGC测试
环境：jdk1.8

## SerialGC
### 1. 启动java程序
`java -Xms1g -Xmx1g -XX:-UseAdaptiveSizePolicy -XX:+UseSerialGC -jar gateway-server-0.0.1-SNAPSHOT.jar`
![20210121-java01](/Resources/20210121-java01.png)

### 2. 使用 `jps` 查看java进程：
![20210121-jps01](/Resources/20210121-jps01.png)
### 3. 使用`jstat -gc 68265 250 4`,采样时间间隔为250ms，采样数为4：
![20210121-jstat01](/Resources/20210121-jstat01.png)

> S0C、S1C、S0U、S1U：Survivor 0/1区容量（Capacity）和使用量（Used）
> EC、EU：Eden区新生代容量和使用量
> OC、OU：老年代容量和使用量
> MC、MU：元数据区容量和使用量
> CCSC、CCSU 压缩的class文件容量和使用量
> YGC、YGT：年轻代GC次数和GC耗时
> FGC、FGCT：Full GC次数和Full GC耗时
> GCT：GC总耗时

### 4. 使用`jstat -gcutil 68265`
![20210121-jstat02](/Resources/20210121-jstat02.png)
> S0：Survivor 0区使用率
> S1：Survivor 1区使用率
> E：Eden区新生代使用率
> O：Old区老年代使用率
> M：元数据区使用率
> CCS：压缩class空间使用率
> YGC：年轻代GC次数
> YGCT：年轻代GC耗时
> FGC：Full GC次数
> FGCT：Full GC耗时
> GCT：GC总耗时
### 5. 使用`jmc`,并通过`wrk -d60s http://localhost:8088`进行压测
![20210121-wrk01](/Resources/20210121-wrk01.png)


![20210121-jmc02](/Resources/20210121-jmc01.png)
###6. 使用`jmap -heap pid`时候报错异常，会使得gate-server终止，未能成功。??
![20210121-jmap01](/Resources/20210121-jmap01.png)

## ParallelGC
`java -Xmx1g -Xms1g -XX:-UseAdaptiveSizePolicy -XX:+UseParallelGC -jar gateway-server-0.0.1-SNAPSHOT.jar`
![20210121-wrk02](/Resources/20210121-wrk02.png)
![20210121-jmc02](/Resources/20210121-jmc02.png)

## ConcMarkSweepGC
`java -Xmx1g -Xms1g -XX:-UseAdaptiveSizePolicy -XX:+UseConcMarkSweepGC -jar gateway-server-0.0.1-SNAPSHOT.jar`
![20210121-wrk03](/Resources/20210121-wrk03.png)
![20210121-jmc03](/Resources/20210121-jmc03.png)



## G1GC
`java -Xmx1g -Xms1g -XX:-UseAdaptiveSizePolicy -XX:+UseG1GC -XX:MaxGCPauseMillis=50 -jar gateway-server-0.0.1-SNAPSHOT.jar`
![20210121-wrk04](/Resources/20210121-wrk04.png)
![20210121-jmc04](/Resources/20210121-jmc04.png)


# JVM笔记与总结
### JVM调试工具
1.JDK 内置命令行工具
jps/jinfo
**jstat**
**jmap**
**jstack**
jcmd
2.JDK 内置图形化工具
jconsole
jvisualvm
3.JVM 图形化工具
VisualGC-->idea
jmc
### 各个GC对比
![GCCompare](/Resources/GCCompare.png)

### 常用的GC组合
 ![GCRelationship](/Resources/GCRelationship.png)

 常用的组合为:
(1)Serial+Serial Old 实现单线程的低延迟 垃圾回收机制;
(2)ParNew+CMS，实现多线程的低延迟垃 圾回收机制;
(3)Parallel Scavenge和Parallel Scavenge Old，实现多线程的高吞吐量垃圾回收机制;


选择正确的 GC 算法，唯一可行的方式就是去尝试，一般性的指导原则:
1. 如果系统考虑吞吐优先，CPU 资源都用来最大程度处理业务，用 Parallel GC; 2. 如果系统考虑低延迟有限，每次 GC 时间尽量短，用 CMS GC;
3. 如果系统内存堆较大，同时希望整体来看平均 GC 时间可控，使用 G1 GC。 对于内存大小的考量:
1. 一般 4G 以上，算是比较大，用 G1 的性价比较高。
2. 一般超过 8G，比如 16G-64G 内存，非常推荐使用 G1 GC。
最后讨论一个很多开发者经常忽视的问题，也是面试大厂常问的问题:JDK8 的默认 GC 是什么? JDK9，JDK10，JDK11...等等默认的是 GC 是什么?
JDK9起，默认GC为G1
JDK5,6，7，8默认GC为并行GC

Java 目前支持的所有 GC 算法，一共有 7 类:
1. 串行 GC(Serial GC): 单线程执行，应用需要暂停;
2. 并行 GC(ParNew、Parallel Scavenge、Parallel Old): 多线程并行地执行垃圾回收， 关注高吞吐;
3. CMS(Concurrent Mark-Sweep): 多线程并发标记和清除，关注降低延迟;
4. G1(G First): 通过划分多个内存区域做增量整理和回收，进一步降低延迟;
5. ZGC(Z Garbage Collector): 通过着色指针和读屏障，实现几乎全部的并发执行，几毫秒级别的延迟，线性可扩展;
6. Epsilon: 实验性的 GC，供性能分析使用;
7. Shenandoah: G1 的改进版本，跟 ZGC 类似。

可以看出 GC 算法和实现的演进路线:
1. 串行 -> 并行: 重复利用多核 CPU 的优势，大幅降低 GC 暂停时间，提升吞吐量。
2. 并行 -> 并发: 不只开多个 GC 线程并行回收，还将GC操作拆分为多个步骤，让很多繁重的任务和应用线程一 起并发执行，减少了单次 GC 暂停持续的时间，这能有效降低业务系统的延迟。
3. CMS -> G1: G1 可以说是在 CMS 基础上进行迭代和优化开发出来的，划分为多个小堆块进行增量回收，这样 就更进一步地降低了单次 GC 暂停的时间
4. G1 -> ZGC::ZGC 号称无停顿垃圾收集器，这又是一次极大的改进。ZGC 和 G1 有一些相似的地方，但是底层的算法和思想又有了全新的突破。


