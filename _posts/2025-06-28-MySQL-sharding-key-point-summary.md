---
title: 白话MySQL分库分表
date: 2025-06-28 16:06:00 +0800
categories: [后端, MySQL]
tags: [后端, MySQL, 分库分表]
music-id: 1437233087
---

## **为什么要分库分表？**
当然是数据库有了性能瓶颈，`IO`或`CPU`瓶颈会导致数据库的活跃连接数增加，可能会达到可承受的最大活跃连接数阈值，导致应用服务无连接可用，造成灾难性后果。可以<font color="red">先从代码、SQL、索引、硬件条件（比如CPU、内存、磁盘IO、网络带宽）等方面优化，如果这几项没有太多优化空间，就该考虑分库分表了</font>。

>分库可以提高`MySQL`集群的**并发能力**和**数据容量**，分表可以降低**单表的数据量**，提高查询效率。
- 库表指标合理的参考范围（仅供参考），具体情况应具体分析
    - 单表记录数，`500w-1000w`
    - 单一DB实例，并发峰值`1500-2000 qps`，数据容量`1TB`
{: .prompt-tip }

**IO瓶颈**

1. 磁盘读`IO`瓶颈：对于热点数据，数据库缓存放不下，查询时产生大量磁盘`IO`，查询慢导致产生大量活跃连接。
- 使用一主多从，读写分离，多个从库分摊查询流量；
- 分库 + 水平分表。

2. 磁盘写`IO`瓶颈：数据库写入频繁，频繁写`IO`会产生大量活跃连接。
- 只能分库，用多个库来分摊写入压力；
- 水平分表后，单表存储的数据量还会减少，插入数据时索引查找和更新的成本会更低，插入速度更快。

**CPU瓶颈**
1. `SQL`中包含`join`、`group by`、`order by`，非索引字段条件查询等增加`CPU`运算的操作，会对`CPU`产生明显压力。`SQL`有优化的空间，就针对`SQL`进行优化，也可以把计算量大的`SQL`放到应用中处理；

2. 单表数据量太大（比如超过`1000w`），查询遍历数据的层次太深，或者扫描行数太多，`SQL`效率会很低，也会非常消耗`CPU`，可以根据业务场景分库分表（水平拆分、垂直拆分）。

## **为什么InnoDB单表的容量瓶颈大概在500w-1000w？**

<font color="red">先看MySQL InnoDB单表的容量瓶颈是怎么来的？</font>

从性能上来说，`MySQL`采用`B+`树类型做主键索引，`B+`树有个特点，记录数超过一定量的时候，`B+`树索引树的高度就会增加。而每增加一层高度，数据检索时就会多一次磁盘`IO`。在单表数据量超过经验阈值时，索引树深度的增加会带来磁盘`IO`次数的增加，进而导致查询性能的下降。

<font color="red">为什么B+树每增加一层，查找时就会多一次IO？</font>

`MySQL` `InnoDB`引擎做磁盘`IO`的单位，不是磁盘块，而是一个数据页。
磁盘上读数据，是以块为单位来读的，一个块大小`1-4KB`（可以配置），我们可以认为一块就是`4KB`。块相对来说比较小，为了减少磁盘读取的次数，不是需要哪块读哪块，而是多读一点（预读），进行`IO`时单位一般都不是磁盘块，是一页一页读的（`PageCache`），`PageCache`大多是`4`个磁盘块，比如`InnoDB`默认每页大小为`16KB`，这跟参数配置有关。

![Desktop View](/assets/img/20250628/B+_tree.png){: width="800" height="600" }
_B+树结构示意图（图片来自网络）_

非叶子节点中，存储的是索引的键值和下一层的指针`p1`（指向的是磁盘地址）。中间的这些值（比如`28`）是键值（有序的）。树上的每一个节点都是一个`Page`，读的时候是以一个`Page`为单位读的，如果树越深，`IO`次数就越多（只读根节点，发生一次`IO`就够了，如果树有两层，比如找`79`，就得先读第一页，再读第三页，做两次`IO`）。树的高度，对性能的影响太大了，高度越矮越好。`IO`性能很低，而且页在磁盘上的物理位置不是顺序写。

根节点大小是固定的，一个页`16KB`，根节点就是`16KB`，根节点常驻内存，但是从第二层开始就不是常驻内存了。

<font color="red">为什么说MySQL的InnoDB不用B树，而用的是B+树？</font>

`B`树，数据和键值（或者说索引）是在一块存储的，我们知道键值很小，有的主键就是`8`个字节的长整型，但是数据很大（一条记录后边有`content`，甚至还有长长的`text`，一条记录大小有`2-3M`），一页能放几条记录很难说，假设`1K`一条记录，一页才能放`16`个数据，`B`树每个节点放的数据记录很少的话，这就导致新节点的扩充，树就变得又瘦又高。每增加一层，查找就增加一次磁盘`IO`。用`B`树一下子就干到`10`层，甚至`100`层，性能怎么能快的起来呢。

