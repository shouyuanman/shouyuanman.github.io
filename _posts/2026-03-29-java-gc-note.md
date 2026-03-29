---
title: 话说JVM GC长啥样
date: 2026-03-29 14:35:47 +0800
categories: [后端, 垃圾回收]
tags: [后端, 分布式微服务, 垃圾回收, JVMGC]
music-id: 442503037
---

## **前言**

学习`JVM`的`GC`机制前，一定要得先了解`JVM`运行时的内存结构。下图是`Java8`运行时的内存结构，主要分为`3`大块，

![Desktop View](/assets/images/20260329/JVM_runtime_memory_struct.png){: width="300" height="150" }
_JVM 运行时内存结构_

第一大块，是栈内存、程序计数器、本地方法栈，这一大块是线程独享的。
- 本地方法栈是`JNI`的，`C`语言的本地`native`方法。
右边是堆内存和本地内存，都是线程所共享的。
- 创建对象时所分配的内存空间，都在堆内存里；
- 使用`DirectByteBuffer`创建直接内存在本地内存里。

常说的`MinorGC`，也叫`YGC`，即年轻代`GC`；`FullGC`是收集整个堆，包括`Young`新生代、`Old`老年代，`PermGen`永久代（在`JDK 1.8`及以后，永久代换成了`metaspace`元空间）等，它是回收所有部分的模式。

## **回到GC机制**

`GC`机制是围绕堆内存开展的，堆外内存的回收还是因为在堆内存里边的那一点点冰山的山顶被回收之后，再来回收堆外内存里边所关联的那一部分。

>堆外内存有两部分，
- 一部分是在堆里边的`Java`对象（`DirectByteBuffer`对象），这个对象很小；
- 更多的字节是在本地内存里，用`C`语言的`malloc`分配的。
`DirectByteBuffer`有两部分，底下直接内存部分的回收和堆内的`DirectByteBuffer`对象是相关联的，所以大部分`Java`开发者一般不用关心堆外内存的回收，只需要关心堆内冰山的山顶那部分，在水面上的那部分。
{: .prompt-tip }

下面主要关注堆内存的垃圾回收机制。

堆内存分为两大块，有不同的垃圾回收器，通常情况下，我们可以简单地理解，一大块叫年轻代，一大块叫老年代。

![Desktop View](/assets/images/20260329/heap_memory_struct.png){: width="300" height="150" }
_堆内存结构_

>看到网上有个很形象的比喻，把`Java`的垃圾回收机制，类比厨师做黄焖鸡，先翻炒，然后小火焖，就这么简单。
- 年轻代是炒锅，老年代是焖锅；
- 年轻代又分为`Eden`（鸡下蛋的地儿）、还有两个炒锅（`from`、`to`）。什么是翻炒？翻炒就是`Minor GC`。在年轻代里的`eden`和`from`、`to`区域不断翻炒，翻个`7-8`遍，最多`15`遍，可能这个时间很短，`2`分钟就翻完了；
- 翻完了之后，把它放在大锅（老年代）里边放水焖，在老年代焖的时间就比较长，焖个半个小时，或者一个小时，甚至半天，一道美味的黄焖鸡就做好了。
{: .prompt-tip }

新生代和老年代的比例，默认是`1:2`，意思就是新生代占`1/3`的堆空间，`-XX:NewRatio`参数可以自定义指定这个值。

`MinorGC`，说的是新生代的垃圾回收，清除`Eden`和`from`，转到`to`中。之后`from`和`to`转换，继续清除`Eden`和新的`from`，转到`to`。清除一次后存活超过年龄的，转到老年代；`to`到了阈值后，部分对象转到老年代。
- 晋升老年代参数：`-XX:MaxTenuringThreshold`，为什么最大是`15`，是因为`HotSpot`会在对象头里的标记字段记录年龄，只分配了`4`位，所以最多只能记录到`15`；
- 如果单个`Survivor`区已经被占用了`50%`，那么较高复制次数的对象也会被晋升至老年代。对应`JVM`参数`-XX:TargetSurvivorRotio`。
在年轻代中经历了`N`次垃圾回收后，仍然存活的对象，就会被放到老年代中，可以认为老年代中存放的都是一些生命周期较长的对象。


类比厨师做黄焖鸡，形象地介绍一下`GC`的全流程

`Minor GC`的过程（翻炒`8`遍） + 小火黄焖鸡`20`分钟

首先看下，在炒锅里边翻炒的过程，每翻炒一次，就是一次`Minor GC`。

看`Minor GC`的过程是怎么做的呢？

把炒锅分为三部分，分别是`Eden`、`from`、`to`，默认比例是`8:1:1`，`Eden`可以理解为鸡窝（鸡下蛋的地儿），鸡窝满了，没空间了，在`Java`里边用`new`创建对象都是在`Eden`里边分配内存，如果`Eden`满了，这个时候就把炒锅翻炒一下（触发一次`Minor GC`）。

`Minor GC`的过程（类比：首先翻炒`8`遍黄焖鸡）

在初始阶段，新创建的对象被分配到`Eden`区，`survivor`的两块空间都为空。

如果`Eden`空间占满了（`MinorGC`的一个简单的前提），会触发`Minor GC`。
`Minor`（`Scavenge`）`GC`后，仍然存活的对象会被复制到`S0`（`survivor 0`）中去。
这样`Eden`就被清空，可以继续分配给新的对象。

![Desktop View](/assets/images/20260329/minor_gc_00.png){: width="300" height="150" }
_从Eden区出生了_

经过扫描与标记，存活的对象被复制到空的`S0`区域，不存活的对象会被回收。

`Minor GC`很简单，首先把炒锅里边的对象做一个标记，对`Eden`里边所有对象做标记，标记的方式就是看对象有没有被引用到，引用到的（强引用）标记为蓝色，没引用（弱引用、虚引用）到的标记为黄色。标记出来，怎么做垃圾回收？通过复制，把被引用到的对象（幸存者）复制到`S`这个小区域，暂且叫`S0`区域，`S0`、`S1`无所谓，反正两块大小一样。复制完之后，`Eden`区可以被全部清理（直接清空），清空完的`Eden`区可以放入新的对象了。这是**第一次做`Minor GC`的流程**，垃圾回收的第一轮翻炒就是这样的。

第一次`Minor GC`，此时`S0`和`S1`区对换，所有存活的对象被移动到`S0`区，`S1`区和`Eden`区对象被清空。

![Desktop View](/assets/images/20260329/minor_gc_01.png){: width="300" height="150" }
_第1次MinorGC，从Eden区长大，还活着的就转到S0，等待下一锅翻炒_

