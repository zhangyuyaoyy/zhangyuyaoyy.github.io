# JVM


本内容根据B站[黑马程序员JVM完整教程](https://www.bilibili.com/video/BV1yE411Z7AP)整理记录。

<!--more-->

## JVM

![avator](/images/jvm/JVM组成.png)

## 内存结构

![avator](/images/jvm/内存结构.png)

### 程序计数器

程序计数器通过寄存器实现

作用：记录下一条JVM指令执行的地址

特点：

- 线程私有
- 不会线程溢出

### 虚拟机栈

#### 定义

- 栈：线程运行时需要的内存空间。
- 栈帧：方法运行时需要的内存空间。
- 每个栈由多个栈帧组成，对应着每次方法调用时所占用的内存。
- 每个线程只能有一个活动栈帧，对应这当前正在执行的那个方法。

#### 问题辨析

1. GC是否会回收栈内存？
   - GC只会回收堆内存里的对象。
2. 栈内存是否越大越好？
   - 分配太大会导致线程数量减少。
3. 方法内的局部变量是否是线程安全？
   - 如果方法内变量没有逃离方法的作用范围，它就是线程安全的。

#### 栈内存溢出

通过-Xss设置栈内存，默认1024k。

1. 栈帧过多；
2. 栈帧过大。

#### 线程诊断

1. CPU占用过多
   - 定位：
     - top定位进程；
     - ps H -ep pid, tid, %cpu | grep pid 定位线程；
     - jstack pid。
2. 程序长时间没有输出结果
   - jstack pid

### 本地方法栈

#### 定义

- 为本地方法提供内存空间

### 堆

#### 定义

- 通过new关键字创建的对象，都会使用堆内存

#### 堆内存溢出

- 不断创建新对象，同时旧对象一直不被回收。
- 通过-Xms设置堆内存大小

#### 堆内存诊断

1. jps工具

   - 查看当前系统中有哪些java进程

2. jmap工具

   - 查看堆内存占用情况

   - jmap -heap pid

3. jconsole、jvisualjvm

   - 图形化界面，多功能监测工具，可以连续监测。

#### 特点

- 是线程共享的，堆中对象都需要考虑线程安全的问题
- 有垃圾回收机制

### 方法区

#### 方法区内存溢出

- 1.8以前会导致永久代内存溢出，-XX:MaxPermSize
- 1.8以后会导致元空间内存溢出，-XX:MaxMetaspaceSize

#### 运行时常量池

##### 定义

​	二进制字节码的组成：类的基本信息、常量池、类的方法定义（包含了虚拟机指令）

​	常量池，就是一张表，虚拟机指令根据这张常量表找到要执行的类、方法名、参数类型、字面量等信息。

​	运行时常量池，常量池是存在于.class，字节码文件中的。当该类被加载时，它的常量池信息就会放入运行时常量池，并把里面的符号地址变为真实地址。

### StringTable

```java
String s1 = "a";
String s2 = "b";
String s3 = s1 + s2; // new StringBuild().append("a").append("b").toString()    new String("a", "b")
String s4 = "a" + "b"; // javac在编译期间的优化，结果已经在编译器确定为"ab"
String s5 = new String("c") + new String("d"); 
// 串池 ["a", b]  堆 new String("c") new String("d") new String("cd")
String s6 = s5.intern();
// 将这个字符串尝试放入串池，如果有则并不会放入，如果没有则放入串池， 并把串池中的对象返回
```

#### 特性

- 常量池中的字符串仅仅是符号，第一次使用时才变成对象；

- 利用串池机制，来避免重复创建字符串对象；

- 字符串变量拼接原理是StringBuilder（1.8）

- 字符串常量拼接原理是编译器优化

- 可以使用intern方法，主动将串池中还没有的字符串对象放入串池

  - 1.8 将这个字符串对象尝试放入串池，如果有则并不会放入，如果没有则放入串池， 会把串

  池中的对象返回

  - 1.6 将这个字符串对象尝试放入串池，如果有则并不会放入，如果没有会把此对象复制一份，

  放入串池， 会把串池中的对象返回

#### StringTable位置

- jdk1.8
  - StringTable放在堆内存中，Minor GC可以回收内存。
- jdk1.8以前
  - StringTable放在永久代中，只有Full GC才可回收内存。

#### StringTable垃圾回收及调优

- 当内存不够时，JVM会对StringTable进行GC

- StringTable由HashTable实现，通过调整HashTable的桶的个数实现调优

  - ```bash
    -XX:StringTableSize=xxxx
    ```

- 若引用的字符串对象过多，可以通过将字符串入池实现调优

  - ``` java
    string.intern()
    ```

### 直接内存

#### 定义

- 常用语NIO操作，用于数据缓冲区   //ByteBuffer.allocateDirect(1024);
- 分配回收成本较高，但读写性能好
- 不会被GC直接回收  // 通过unsafe释放内存，Cleaner是一个虚引用，当java对象被回收时触发直接内存回收。
- IO操作:
  ![avatar](/images/jvm/IO操作.png)
- 直接内存
  ![avatar](/images/jvm/直接内存.png)

## 垃圾回收

### 如何判断对象可以回收

#### 引用计数法

1. 定义：对于一个对象A，只要有任何一个对象引用了A，则A的引用计数器就加1，当引用失效时，引用计数器就减1。只要对象A的计数器为0，则A不再被使用。

2. 存在的问题：

   1. 引用和去引用伴随加法和减法，影响性能

   2. 很难处理循环引用

      ``` mermaid
      graph LR;
      A-->B
      B-->A
      ```

#### 可达性分析

- java虚拟机中的垃圾回收器采用了可达性分析来探索活着的对象
- 扫描堆中对象，看是否能够沿着GC ROOT对象为起点的引用链，找不到则表示可回收
- 可以通过Memary Analysis(MAT)查看GC ROOT

#### 五种引用

![avatar](/images/jvm/五种引用.png)

1. 强引用

   - 只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收

2. 软引用

   - 仅有软引用引用该对象时，在垃圾回收后，内存仍不足时会再次出发垃圾回收，回收软引用

   - 对象可以配合引用队列来释放软引用自身

3. 弱引用

   - 仅有弱引用引用该对象时，在垃圾回收时，无论内存是否充足，都会回收弱引用对象
   - 可以配合引用队列来释放弱引用自身

4. 虚引用

   - 必须配合引用队列使用，主要配合 ByteBuffffer 使用，被引用对象回收时，会将虚引用入队，由 Reference Handler 线程调用虚引用相关方法释放直接内存

5. 终结器引用

   - 无需手动编码，但其内部配合引用队列使用，在垃圾回收时，终结器引用入队（被引用对象暂时没有被回收），再由 Finalizer 线程通过终结器引用找到被引用对象并调用它的 fifinalize()，第二次 GC 时才能回收被引用对象

### 垃圾回收算法

#### 标记清除

​		标记-清除算法是现代垃圾回收算法的思想基础。标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。一种可行的实现是，在标记阶段，首先通过根节点，标记所有从根节点可达对象（可达性分析）。未被标记的对象就是未被引用的垃圾对象。然后再清除阶段，清除所有未被标记的对象（记录地址的起始位置，不会实际清空字节）。

​		优点：速度快

​		缺点：会产生内存碎片

#### 标记整理(标记压缩)

​		标记压缩算法适合于存活对象较多的场合，如老年代。它在标记-清除算法上做了一定的优化。和标记-清除算法一样，标记-压缩算法也首先需要从根节点开始，对所有可达对象做一次标记。但之后，它并不简单的清理未标记对象，而是将所有存活对象压缩到内存的一端。之后清理边界外所有的空间。

​		优点：不会产生内存碎片

​		缺点：速度慢

#### 复制算法

​		与标记-清除算法相比，复制算法是一个相对高效的回收方法。不适用于存活对象较多的场合。将原有的内存空分为两块，每次只是用其中一块，在垃圾回收时，将正在使用的内存中的存活对象复制到未使用的内存块中。之后清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。

​		优点：不会有内存碎片

​		缺点：需要两倍的内存空间

### 分代回收

#### 定义

- 对象首先分配在伊甸园区域
- 新生代空间不足时，触发 minor gc，伊甸园和 from 存活的对象使用 copy 复制到 to 中，存活的
- 对象年龄加 1并且交换 from to
- minor gc 会引发 stop the world，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行
- 当对象寿命超过阈值时，会晋升至老年代，最大寿命是15（4bit）
- 当老年代空间不足，会先尝试触发 minor gc，如果之后空间仍不足，那么触发 full gc，STW的时间更长

#### 相关参数

| 注释               | 参数                                                         |
| :----------------- | ------------------------------------------------------------ |
| 堆初始大小         | -Xms                                                         |
| 堆最大大小         | -Xmx 或 -XX:MaxHeapSize=size                                 |
| 新生代大小         | -Xmn 或 (-XX:NewSize=size + -XX:MaxNewSize=size )            |
| 幸存区比例（动态） | -XX:InitialSurvivorRatio=ratio 和 -XX:+UseAdaptiveSizePolicy |
| 幸存区比例         | -XX:SurvivorRatio=ratio                                      |
| 晋升阈值           | -XX:MaxTenuringThreshold=threshold                           |
| 晋升详情           | -XX:+PrintTenuringDistribution                               |
| GC详情             | -XX:+PrintGCDetails -verbose:gc                              |
| FullGC 前 MinorGC  | -XX:+ScavengeBeforeFullGC                                    |

### 垃圾回收器

#### 串行

 - -XX:+UseSerialGC=serial/serialOld

 - serial作用于新生代，使用复制算法；serialOld作用于老年代，使用标记整理算法。

 - 单线程

 - 堆内存较小，适合个人电脑

   ![avator](/images/jvm/串行.png)

   **安全点**：让其他线程都在这个点停下来，以免垃圾回收时移动对象地址，使得其他线程找不到被移动的对象

   因为是串行的，所以只有一个垃圾回收线程。且在该线程执行回收工作时，其他线程进入**阻塞**状态

   ##### Serial 收集器

   Serial收集器是最基本的、发展历史最悠久的收集器

   **特点：**单线程、简单高效（与其他收集器的单线程相比），采用**复制算法**。对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程手机效率。收集器进行垃圾回收时，必须暂停其他所有的工作线程，直到它结束（Stop The World）

   ##### ParNew 收集器

   ParNew收集器其实就是Serial收集器的多线程版本

   **特点**：多线程、ParNew收集器默认开启的收集线程数与CPU的数量相同，在CPU非常多的环境中，可以使用-XX:ParallelGCThreads参数来限制垃圾收集的线程数。和Serial收集器一样存在Stop The World问题

   ##### Serial Old 收集器

   Serial Old是Serial收集器的老年代版本

   **特点**：同样是单线程收集器，采用**标记-整理算法**

#### 吞吐量优先

 - 多线程

 - 堆内存较大，多核CPU

 - 单位时间内，STW时间最短

 - jdk1.8默认

   ![avator](/images/jvm/吞吐量优先.png)

   ##### Parallel Scavenge 收集器

   与吞吐量关系密切，故也称为吞吐量优先收集器

   **特点**：属于新生代收集器也是采用**复制算法**的收集器（用到了新生代的幸存区），又是并行的多线程收集器（与ParNew收集器类似）

   该收集器的目标是达到一个可控制的吞吐量。还有一个值得关注的点是：**GC自适应调节策略**（与ParNew收集器最重要的一个区别）

   **GC自适应调节策略**：Parallel Scavenge收集器可设置-XX:+UseAdptiveSizePolicy参数。当开关打开时**不需要**手动指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRation）、晋升老年代的对象年龄（-XX:PretenureSizeThreshold）等，虚拟机会根据系统的运行状况收集性能监控信息，动态设置这些参数以提供最优的停顿时间和最高的吞吐量，这种调节方式称为GC的自适应调节策略。

   Parallel Scavenge收集器使用两个参数控制吞吐量：

   - XX:MaxGCPauseMillis 控制最大的垃圾收集停顿时间
   - XX:GCRatio 直接设置吞吐量的大小

   ##### **Parallel Old 收集器**

   是Parallel Scavenge收集器的老年代版本

   **特点**：多线程，采用**标记-整理算法**（老年代没有幸存区）

#### 响应时间优先

 - 多线程

 - 堆内存较大，多核CPU

 - 垃圾回收时，尽可能让单次STW时间最短

   ![avator](/images/jvm/响应时间优先.png)

   ##### CMS 收集器

   Concurrent Mark Sweep，一种以获取**最短回收停顿时间**为目标的**老年代**收集器

   **特点**：基于**标记-清除算法**实现。并发收集、低停顿，但是会产生内存碎片

   **应用场景**：适用于注重服务的响应速度，希望系统停顿时间最短，给用户带来更好的体验等场景下。如web程序、b/s服务

   **CMS收集器的运行过程分为下列4步：**

   **初始标记**：标记GC Roots能直接到的对象。速度很快但是**仍存在Stop The World问题**

   **并发标记**：进行GC Roots Tracing 的过程，找出存活对象且用户线程可并发执行

   **重新标记**：为了**修正并发标记期间**因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在Stop The World问题

   **并发清除**：对标记的对象进行清除回收

   CMS收集器的内存回收过程是与用户线程一起**并发执行**的

#### G.1 Garbage First

- JDK9默认

##### 适用场景

- 同时注重吞吐量（Throughput）和低延迟（Low latency），默认暂停目标是200ms；
- 超大堆内存，会将堆划分为多个大小相等的Region
- 整体上是标记-整理算法，两个Region之间是复制算法

##### 相关参数

- -XX:+UseG1GC 

- -XX:G1HeapRegionSize=size 

- -XX:MaxGCPauseMillis=time

##### G1垃圾回收阶段

![avatar](/images/jvm/G1.png)

##### Young Collection

- 会STW

![avatar](/images/jvm/G1_YoungCollection.png)

![avatar](/images/jvm/G1_YoungCollection_1.png)

![avatar](/images/jvm/G1_YoungCollection_2.png)

##### Young Collection + CM

- 在 Young GC 时会进行 GC Root 的初始标记

- 老年代占用堆空间比例达到阈值时，进行并发标记（不会 STW），阈值由下面的 JVM 参数决定

  -XX:InitiatingHeapOccupancyPercent=percent （默认45%）

![avatar](/images/jvm/G1_CM_1.png)

##### Mixed Collection

会对E、S、O进行全面回收

- 最终标记（Remark）会STW
- 拷贝存活（Evacuation）会STW

![avatar](/images/jvm/G1_Mixed.png)

##### FullGC

- SerialGC
  - 新生代内存不足发生的垃圾收集 - minor gc
  - 老年代内存不足发生的垃圾收集 - full gc

- ParallelGC
  - 新生代内存不足发生的垃圾收集 - minor gc
  - 老年代内存不足发生的垃圾收集 - full gc

- CMS
  - 新生代内存不足发生的垃圾收集 - minor gc
  - 老年代内存不足
    - 垃圾回收速度小于垃圾生产速度时，转化为SerialGC-full gc

- G1
- 新生代内存不足发生的垃圾收集 - minor gc
- 老年代内存不足
  - 垃圾回收速度小于垃圾生产速度时，转化为SerialGC-full gc

##### Young Collection 跨代引用

- 卡表与 Remembered Set

- 在引用变更时通过 post-write barrier + dirty card queue 

- concurrent refinement threads 更新 Remembered Set

![avatar](/images/jvm/跨代引用.png)

![avatar](/images/jvm/跨代引用_2.png)

##### Remark

- pre-write barrier + satb_mark_queue

![avatar](/images/jvm/Remark.png)

##### JDK 8u20 字符串去重

- 优点：节省大量内存

- 缺点：略微多占用了 cpu 时间，新生代回收时间略微增加

- 参数：-XX:+UseStringDeduplication

- ```java
  String s1 = new String("hello"); // char[]{'h','e','l','l','o'} 
  String s2 = new String("hello"); // char[]{'h','e','l','l','o'}
  ```

- 将所有新分配的字符串放入一个队列

- 当新生代回收时，G1并发检查是否有字符串重复

- 如果它们值一样，让它们引用同一个 char[]

- 与 String.intern() 不一样

  - String.intern() 关注的是字符串对象
  - 字符串去重关注的是 char[]
  - 在 JVM 内部，使用了不同的字符串表

#####  JDK 8u40 并发标记类卸载

- 所有对象都经过并发标记后，就能知道哪些类不再被使用，当一个类加载器的所有类都不再使用，则卸载它所加载的所有类
-  -XX:+ClassUnloadingWithConcurrentMark 默认启用
- 自定义类加载器

 ##### JDK 8u60 回收巨型对象

- 一个对象大于 region 的一半时，称之为巨型对象
- G1 不会对巨型对象进行拷贝
- 回收时被优先考虑
- G1 会跟踪老年代所有 incoming 引用，这样老年代 incoming 引用为0 的巨型对象就可以在新生
- 代垃圾回收时处理掉
- ![avatar](/images/jvm/巨型对象.png)

##### JDK9 并发标记起始时间的调整

- 并发标记必须在堆空间占满前完成，否则退化为 FullGC
- JDK 9 之前需要使用 -XX:InitiatingHeapOccupancyPercent
- JDK 9 可以动态调整 
  - -XX:InitiatingHeapOccupancyPercent 用来设置初始值
  - 进行数据采样并动态调整
  - 总会添加一个安全的空档空间

##### JDK9 更高效回收

- 250+增强
- 180+bug修复
- https://docs.oracle.com/en/java/javase/12/gctuning











### 垃圾回收调优