![Desktop View](/assets/img/20250628/B_tree.png){: width="800" height="600" }
_B树结构示意图（图片来自网络）_

`B+`树和`B`树的根本区别，`B+`树把数据部分从非叶子节点，迁移到叶子节点里边去。非叶子节点省下来的磁盘页空间专门用来放键值和索引，查找时我们都是用键来检索，又不是全文检索，用不到数据来查找。用非主键查找，建好非主键的索引（非聚簇索引），找到主键后再回表查。

<font color="red">回到原来的问题，为什么InnoDB单表的容量瓶颈大概在500w-1000w？</font>

**简单推算下，**

<font color="red">一页16KB，16KB能放多少条索引记录？</font>

一条索引记录分两部分，键值（`BIGINT` `8`字节）、下一页的物理地址（`8`个字节），加起来`16`字节，一页能放`16KB/(8+8)=1K`个键值，为了方便估算，取`10^3`。

<font color="red">500w或1000w条记录，B+树大概有几层呢？换句话说500w或1000w条记录会产生多少次磁盘IO呢？B+树什么时候会增加一层呢？记录数增加到多少的时候，树的高度由两层变成三层呢？</font>

`100w`，就是`10^3 * 10^3`，就两层就够了。理想情况下，三层就是`10^3 * 10^3 * 10^3`，`10`亿条记录。实际上，每个节点可能不能完全填满，如果没能填满，`10`亿条可能就得`4`层或者`5`层了。

所以可以估算`1000w`就是`2`层多一点，可能就是`3`层，`1000w`以上可能就得到`4`层或`5`层以上了。

综上所述，**1000w条记录的B+树高度一般在2-4层**。

<font color="red">3层要发生多少次IO操作呢？</font>

`InnoDB`存储引擎在设计时是将根节点常驻内存的，也就是说，查找某一键值的行记录时最多只需要`1~3`次磁盘`IO`操作。

`B+`树，根本在**减少树的深度**，进而**减少磁盘IO次数**。

## **怎么评估分库分表的规模？**