第二次翻炒，就有点不一样了。
再下一次的`Minor GC`的流程，大致如下，`S0`和`Eden`中存活的对象被复制到`S1`中，并且`S0`和`Eden`被清空。

`Eden`区不大，过一会又满了，接下来就是下一次翻炒，这次翻炒的情况就变了，`Eden`有对象，`S0`也有对象了，这次把`Eden`和`S0`中的对象都标记，把`Eden`和`S0`区的存活对象都复制到`S1`区，复制完之后，把`Eden`和`S0`中的对象全部清空，这时`Eden`空出来了，可以放入新的对象了。这是**第二轮翻炒**。

![Desktop View](/assets/images/20260329/minor_gc_02.png){: width="300" height="150" }
_第2次MinorGC，从Eden区长大的和S0长大的，活着的都就转到S1，等待下一锅翻炒_

在这一次的`Minor GC`中，`Eden`区和之前一样，存活的对象被复制到`Survivor`区，未存活的对象被回收。
不过这一次不同的是，`Eden`区中存活的对象被复制到了`S1`区，而`S0`中未存活的对象被回收，存活的对象被移动到了`S1`区，`S0`被清空，但是这里之前在`S0`中存活的对象在移动到`S1`中后，年龄要加1，所以此时`S1`中存活了不同年龄的对象。

第二轮翻炒完后，过段时间`Eden`又满了，满了之后又得翻炒，这次把`Eden`和`S1`区的对象都标记了，标记完后再做复制，`Eden`和`S1`区存活的对象都复制到`S0`区，复制完后，把`Eden`和`S1`区全部清空。这是**第三轮翻炒**。

这里有一个重要的点，幸存的对象每做一次复制，年龄要加`1`，为什么叫`Eden`，`Eden`是新出生儿的地方，`0`岁。

![Desktop View](/assets/images/20260329/minor_gc_03.png){: width="300" height="150" }
_第3次MinorGC，从Eden区长大的和S1长大的，活着的都就转到S0，等待下一锅翻炒_

此处省略后面几次翻炒过程，可参照前`3`次翻炒的方式脑补下...

经过几次`Minor GC`过程后，当年轻代中存活的对象年龄达到一个值时，这个值就是前面说的`MaxTenuringThreshold`参数设置的，不设置默认是`8`，就会被从年轻代升级`Promotion`到老年代，从炒锅升级到焖锅。

![Desktop View](/assets/images/20260329/young_promotion_01.png){: width="300" height="150" }
_年轻代准备升级_

送入老年代（类比：小火黄焖鸡`20`分钟）

随着一次次的`Minor GC`，会有对象在达到年龄后被送到老年代。

简单的一个图如下，

在年轻代里边进行分配，反复翻炒，翻炒到`7-8`遍后，升级到老年代（焖锅），截止到目前，鸡到了焖锅里边。

![Desktop View](/assets/images/20260329/young_promotion_02.png){: width="300" height="150" }
_升级到老年代_

>总结
- 在同一时刻，只有`Eden`和一个`Survivor Space`同时被操作；
- 当每次对象从`Eden`复制到`Survivor Space`或者从`Survivor Space`中的一个复制到另一个，
    - 有一个计数器会自动增加值；
    - 默认情况下，如果复制发生超过`8`次，`JVM`会停止复制，并把他们移到老年代中去。
{: .prompt-tip }

老年代的清理

焖锅里边怎么清理？
方法和炒锅不一样。炒锅里边是不断的标记-复制，因为炒锅比较特殊，它有`3`块，其中一块一直空在那里，焖锅是完整的一只锅，焖锅里边不是用的标记-复制，而是用的标记-整理（压缩）的方法。
在回收时，首先做下标记，黄色部分是存活的对象，灰色是可以回收的对象，标记完了之后，就开始做压缩，怎么压缩呢？简单来说，就是从前面挨个儿往前移，把所有存活的对象做整理，后边的地方就空出来了。这时，就能存放从年轻代升级过来要焖的黄焖鸡了。

标记-复制算法
年轻代的`Eden`区和`Survivor`区，`Eden`区的分配是连续的，而且总有一个`Survivor`区是空的，经过一次`GC`和复制，`Eden`区和一个`Survivor`区中的存活对象被复制到另一个`Survivor`区，之后他们里边的对象都被清空，在下一次`GC`时，两个`Survivor`区再交换角色，重复上述过程。

在老年代，存活对象时间较长，标记-复制算法效率不高，一般用的标记-整理算法，存活对象标记，清除未存活对象，并将对象往一端移动，内存是连续的。

参见标记-整理


`JVM`三大`GC`算法

不管是在年轻代，还是在老年代，都分为以下两个阶段。第一阶段都是标记，第二阶段因为区域、垃圾回收器不同，机制都不一样。在老年代里边，复制不好使，复制需要一块空地，老年代没有留这么一块空地，因为要存活的对象比较多，所以用的是整理算法。整理算法上面还有其他提效的方法，比如直接清除。

垃圾回收器的工作流程，大体如下，
1. 标记出哪些对象是存活的，哪些是垃圾（可回收）；
2. 进行回收（清除/复制/整理），如果有移动过对象（复制/整理），还需要更新引用。

先看看标记的部分。

`GC`常用的垃圾回收算法有三种，
- 标记-清除
- 标记-复制
- 标记-整理

| 标记-复制算法 |
| 回收前 | 回收后 |
| :------------- | :------------- |
| ![Desktop View](/assets/images/20260329/gc_mark_copy_01.png){: width="500" height="300" } | ![Desktop View](/assets/images/20260329/gc_mark_copy_02.png){: width="500" height="300" } |

假设绿色标记存活的，黄色标记回收的，做完标记，在回收阶段做复制工作，把绿色存活的对象都复制到空地里边去，复制完后把原来的内存统一做清理。这有两个比较耗时的地方，它要进行对象的复制工作，内存的复制都是比较费时间的，还有一个特点就是，要有一块差不多的空地在那里等着，如果全部是绿色，就得全都复制一遍，如果没有这块所谓的空地，就得需要额外空间进行担保，比较耗内存空间。既浪费了时间，也耗费了空间。

如果绿色的比较少，占用空间小，这种标记-复制算法还是可以的，比如在年轻代里边。

复制回收算法，在对象存活率较高时，就要进行较多的复制操作，效率将会变低。

更关键的是，如果不想浪费`50%`的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都`100%`存活的极端情况，所以在老年代一般不能直接选用这种算法。

标记-清除算法

`Mark-Sweep`是一种非常常见的垃圾回收算法，简单地说，先找出所有不可达的对象，并将它们放入空闲列表`Free`，该算法被`J McCarthy`等人在`1960`年提出，并应用于`Lisp`语言。

`Mark-Sweep`算法分为两个阶段，标注和清除。标记阶段标记出所有需要回收的对象，清除阶段回收被标记的对象所占用的空间。

| 标记-清除算法 |
| :------------- |
| ![Desktop View](/assets/images/20260329/gc_mark_sweep.png){: width="500" height="300" } |

标记阶段和之前的没什么区别，区别在第二阶段，这里直接把可回收对象清理掉，跟标记-复制算法比，既不耗时间（不需要做整体的对象移动），也不耗空间。

从图中可以发现，该算法最大的问题是，内存碎片化严重，后续可能发生大对象不能找到可利用空间的问题。

这些算法都有在垃圾回收器在用，待会可以看下，有哪些垃圾回收器在用标记-清除算法，它们是怎么解决内存碎片化问题的。

| 标记-整理算法 Mark-Compact |
| :------------- |
| ![Desktop View](/assets/images/20260329/gc_mark_compact.png){: width="300" height="150" } |

标记过程和标记-清除算法的一样，但是后续步骤不是直接对可回收对象进行清理。而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

和清理算法比，有个移动对象的时间，有大量的内存复制，这个很耗时间。在没有优化的前提下，这种内存移动必须暂停应用程序，因为对象的引用地址变了，在这个过程中还在修改，就麻烦了。这种算法不耗空间，只耗时间。

>在老年代每次回首时，都有大量存活对象，移动存活对象并更新所有引用对象时，必须全程暂停用户引用程序，这种停顿就是`Stop The World`（对`JVM`的应用层进行停顿）。
{: .prompt-tip }


三色标记法

三色标记法的基本流程

三色标记法，实际上是一个迭代和遍历的过程，迭代的起点是在`GC`的根部开始的，有一个集合`GC Root Set`记录了根部的对象，比如类里的静态变量就是根部的对象，从根部对象开始，不断地遍历堆里边的对象。

要找出存活对象，根据可达性分析，从`GC Roots`开始进行遍历访问，可达的则为存活对象，

我们把遍历对象图过程中遇到的对象，按**是否访问过**这个条件标记成以下三种颜色，
- 白色：尚未遍历过；
- 黑色：本对象已遍历过，而且本对象引用到的其他对象也全部遍历过了；
- 灰色：本对象已遍历过，但是本对象引用到的其他对象尚未全部遍历完。全部遍历后，会转换为黑色。

在刚开始的时候，所有的对象都没有遍历过，所有的对象通通都是白色，

灰色代表的是一种中间状态，处理问题的时候不是非黑即白，灰色代表的就是人情世故里的灰度。

三色标记迭代过程

有一个很重要的前提，遍历是从根部开始的，根部当然就是要留下来的，根部所有的对象都在白色集合中，

假设现在有白、灰、黑三个集合（表示当前对象的颜色），其遍历过程为，

初始时，所有对象都在**白色集合**中；

![Desktop View](/assets/images/20260329/three_color_mark_01.png){: width="300" height="150" }

将GC Roots直接引用到的对象挪到**灰色集合**中；
从灰色集合中获取对象，
- 将本对象引用到的其他对象全部挪到**灰色集合**中；
- 将本对象挪到**黑色集合**中。

![Desktop View](/assets/images/20260329/three_color_mark_02.png){: width="300" height="150" }

把`AD`从白色集合，挪到灰色集合中，接下来开始迭代，从灰色集合获取到对象，开始一个一个处理处于中间状态的对象，先将`AD`的引用移到灰色集合，

假设`D`引用到`E`，先把`E`移到灰色集合里边，把`D`移动到黑色集合里边。等到`AD`处理完了之后（即把所引用的对象都挪到灰色集合中后），把`AD`移到黑色集合里边，表示`AD`的中间状态已经结束，并且`AD`是存活的对象。

![Desktop View](/assets/images/20260329/three_color_mark_03.png){: width="300" height="150" }

接下来不断地迭代，

| :------------- |
| ![Desktop View](/assets/images/20260329/three_color_mark_04.png){: width="300" height="150" } |
| ![Desktop View](/assets/images/20260329/three_color_mark_05.png){: width="300" height="150" } |

重复以上步骤，直至**灰色集合**为空时结束。
结束后，仍在**白色集合**的对象即为`GC Roots`不可达，可以进行回收。

注：如果标记结束后对象仍为白色，意味着已经找不到该对象在哪了，不可能会再被重新引用。

问题
当`Stop The World`（简称`STW`）时，对象间的引用是不会发生变化的，可以轻松完成标记。而当需要支持并发标记时，即标记期间应用线程还在继续跑，对象间的引用可能发生变化，多标和漏标的情况就有可能发生。
`G1`有使用屏障机制来解决多标、漏标问题，待会再说。


常见的`JVM8`垃圾回收器的比较

对主流`JDK`使用到的`JVM`垃圾回收器用的算法，做下简单总结，如下，

新生代垃圾回收器

新生代里边用的垃圾回收器里边用的算法，一般是标记-复制算法，

`Serial GC`是新生代单线程版本，单线程版本效率都不高，使用复制算法，可以说是最基本、发展历史最悠久的回收器。`Serial-New`算法在进行垃圾回收时，必须暂停其他所有工作线程，直到它回收完成。

`Java`程序现在都变成了多线程版本，所以出现了`Parallel GC`，并发的标记-复制，新生代并行回收器，简称为`ParNew`（`ParNew`回收器-复制算法），就是`Searial GC`回收器的多线程版本。

经过不断发展，出现了一种新的框架——`Parallel Scavenge`（并行回收-复制算法），也是新生代并行回收器，这个已经不是用的分代式`GC`算法框架（代码维度）了，它是以目标导向的，目标是达到一个可控制的吞吐量。

用户代码执行时间越高，效率就越好，吞吐量越高。吞吐量可以用参数来配置，待会再说。`Parallel Scavenge`是一个目标导向的回收器，可以配置吞吐量来控制。目标是追求高吞吐量，高效利用`CPU`。

吞吐量，是`CPU`用于运行用户代码的时间，与`CPU`总消耗时间的比值，即吞吐量 = 运行用户代码时间 / (运行用户代码时间 + 垃圾回收时间)。
停顿时间越短，就越适合需要与用户交互的程序，良好的响应速度能提升用户体验，而高吞吐量则可用高效率地利用CPU时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。

老年代垃圾回收器