参见[存储架构篇](https://shouyuanman.github.io/posts/backend-arch-0-1/#%E5%AD%98%E5%82%A8%E7%BB%B4%E5%BA%A6)

## **为什么官方建议使用增长主键作为索引？为什么不能使用UUID作为索引的值？**

**减少页分裂和移动的频率**

结合`B+`树结构的特点，自增主键是连续的，在插入过程中尽量减少页分裂，即使要进行页分裂，也只会分裂很少一部分。并且能减少数据的移动，每次插入都是插入到最后。

>为什么会产生页的分裂？
- 回头看`B+`树的结构，叶子节点中，如果是聚簇索引，`data`就是数据，如果是非聚簇索引，`data`就是主键。键是有序的，方便做二分查找。
- 如果插入数据时，键值是不断增长的，插入数据只需要往后增加即可，不需要改前面的，不需要前面的节点做分裂。
{: .prompt-tip }

## **从B+树出发介绍了垂直拆分表的意义**

为了减少磁盘`IO`，就应该减少每一页中单条记录的大小，每一页就尽可能多放一点记录，读出来的数据行就会更多，在查询时就会减少`IO`次数。
- 列数多，读一个范围的数据可能要读好几页。
    - 假设`100`列，每列`10`个字节，一行就是`1kb`，一个`Page` `16k`，`16`行
- 列数少，读一个范围的数据可能只用读一页一次`IO`就够了。
    - 假设`50`列，每列`10`个字节，一行就是`512` `byte`，一个`Page` `16k`，`32`行

## **分库分表时，分片键怎么选择？**
分片键选择的时候，大概要考虑两个维度，一个是尽可能减少数据倾斜，一个是提高检索性能。

![Desktop View](/assets/img/20250628/sharding_key_select.png){: width="400" height="200" }
_怎么选择分片键_

两大维度不仅涉及到分片键的选择，还涉及到关联关系的考量。选择分片键不止看一个表，还会看周边的关联表，考虑到业务本身，要查哪些表。

梳理下路由规则，看哪些是高性能的，哪些是低性能的。

![Desktop View](/assets/img/20250628/sharding_query_type.png){: width="400" height="200" }
_sharding查询类型_

不包含分片键查询，全库表路由的性能是比较低的，尽可能携带分片键查询。比如，分`100`张表，用年龄来查询，每张表都会去查询，一次查询相当于走了`100`次数据库访问。**不携带分片键查询的场景比较多，或者比较复杂，怎么提高它的性能？其中一种方案是用`NoSQL`解决方案**

**总结**
1. 关联表场景下面，尽量减少笛卡尔积路由（笛卡尔积路由有很多无效查询）；
2. 对于标准路由来说，优化空间比较大的`case`，是关联表绑定（主表、子表）。绑定表要求两个表是同一个分片键，非常适合主子表模式；
3. 使用主键分片，会比较少的产生热点，因为主键在设计的时候，会保证均匀递增；
4. 做分库分表设计的时候，要注意一个大问题，要关联的数据，分片必须要分在同一个库，上面说的所有路由策略，都没有跨库。

![Desktop View](/assets/img/20250628/sharding_non_diff_repo_relation.png){: width="400" height="200" }
_sharding非跨库关联查询_

## **基因法**

用户的订单不在同一个库，查的时候是查不出来的，希望同一个用户的订单路由到同一个库，<font color="red">避免跨库</font>。用户表和订单表，无法通过绑定表的方式实现避免跨库，只能用基因法实现分库基因。

>业务：查询用户的所有订单、查询订单详情<br/>
字段：用户`ID`、订单`ID`<br/>
普通水平切分：
- 根据订单`ID`切分，则无法一次查询用户的所有订单；
- 根据用户`ID`切分，则需要先查订单所属用户。
{: .prompt-tip }

>**基因ID算法**：在分库分表算法一样的情况下，分片的尾数值一样。<br/>
**什么是分库基因？**<br/>
通过`uid`分库，假设分为`16`个库，采用`uid % 16`的方式来进行数据库路由，这里的`uid % 16`，其本质是`uid`的最后`4`个`bit`决定这行数据落在哪个库上，这`4`个`bit`，就是分库基因。<br/>
![Desktop View](/assets/img/20250628/gene.png){: width="400" height="200" }
_基因法_
{: .prompt-tip }

>`32`位是`1`个`G`，`60`位能存放`2^18`个`G`，足够用了。<br/>
如上图所示，`uid=666`的用户发布了一条订单（`666`的二进制表示为：`1010011010`），
- 使用`uid % 16`分库，决定这行数据要插入到哪个库中
- 分库基因是`uid`的最后`4`个`bit`，即`1010`
- 在生成`id`时，先使用一种分布式`ID`生成算法生成前`60 bit`
- 将分库基因加入到`tid`的最后`4`个`bit`
- 拼装成最终`64bit`的`tid`
- 这就保证了同一个用户发布的所有订单的`tid`，都落在同一个库上，`tid`的最后`4`个`bit`都相同
- 通过`uid%16`能定位到库，`tid%16`也能定位到库
- 全局唯一`ID`也可以通过表自增位置和自增范围实现，比如用户`ID`哈希为`1`的表，自增位置为`1`，自增范围为`10`
{: .prompt-tip }

## **分库分表之后怎么进行join操作？**

>首先看是哪种`join`，`join`大致分为左连接、右连接、内连接、外连接、自然连接。<br/>
`Shardingjdbc`不支持跨库关联，之前路由都是说的同一个数据源里边。尽可能使用策略，让查询的数据分到同一个库里。
{: .prompt-tip }

**分库的join怎么解决？**
- 比如经常用左外连接，两个表用相同的分片键，并进行**表绑定**，防止产生数据源实例内的笛卡尔积路由。使得`join`的时候，一个分片内部的数据，在分片内部完成`join`操作，再由`shardingjdbc`完成结果的归并，从而得到最终的结果。
- 另外，可以用**基因法**。

## **分库分表后，跨库关联怎么解决？模糊条件查询怎么处理？**

这两个问题可以用相同的方案解决，采用**查询和存储分离**的方案，即**索引和存储隔离架构**。在结构化存储里完成基础的工作；对于复杂检索或跨库关联，不在`shardingjdbc`做了，引入`NoSQL`。

>业内比较标准的架构
- 使用结构化的`RDBMS`（比如`MySQL`）存储基础数据，完成日常操作（业务通过分片键查询`MySQL`）；
- 对于检索的数据，存在`es`，走高速检索（通过条件查询`es`索引数据）；
- 不是所有数据都需要检索，有大量字段不需要检索，把全量数据存在海量存储`hbase`里（通过`rowKey`查询`hbase`全量数据）。
{: .prompt-tip }

>查询大致分为两类，
- 带着分片键的，就走分库分表；
- 复杂模糊查询或者跨库，把符合条件的`id`从`es`里边查出来，拿到`id`取数据交给`hbase`。
    - `hbase`，`Hadoop`体系下的分布式海量存储中间件，根据`rowKey`查询（前缀树），可以优化到`50w` `qps`。
{: .prompt-tip }

<font color="red">引入这么多的中间件（包含结构化数据、搜索数据、全量海量数据），怎么保障数据一致性？</font>
- **binlog同步保障数据一致性的架构**

在很多业务情况下，我们都会在系统中加入`redis`缓存做查询优化， 使用`es`做全文检索。如果数据库数据发生更新，这时候就需要在业务代码中写一段同步更新`redis`的代码。这种数据同步的代码跟业务代码糅合在一起会不太优雅，可以把这些数据同步的代码抽出来形成一个独立的模块。<font color="red">数据来源是MySQL，把增删改同步到es和hbase里边，使用Canal + MQ，订阅MySQL Binlog日志，发送到消息队列，同步到多个目标。</font>