先看前面`3`大回收算法，哪些适合老年代。
- 标记-复制算法肯定不适合，老年代的内存占比很大，占了堆内存的`2/3`，如果要预留这么大一块空间等着，假如内存`9G`，老年代占`6G`，那么还得有一个预留空间`6G`空闲着；
- 标记-清除算法，适合老年代，但要解决内存碎片；
- 标记整理算法，适合老年代，但要提升移动效率。

首先是`Serial Old`，Serial回收器的老年代版本，用的是标记-整理算法，但也是单线程版本，串行模式，虽然效率低，但也是有使用场景的，比如在`Client`模式的虚拟机里边使用，`Client`模式主要用于客户端的桌面程序。

`Parallel Old`，并行模式，`Serial Old`的多线程版本，即`Parallel Scavenge`回收器的老年代版本，用的标记-整理算法。这是两种传统的`GC`。

`CMS`（`Concurrent Mark Sweep`），标记-清除算法，既不耗时间，也不耗空间。

`G1`（`Garbage First Garbage Collector`），一种更先进的算法，也是基于标记-整理算法，分布式的垃圾回收，分而治之，把内存分成了很多新生代和很多老生代，先不展开，待会再说。`G1`没有传统意义上的新生代、老生代。


垃圾回收器组合选型

哪些是串行，哪些是并行？
哪些效率高，哪些效率低？
谁和谁可以合作，谁和谁不可以合作？

这些垃圾回收器都会导致`STW`，串行停顿时间是最长的，并行短点，`G1`停顿最短。

年轻代算法都是基于复制算法，准确的说是标记-复制算法。因为第一步是要先标记可达对象，然后把可达对象复制到一块空区域，再把原来的区域清空。区别只是`Serial`是串行的，`Serial`工作过程中用户线程都是停掉的。

`ParNew`和`Parallel Scavenge`是并行的。所谓并行是指多个线程同时做垃圾收集的事情，但是仍然是要停下用户线程的工作的。

`Parallel Scavenge`比`ParNew`的一个优势在于`Parallel Scavenge`可以设置自适应调节`Eden`与`Survivor`区的比例、晋升老年代的比例。

`Serial Old`老年代算法采用的是标记整理算法，`Paralled Old`老年代算法采用的也是标记整理算法，不同点只是一个是完全串行的。

`Paralled Old`垃圾回收的时候有多个线程来跑，但是不可以跟用户线程一起跑。但是不管年轻代、老年代，以及目前市面上所有算法都不能避免`STW`（`stop the world`停止用户线程）。

CMS是Concurrent Mark Sweep的缩写，就是并发标记清除算法，它与其他两种老年代算法不同的是它是只标记清除，不整理。
目标是减少STW。

CMS不能配合Paralled Scavenge使用，只能用ParNew。为啥啊。Parallel Scavenge没有使用原本HotSpot其它GC通用的那个GC框架，所以不能眼使用了那个框架的CMS搭配使用

ParallelScavenge（PS）的young collector就如其名字所示，是并行的拷贝式收集器。因为它不兼容原本的分代式GC框架，为了凸显出它是不同的，所以它的young collector带上了PS前缀，全名变成PS Scavenge。对应的，它的old collector的名字也带上了PS前缀，叫做PS MarkSweep。

G1是分布式的标记整理算法

通常来讲STW引用线程的停顿时间：Serial Old > Paralled Old > CMS > G1。但是CMS有个致命的弱点，CMS必须要在老代码堆内存用尽之前完成垃圾回收，否则会触发担保机制，退化成Serial Old来垃圾回收，这时会造成较大的STW停顿。所以DK1.8默认的垃圾收集器是Paralled Scavenge+Paralled Old方式。年轻代用Parallel Scavenge，老年代用Parallel Old。

CMS仅用于老年代回收，

CMS收集器：
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，
CMS基于并发“标记清理”实现，在标记清理过程中不会导致用户线程无法定位引用对象。
CMS仅作用于老年代收集。
CMS的步骤如下：
1. 初始标记（CMS initial mark）：独占CPU，stop-the-world，仅标记GCroots能直接关联的对象，速度比较快；
2. 并发标记（CMS concurrent mark）可以和用户线程并发执行，通过GCroots Tracing标记所有可达对象；
3. 重新标记（CMS remark）：独占CPU，stop-the-world，对并发标记阶段用户线程运行产生的垃圾对象进行标记修正，以及更新逃逸对象；
4. 并发清理（CMS concurrent sweep）：可以和用户线程并发执行，清理在重复标记中被标记为可回收的对象。

优点
- 支持并发收集。
- 低停顿，因为CMS可以控制将耗时的两个stop-the-world操作，保持与用户线程恰当的时机并发执行，并且能保证在短时间执行完成，这样就达到了近似并发的目的。

缺点
- CMS收集器对CPU资源非常敏感，在并发阶段虽然不会导致用户线程停顿，但是会因为占用了一部分CPU资源，如果在CPU资源不足的情况下应用会有明显的卡顿。
- 无法处理浮动垃圾：在执行并发清理步骤时，用户线程也会同时产生一部分可回收对象，但是这部分可回收对象只能在下次执行清理时才会被回收。如果在清理过程中预留给用户线程的内存不足就会出现“Concurrent Mode Failure”，一旦出现此错误时便会切换到SerialOld收集方式。
- CMS清理后会产生大量的内存碎片，当有不足以提供整块连续的空间给新对象/晋升为老年代对象时又会触发FullGC。且在1.9后将其废除。

CMS可以启用一个标记-整理的动作，可以配置，但是这样一来，就回到了解放前，退回到Parallel Old。

但是CMS有个致命的弱点，CMS必须要在老代码堆内存用尽之前完成垃圾回收，否则会触发担保机制，退化成Serial Old来垃圾回收，这时会造成较大的STW停顿。
所以DK1.8默认的垃圾收集器是Parallel Scavenge+Parallel Old方式。

使用场景
它关注的是垃圾回收最短的停顿时间（低停顿），在老年代并不频繁GC的场景下，是比较适用的。


G1收集器

G1全称是Garbage First Garbage Collector，JDK 7更新4中引入的Garbage first收集器（G1）旨在更好地支持大于4GB的堆。
G1收集器的内存结构完全区别于CMS，弱化了CMS原有的分代模型（分代可以是不连续的空间）。
G1的原理是分治法，将堆分成若干个等大的区域。
G1将堆内存划分成一个个Region（1MB-32MB，默认2048个分区），这么做的目的是在进行收集时不必在全堆范围内进行。

应用程序内存，我们往往把堆内存会配置32G，比如ES，Java程序往往会配置8G-16G，
大内存场景下，做一次堆内存老年代的复制、移动、压缩，耗费的时间挺长的，要了命了。
G1原理很简单，就是做分治法，在小的区域上，用多线程做垃圾回收（标记-整理），可以理解为分布式的标记整理，不需要在全堆范围STW，导致JVM停车的时间缩短了，效率也就大大提升了。

![Desktop View](/assets/images/20260329/G1_memory_struct.png){: width="300" height="150" }

每个Region在G1中扮演了不同的角色，比如Eden(新生区)、Survivor(幸存区)或者Old(老年代)。除了传统的老年代、新生代，G1还划分出了Humongous区域，用来存放巨大对象（humongous object，H-obj）。

G1 GC由Young Generation和Old Generation组成。G1将java堆空间分割成了若干个Region，即年轻代/老年代是一系列Region的集合，这就意味着在分配空间时不需要一个连续的内存区间，即不需要在JVM启动时决定哪些Region属于老年代，哪些属于年轻代。因为随着时间推移，年轻代Region被回收后，又会变为可用状态（后面会说到的Unused Region或Available Region）了。

G1年轻代收集器是并行Stop-the-world收集器，和其他HotSpot GC一样，当一个年轻代GC发生时，整个年轻代被回收。

G1的老年代收集器有所不同，它在老年代不需要整个老年代回收，只有一部分Region被调用。

G1 GC的年轻代由Eden Region和Survivor Region组成。当一个JVM分配Eden Region失败后就触发一个年轻代回收，这意味着Eden区间满了。然后GC开始释放空间，第一个年轻代收集器会移动所有的存储对象从Eden Region到Survivor Region，这就是"Copy to Survivor"过程。

对于每个区域使用的垃圾收集算法，实际上G1没有什么创新，年轻代还是并行拷贝，老年代主要采用并发标记配合增量压缩。算法方面也比较成熟了。

G1主要特点在于达到可控的停顿时间，用户可以指定收集操作在多长时间内完成，即G1提供了接近实时的收集特性。

G1的步骤如下：

1. 初始标记（Initial Marking）：标记一下GC Roots能直接关联到的对象，伴随一次普通的Young GC发生，并修改NTAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象，此阶段是stop-the-world操作。
2. 根区间扫描，标记所有幸存者区间的对象引用，扫描Survivor到老年代的引用，该阶段必须在下一次Young GC发生前结束。
3. 并发标记（Concurrent Marking）：是从GC Roots开始堆中对象进行可达性分析，找出存活的对象，这阶段耗时较长，但可与用户程序并发执行，该阶段可以被Young GC中断。
4. 最终标记（Final Marking）：是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set中，此阶段是stop-the-world操作，使用snapshot-at-the-beginning（SATB）算法。
5. 筛选回收（Live Data Counting and Evacuation）：首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划，回收没有存活对象的Region并加入可用Region队列。这个阶段也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率。


G1的特点
- 并行与并发‌：G1充分发挥多核性能，使用多CPU来缩短Stop-The-world的时间。
- 分代收集‌：G1能够自己管理不同分代内已创建对象和新对象的收集。
- 空间整合‌：G1从整体上来看是基于标记-整理算法实现，从局部（相关的两块Region）上来看是基于复制算法实现，这两种算法都不会产生内存空间碎片。
- 可预测的停顿‌：它可以自定义停顿时间模型，可以指定一段时间内消耗在垃圾回收上的时间不大于预期设定值。

使用场景
- G1 GC切分堆内存为多个区间（Region），从而避免很多GC操作在整个java堆或者整个年轻代进行。
- G1 GC 是基于Region的GC，适用于大内存机器。即使内存很大，Region扫描，性能还是很高的。服务端多核CPU、JVM内存占用较大的应用（至少大于4G）
- G1 适用于想要更可控、可预期的GC停顿周期；防止高并发下应用雪崩现象

我们可以从G1的机制和高并发场景的特性来理解这句话：

一、G1如何实现可控、可预期的GC停顿周期
1. ‌基于Region的灵活回收‌
G1将Java堆划分为多个独立的Region，回收时可以精准选择回收价值高、回收成本低的Region，而不是像传统GC那样全量回收整个年轻代或老年代。它会根据用户设定的停顿时间目标，动态调整回收的Region数量，比如用户指定GC停顿时间不超过200ms，G1就会只回收能在这个时间内完成的Region，从而把GC停顿控制在预期范围内。
2. 停顿时间可自定义‌
G1支持通过参数（如-XX:MaxGCPauseMillis）直接指定最大GC停顿时间，虚拟机会在运行过程中根据堆内存的使用情况、对象存活比例等数据，自动调整回收策略，确保GC消耗的时间不会超过设定值，让停顿时间可预测、可控制。

二、如何防止高并发下应用雪崩现象
1. 避免长停顿拖垮系统‌
在高并发场景下，用户请求量巨大，系统需要持续处理请求。如果GC停顿时长不可控，出现长时间的Stop-The-World，会导致大量请求堆积，无法及时处理，进而引发雪崩。G1的可控停顿机制，能保证GC停顿时间在可接受的范围内，不会让系统长时间无响应，避免请求堆积到超出系统承载能力。
2. 并发回收降低影响‌
G1的并发标记、并发回收等阶段可以和用户程序同时运行，不会完全阻塞业务线程。在高并发时，即使进行GC，业务请求也能继续被处理，最大程度降低GC对系统性能的影响，防止因GC导致系统负载骤增、响应超时，最终引发雪崩。

G1是多线程的回收方式，可以发挥多核优势，缩短JVM停顿的时间。

分代收集

整体上来看，它是标记-整理算法；局部来说，是基于复制算法，找到一个空白区来完成复制，因为有很多空白区。
JVM停顿的时间可以预测，有相关的参数可以调整。和Scavenge有点像，比如吞吐量配置也是可以影响STW时间的。


JAVA8的三种垃圾回收器配置

Java8默认的GC收集器

面试问题:JDK8默认使用的垃圾收集器是什么?

用java命令查看了一下

```terminal
$ java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=128183424 -XX:MaxHeapSize=2050934784 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC
java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)
```

UseParallelGC 即 Parallel Scavenge + ParOldGen。

PSYoungGen，PS就是Parallel Scavenge的缩写，

ParOldGen，就是Parallel Old，


进一步

```terminal
$ java -XX:+PrintGCDetails -version

java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)
Heap
 PSYoungGen         total 76288K, used 2621K [0x000000076b500000, 0x0000000770a00000, 0x00000007c0000000)
                    eden space 65536K, 4% used [0x000000076b500000,0x000000076b7f748,0x000000076ff500000)
                    from space 10752K, 0% used [0x000000076ff80000,0x000000076ff80000,0x0000000770a00000)
                    to   space 10752K, 0% used [0x000000076f500000,0x000000076f500000,0x000000076ff80000)
 ParOldGen          total 175104K, used 0K [0x00000006c1e00000, 0x00000006cce90000, 0x000000076b500000)
                    object space 175104K, 0% used [0x00000006c1e00000,0x00000006c1e00000,0x00000006cc900000)
 Metaspace          used 2291k, capacity 4480k, committed 4480k, reserved 1056768k
                    class space    used 254k, capacity 384k, committed 384k, reserved 1048576K
```

这个收集器是在JDK1.6中才开始提供的，在此之前Parallel Scavenge一直处于尴尬的状态。
JDK1.6之前 Parallel Scavenge + Serial Old
JDK1.6以及之后 Parallel Scavenge + Parallel Old
原因是如果新生代选择了Parallel Scavenge收集器，老年代除了Serial Old别无选择，由于老年代Serial Old性能上的拖累，使用了Parallel Scavenge收集器也未必能够在整体应用上获得吞吐量的最大化效果，直到Parallel Old收集器出现后，"吞吐量优先"收集器终于有了名副其实的应用组合
Parallel Old是Parallel Scavenge收集器的老年版本，使用**多线程**和**标记-整理**算法
Parallel Scavenge收集器的关注点与其他收集器不同。Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量（Throughput）
(吞吐量=运行用户代码时间/(运行用户代码时间+垃圾收集时间))

吞吐量（Throughput） CPU用于运行用户代码的时间与CPU总消耗时间的比值；
吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间）；

高吞吐量即减少垃圾收集时间，让用户代码获得更长的运行时间；主要适合在后台计算而不需要太多交互的任务；

Parallel Scavenge & Parallel Old使用场景
适用于一些需要长期运行且对吞吐量有一定要求的后台程序。

怎么控制吞吐量？

Scavenge提供了几个可配置的参数，先简单看下，细节只有在实际用的时候才会深入研究，也不会经常研究，研究完会形成一个模板，后面再用的时候就是微调下模板就可以用了。

重要的参数有三个，其中两个参数用于精确控制吞吐量，分别是
-XX:MaxGCPauseMillis
-XX:GCTimeRatio
-XX:UseAdaptiveSizePolicy

-XX:MaxGCPauseMillis参数 控制最大垃圾收集停顿时间
MaxGCPauseMillis参数允许的值是一个大于0的毫秒数，收集器将尽力保证内存回收花费的时间不超过设定值。

-XX:GCTimeRatio 参数 直接设置GCTime的比例
此参数的值表示运行用户代码时间是GC运行时间的nnn倍。
表示希望在GC花费不超过应用程序执行时间的1/(1+nnn)，nnn为大于0小于100的整数。
举个官方的例子，参数设置为19，那么GC最大花费时间的比率=1/(1+19)=5%，程序每运行100分钟，允许GC停顿共5分钟，其吞吐量=1-GC最大花费时间比率=95%
默认情况下，VM设置此值为99，运行用户代码时间是GC停顿时间的99倍，即GC最大花费时间比率为1%
GCTimeRatio参数的值应当是一个大于0小于100的整数，默认值为99，就是允许最大1%（即1 / (1+99)）的垃圾收集时间。


-XX:+UseAdaptiveSizePolicy是一个开关参数，当这个参数打开之后，就不需要手工指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRatio）、晋升老年代表对象年龄（-XX:PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或最大的吞吐量，这种调节方式称为GC自适应的调节策略（GC Ergonomics）。


JAVA8 使用CMS回收器参考配置
在G1出来之前，CMS绝对是OLTP系统的标配，即使G1出来几年了，生产环境很多的JVM实例还是采用ParNew+CMS的组合。
注意jDK8的默认垃圾收集器是ParMred Scanvenge，不采用CMS是因为CMS不稳定可能会退化成Serial Old。

-server //服务端模式
-Xmx2g //JVM最大允许分配的堆内存，按需分配
-Xms2g //JVM初始分配的堆内存，一般和Xmx配置成一样以避免每次gc后JVM重新分配内存。
-Xm256m //年轻代内存大小，整个JVM堆内存=年轻代 + 年老代 + 持久代
-Xx:PermSize=128m //持久代内存大小
-Xss256k //设置每个线程的堆栈大小
-XX+DisableExplicitGC //忽略手动调用GC，System.gc()的调用就会变成一个空调用，完全不触发GC
-XX+UseConcMarkSweepGC //并发标记清除（CMS）收集器
-XX:+CMSParallelRemarkEnabled //降低标记停顿
-XX+UseCMSCompactAtFullCollection //在FULL GC的时候对老位的压缩
-XX+LargePageSizeInBytes=128m //内存的大小
-XX+UseFastAccessorMethdgs //原始类型的快速优化
-XX+UseCMSInitiatingOccupancyOnly //使用手动定义初始化定义开始CMS收集
-XX:CMSInitiatingOccupancyFraction=70 //使用cms作为垃圾回收使用70%后开始CMS收集

-Xmn和-Xmx之比大概是1:9，如果把新生代内存设置得太大会导致young gc时间较长

一个好的Web系统应该是每次http请求申请内存都能够在young gc回收掉，Fullgc永不发生，当然这是最理想的情况




理论上来说，如果我们内存充足，比如有8-16G，用CMS要比用Scavenge要好点，Scavenge有硬伤，用的是标记-整理算法，整理伴随着停顿，停顿还是比较大的。

CMS在大内存场景下，虽然会产生碎片，但是清除的效率高，而且内存比较足，在G1出来前，在大内存场景下，还是推荐使用CMS。

需要对老年代的碎片做压缩和整理，-XX:UseCMSCompactAtFullCollection，在FullGC时对老年代进行压缩，这是必要的。

G1出来后，对大内存，通通都是用G1回收器


JAVA8 使用G1回收器参考配置

使用g1的核心配置
-XX:+UseG1GC //使用G1收集器
-XX:MaxGCPauseMillis=200 //用户设定的最大gc 停顿时间，默认是200ms。

java 8 update 20刚刚推出的另一个漂亮的优化是G1收集器字符串重复数据删除。
由于字符串（及其内部char[]数组）占用了我们的大部分堆空间，因此进行了新的优化，使G1收集器可以识别在整个堆中重复多次的字符串，并更正它们以指向同一内部字符[]数组，以避免同一字符串的多个副本无效地驻留在堆中。
如果你的应用使用的是G1收集器，并且JDK的版本大于JDK 8 update 20，那么可以尝试开启-XX:+UseStringDeduplication，如果你的应用中存在大量长时间存活的对象，那结果一定还不错。


G1生产环境的配置
-Xms32G // 最小堆
-Xmx320G // 最大堆
-Xss10M // 栈空间
-XX:MetaspaceSize=2G // Metaspace扩容时触发FullGC的初始化阈值

-XX:+UseStringDeduplication // JVM在做GC的同时会做重复字符串消除
-XX:+PrintStringDeduplicationStatistics

-XX:+UseG1GC // 使用G1 GC
-XX:+UnlockExperimentalVMOptions // 允许使用experimental的参数
-XX:G1HeapWastePercent=5 // Sets the percentage of heap that you are willing to waste.
-XX:G1MixedGCLiveThresholdPercent=85 // Sets the occupancy threshold for an old region to be included in a mixed garbage collection cycle. (experimental)
-XX:G1HeapRegionSize=32M // Sets the size of a G1 region.
-XX:MaxGCPauseMillis=10000 // Sets a target value for desired maximum pause time.
-verbose:gc // alias for -XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintGCDateStamps
-Xloggc:path_to_my_gc.log // GC log 文件存放位置

-Dcom.sun.management.jmxremote.port=1234 // 远程调试端口
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

-Dlog4j.configuration=file://$LOG4J_FILE // log4j文件存放位置


关于STW卡顿

从JDK1.3到现在，从Serial收集器→Parallel收集器→CMS→G1，用户线程停顿时间不断缩短，但仍然无法完全消除；



YGC（Minor GC俗称）与FullGC的发生频率和时机

这个问题和直接内存的回收也有关系

Minor GC 实际是很频繁的，比如Minor GC只要大于10秒就不算频繁，算是正常：

一般需要进行GC优化参考指标：
如果各项参数设置合理，系统没有超时日志出现，GC频率不高，GC耗时不高，那么没有必要进行GC优化；如果GC时间超过1~3 秒，或者频繁GC ，则必须优化。
如果满足下面的指标，则一般不需要进行GC优化：
- Minor GC执行时间不到50ms，不需要优化；反之，超过50mS需要优化；
- Minor GC执行不频繁，约10秒一次，不需要优化；反之，超过10s需要优化；
- Full GC执行时间不到1s；反之，超过1S需要优化；
- Full GC执行频率不算频繁，不低于10分钟1次。反之，如果超过10分钟需要优化；

也就是说，最长150S，也就是3分钟之后，如果DirectByteBuffer没有被回收，就会移动到old。



YGC，Eden空间不足，创建对象时就会执行YGC。

FullGC，老年代的空间不足，或者显式调用System.gc()，YGC时可能触发FullGC，dump live内存信息时（导出快照），都会执行FullGC。


面试题：什么时候ygc，什么时候fullgc?
young gc：Eden空间不足执行 young gc，Full GC的时候会先触发 Minor GC
full gc：old空间不足，调用方法System.gc()，ygc时可能触发full gc，dump live的内存信息时(jmap -dump:live)，都会执行full gc

特殊情况：
1. 如果创建一个大对象，Eden区域当中放不下这个大对象，会直接保存在老年代当中，如果老年代空间也不足，就会触发Full GC。为了避免这种情况，最好就是不要创建太大的对象。
2. 发生Minor GC之前，判断：老年代可用空间是否大于当此GC时新生代所有对象容量，或者老年代可用空间是否大于平均晋升大小，如果满足，则执行Minor GC，否则full GC





YGC时可能触发FullGC，就是特殊情况里边的第2种。


Minor GC的结果是，有一些对象要升级，如果所有的对象都升级，都升到老年代去，老年代可用空间如果大于当次GC时新生代所有对象容量，那么这次Minor GC时没有风险的。

如果满足不了上面的条件，还有一个保底的条件，老年代可用空间大于平均晋升大小（历次MinorGC的平均大小），就认为这次升级的大小也会是平均大小左右，所以这次晋升风险是有，但是不大，可以勉强尝试MinorGC。

如果条件二都达不到，毫无疑问，晋升的风险太大了，干脆不做MinorGC，直接FullGC。


垃圾回收线程自动扫描，不是每次都等到创建对象时才做MinorGC。



什么时候发生minor GC(ygc)
- Eden区域满了，或者新创建的对象大小 > Eden所剩空间
- CMS设置了CMSScavengeBeforeRemark参数，这样在CMS的Remark之前会先做一次Minor GC来清理新生代，加速之后的Remark的速度。这样整体的stop-the world时间反而短
- Full GC的时候会先触发Minor GC。

虚拟机在进行minorGC之前会判断老年代最大的可用连续空间是否大于新生代的所有对象总空间
1. 如果大于的话，直接执行minorGC
2. 如果小于，判断是否开启HandlerPromotionFailure，没有开启直接FullGC
3. 发生Minor GC之前，判断：老年代可用空间是否大于平均晋升大小，或者老年代可用空间是否大于当此GC时新生代所有对象容量，如果满足，则执行Minor GC，否则full GC



什么时候会发生FullGC

从年轻代空间（包括 Eden 和 Survivor 区域）回收内存被称为 Minor GC，对老年代GC称为 Major GC，而Full GC是对整个堆来说的，在最近几个版本的JDK里默认包括了对永生代即方法区的回收（JDK8中无永生代了），出现Full GC的时候经常伴随至少一次的Minor GC,但非绝对的。Major GC的速度一般会比Minor GC慢10倍以上。下边看看有那种情况触发JVM进行Full GC及应对策略。

1. System.gc()方法的调用
此方法的调用是建议JVM进行Full GC，虽然只是建议而非一定，但很多情况下它会触发Full GC,从而增加Full GC的频率，也即增加了间歇性停顿的次数。
强烈建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过-XX:+DisableExplicitGC来禁止RMI调用System.gc。
在使用堆外内存的场景，此项配置要非常小心，不建议使用

2. 老年代空间不足
老年代空间只有在新生代对象转入及创建大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误：
java.lang.OutOfMemoryError: Java heap space
为避免以上两种状况引起的Full GC，调优时应尽量做到让对象在Minor GC阶段被回收，让对象在新生代多存活一段时间，以及不要创建过大的对象及数组。

3. 堆中分配很大的对象
所谓大对象，是指需要大量连续内存空间的java对象，例如很长的数组，此种对象会直接进入老年代，而老年代虽然有很大的剩余空间，但是无法找到足够大的连续空间来分配给当前对象，此种情况就会触发JVM进行Full GC。
为了解决这个问题，CMS垃圾收集器提供了一个可配置的参数，即-XX:+UseCMSCompactAtFullCollection开关参数，用于在“享受”完Full GC服务之后额外免费赠送一个碎片整理的过程，内存整理的过程无法并发的，空间碎片问题没有了，但卡顿时间不得不变长了。
JVM设计者们还提供了另外一个参数-XX:CMSFullGCsBeforeCompaction，这个参数用于设置在执行多少次不压缩的Full GC后，跟着来一次带压缩的

4. 永生区空间不足
JVM规范中运行时数据区域中的方法区，在HotSpot虚拟机中又被习惯称为永生代或者永生区，Permanet Generation中存放的为一些class的信息，常量、静态变量等数据，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下也会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：
java.lang.OutOfMemoryError: PermGen space
为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

5. CMS GC时出现promotion failed和concurrent mode failure
对于采用CMS进行老年代GC时，尤其要注意GC日志中有promotion failed和concurrent mode failure两种状况，当这两种状况出现时可能会触发Full GC。

6. Minor GC引发Full GC
发生 Minor GC之前，判断：老年代可用空间是否大于平均晋升大小，或者老年代可用空间是否大于当此GC时新生代所有对象容量，如果满足，则执行，否则full GC




查看GC频率

jstat -gc pid 垃圾回收统计

[root@cdh1 ~]# jps
3712 cloud-config-1.0-SNAPSHOT.jar
3078 QuorumPeerMain
3609 cloud-eureka-1.0-SNAPSHOT.jar
3801 BrokerStartup
3738 sentinel-dashboard-1.7.1.jar
3389 nacos-server.jar
3773 NamesrvStartup
4589 Jps
1071 QuorumPeerMain

[root@cdh1 ~]# jstat -gc 3389 3000
SOC S1C SOU S1U EC EU OC OU MC MU CCSC CCSU YGC
YGCT FGC FGCT GCT
22528.0 15360.0 0.0 14922.8 87040.0 59405.7 29184.0 22575.0 67072.0 63993.0 8704.0 7884.4 21
0.997 4 1.523 2.521
22528.0 15360.0 0.0 14922.8 87040.0 59503.6 29184.0 22575.0 67072.0 63993.0 8704.0 7884.4 21
0.997 4 1.523 2.521


参数说明
- SOC: 第一个幸存区的大小
- S1C: 第二个幸存区的大小
- S0U: 第一个幸存区的使用大小
- S1U: 第二个幸存区的使用大小
- EC: 伊甸园区的大小
- EU: 伊甸园区的使用大小
- OC: 老年代大小
- OU: 老年代使用大小
- MC: 方法区大小
- MU: 方法区使用大小
- CCSC: 压缩类空间大小
- CCSU: 压缩类空间使用大小
- YGC: 年轻代垃圾回收次数
- YGCT: 年轻代垃圾回收消耗时间
- FGC: 老年代垃圾回收次数
- FGCT: 老年代垃圾回收消耗时间
- GCT: 垃圾回收消耗总时间



jstat -gccause进行gc原因分析
上一次gc的原因，并且持续监控，3秒输出一次，可以执行以下命令

[root@cdh1-]#jstat -gccause 3389 2000
S0  S1    E      O    M   CCS          YGC            YGCT              FGC                FGCT                    GCT LGCC                    GCC
0.00  97.15  59.81      77.35        95.41          90.58            21              0.997       4       1.523                2.521 Allocation Failure                   No GC
0.00  97.15  59.81    77.35      95.41        90.58          21            0.997                 4    1.523                2.521 Allocation Failure                  No GC
0.00  97.15    59.81      77.35        95.41        90.58          21            0.997              1.523                2.521 Allocation Failure                  No GC
0.00  97.15  60.05    77.35      95.41      90.58        21          0.997            4              1.523                2.521 Allocation Failure                No GC
0.00  97.15  60.05    77.35      95.41      90.58        21          0.997            4              1.523                2.521 Allocation Failure                No GC
0.00  97.15  60.05    77.35      95.41      90.58        21          0.997            4              1.523                2.521 Allocation Failure                No GC
0.00  97.15  60.05    77.35      95.41      90.58        21          0.997            4              1.523                2.521 Allocation Failure                No GC
0.00  97.15  60.19      77.35        95.41          90.58            21              0.997                4                  1.523                    2.521 Allocation Failure                      No GC
0.00  97.15  60.30      77.35        95.41          90.58            21              0.997                4                  1.523                    2.521 Allocation Failure                      No GC
0.00  97.15  60.38      77.35        95.41          90.58            21              0.997                4                  1.523                    2.521 Allocation Failure                      No GC
0.00  97.15    60.42      77.35        95.41          90.58            21              0.997          4      1.523                    2.521 Allocation Failure                    No GC
0.00  97.15    60.42      77.35        95.41          90.58            21              0.997          4      1.523                  2.521 Allocation Failure                    No GC
0.00  97.15    60.42      77.35        95.41          90.58            21              0.997          4      1.523                    2.521 Allocation Failure                      No GC
0.00  97.15    60.52      77.35        95.41          90.58            21              0.997          4      1.523                2.521 Allocation Failure                    No GC
0.00  97.15    60.54      77.35        95.41          90.58            21              0.997          4      1.523              2.521 Allocation Failure                    No GC
0.00  97.15    60.54      77.35        95.41          90.58            21              0.997          4      1.523                  2.521 Allocation Failure                    No GC
0.00  97.15    60.64      77.35        95.41          90.58            21              0.997          4      1.523                  2.521 Allocation Failure                    No GC









Allocation Failure：
表明本次引起GC的原因是因为在年轻代中没有足够的空间能存储新的数据了。

接下来看下YGC和FullGC的频率，下面有个GC优化的参考指标，我们从这个参考指标不是看怎么做GC优化，而是看在合理情况下的GC频率。


ygc（Minor GC俗称）与Full GC 的优化参考指标

Minor GC 实际是很频繁的，比如Minor GC 只要大于10秒就不算频繁，算是正常：

一般需要进行GC优化参考指标：

如果各项参数设置合理，系统没有超时日志出现，GC频率不高，GC耗时不高，那么没有必要进行GC优化；
如果GC时间超过1-3秒，或者频繁GC，则必须优化。

如果满足下面的指标，则一般不需要进行GC优化：
■ Minor GC执行时间不到50ms，不需要优化；反之，超过50mS需要优化；
■ Minor GC执行不频繁，约10秒一次，不需要优化；反之，小于10s需要优化；
■ Full GC执行时间不到1s；反之，超过1s需要优化；
■ Full GC执行频率不算频繁，不低于10分钟1次，不需要优化。反之，如低于10分钟需要优化；

也就是说，最长150S，也就是3分钟之后，如果DirectByteBuffer没有被回收，就会移动到old。


频率需要看进程的启动的时间来判断，除以它的次数
YGC / YGCT = 单次YGC时间
FGC / FGCT = 单次FGC时间











