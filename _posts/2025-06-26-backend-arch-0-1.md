---
title: 笔记——从0-1做业务服务架构
date: 2025-06-26 19:32:00 +0800
categories: [随笔, 架构]
tags: [随笔, 架构, 后端]
music-id: 2609985244
---

## **序言**
>引子<br/>
毕玄曾在`QCon`的围炉夜话中分享——`09`年之前，阿里的技术团队都处于陪（业务）跑状态。
{: .prompt-tip }

大型互联网公司都一定经历了业务初创阶段，再到业务稳定阶段的过程。公司在不同阶段，技术团队的重心会有所不同。初创阶段更多重视业务，业务稳定阶段则会更重视技术本身。

**业务初创时期**，面临的主要问题是把业务转变成真正的系统工程实现，这时的系统架构并不需要设计的非常复杂，因为还没有外部真实流量，仅仅**为了快速迭代和内测实验**，系统由“`APP`-->业务服务（单体）-->单库单表”组成，服务并未按照不同业务拆分，只是从工程角度出发，将应用进行工程包的拆分。

**业务开始趋于成熟**，需要将产品推向`C`端用户，考虑到**业务复杂度**（业务起步阶段并没有C端大流量高并发的特征），开始**按照业务进行简单地微服务拆分（垂直拆分）**。从用户规模、成本控制、系统稳定性、可扩展性，以及快速迭代等角度考虑，系统使用“`App`-->网关-->下游业务服务-->`MySQL`单库单表（按业务服务分库，但每个业务服务依然是单库单表）”的简单架构模式来实现，**初创阶段更侧重于业务功能实现**。此时团队也开始相应地扩张和组织架构调整，组织架构仅粗略地分为业务产研、基础架构研发和运维，其中业务产研按照不同的业务分别负责不同的业务服务，比如账号、商城、消息、内容、活动、权益、金融等。经过一系列重构和扩展，整个系统就形成了以APP为中心的一套轻量级微服务体系架构。

**业务复杂度、业务容量和用户体量不断膨胀**，组织架构不得不开始演变以适应当前阶段（分工越来越细化），比如从业务产研分离出业务网关、业务中台（比如将商城中的订单、交易、支付等能力抽象成中台能力，使得商城业务能更好地聚焦业务，原本的业务服务逐渐演变为业务网关），单独组建数据中台等。**微服务的流量和容量要求也逐渐变大**，各个团队都开始从**三高架构**（高并发、高性能、高可用）出发考虑系统重构，系统逐步演变为“`App`-->网关-->业务接入层-->业务中台层-->`MySQL`分库分表`+``Nosql`”的三高架构模式（包含服务注册发现、分布式缓存、消息队列、熔断限流降级、流量控制（限流/削峰）、无状态设计、弹性伸缩机制、监控告警体系等），理论上系统里边最早能感知到三高需求的业务应该是账号/商品/订单。

重构是在一定阶段后作出的重要决策，不仅仅是重新拆分，也是对业务系统的重新塑造，所以重构时一定要有前瞻性，设计合理的系统架构和成本，切不可盲目。务必对必须完成的不同任务进行优先级排序，找出在哪个阶段应该处理哪些任务，避免过早地进行优化。比如`MySQL`分库分表`+``Nosql`的方案，虽然带来了高性能，但同时也引入了系统复杂度，除非系统不得不重构（`Trade-off Balance`，比如现有架构不能满足当前或未来几年的业务发展）。

>引用`Donald Knuth`老爷子在《计算机编程艺术》这本书中说的一句话——`Premature optimization is the root of all evil`（过早优化是万恶之源）。<br/>
释义：我们应该忘记细枝末节的优化,在`97％`时间，过早的优化是所有邪恶的根源。然而，我们不应该在那个关键的`3％`中放弃我们的机会。
{: .prompt-tip }

## **架构考量维度**
>如果给某一个专业领域的业务服务做系统架构，应该怎么入手？
{: .prompt-tip }

总结下系统架构的结构化思维，我们经常需要考量以下这几个维度，
- 业务维度，做业务梳理和功能分离；
- 模块维度，考虑模块解耦和系统分层；
- 存储维度，对数据存储做技术选型；
- 流量维度，做流量规划，规划每一层用哪些组件；
- 部署维度，根据每个阶段的量去做实际的部署。

以消息中台服务为例，

![Desktop View](/assets/img/20250626/msg_arch_eg.png){: width="600" height="500" }
_示例 整体架构图_

### **业务维度**
>上面这张架构图是经过全面梳理业务之后，精心分析总结得到的，一开始肯定是没有这张图的。
{: .prompt-tip }

架构师要具备**前瞻性**，做业务架构要尽可能地**全面**，该**规划**的就得规划，落实还是另外一回事。前期设计和后期实施，允许有所偏差，但大致主线得差不多，不能偏离太多，否则架构就没什么意义了。

**功能分离**是划分一个个业务模块。业务的功能模块很多，做工程时肯定不可能一蹴而就，一般都得**分阶段架构**，比如可以分一期、二期开发。

模块划分后，从重要性、高可用层面出发做**业务分级**（核心业务/非核心业务），核心业务要重点保障高性能、高可用。

### **模块维度**
#### **模块解耦**
模块解耦和功能分块差不多，但实际上也不一样。一个业务模块在具体实施时可能分为很多子模块。这里关注的是**区分模块外部边界和内部边界**，**分离变化的和不变的**部分，最终形成一个个系统模块。

>以消息服务为例，解耦成两条主线开发，分别是消息下行的主流程、运营后台。
- **消息下行的主流程**去繁就简，大致就是上游业务调用消息服务进行消息下行推送，流量打进来后，先经过接入层（外部`Nginx`网关/`K8S CoreDNS` + 内部微服务网关），之后再到消息服务层，由服务层进行消息的存储和分发，不同通道的消息会分发到不同的下行通道服务，最终通过具体的下行通道完成一次消息下行的过程。
- **消息触达运营后台**，提供的功能是面向内部运营人员的，所以和消息主流程，是完全不同的一条线。
{: .prompt-tip }

针对主流程，怎么做模块设计呢？
- 在**分布式微服务架构开发层面**，肯定要有比如微服务注册中心、内部微服务网关（当然运维部署层面还要有四七层代理、`K8S Service`等基础设施）；
- **网关路由消息到消息分发层**，这里流量打过来有两种模式——同步做接口调用；消息队列异步做消息发送；
- **消息分发层**有两个工作——消息持久化，消息分发（路由）；
- 下边就是**不同渠道的发送服务**，可以把不同渠道分别做一个微服务去对接外部通道；
- **分离变与不变**，在主流程中还穿插着一些重要的公共技术功能模块，比如业务接入流量的认证授权、风控策略、失败补偿、通道调度，把这些公共的功能识别出来，做成公共基础模块。

#### **系统分层**
在微服务架构体系下，系统分层一般包含接入层、服务层、缓存层、数据库层、中间件层、监控预警、基础设施等。
- 接入层：反向代理（业务网关、四七层网关、`K8S`等）；
- 服务层：微服务全家桶生态链，从微服务网关进来后，到微服务实例，包含高可用的注册中心；
- 缓存层：服务本地缓存，或者分布式缓存组件`Redis`等；
- 数据库层：`MySQL`、`Nosql`等；
- 中间件层：`Apollo`、`Nacos`、`Etcd`、`Consul`、`Rocketmq`、`Kafka`、`Xxljob`等；
- 监控预警：`Grafana`、`Prometheus`等。

### **存储维度**
>技术选型要考虑哪些维度？
1. 充分调研业务并抽象出业务模型，看数据模型是否结构化，数据结构是否固定；
2. 数据规模是否很大（海量），这个是按时间段来的，比如一两年内会达到什么样的量级；这里的规模很大，指的是海量`PB`级，如果规模很大，即使用`sql`，需要考虑把历史数据备份，放到`Nosql`里边。
3. 团队是否具备`Nosql`人才，是否能支撑`Nosql`的开发和维护工作。
{: .prompt-tip }

技术两大方案
1. 结构化（`MySQL`）
2. 结构化+非结构化混合（`MySQL + ES + HBase`）
3. ~~纯NoSQL？~~<font color="red">完全抛弃结构化，目前不太现实（比如`Nosql`的事务支持问题），所以这种暂时不考虑。</font>

存储数据规模要从**流量**和**容量**两方面考虑，流量和容量规模决定了存储架构的技术选型。

#### **流量规模**
首先是**数据的吞吐量**（数据流量规模），根据不同阶段的流量，做不同阶段的流量架构。数据访问量指标一般有`Qps`、`Tps`，我们需要预估访问数据库的峰值。

>服务的峰值预估多少才合理？
{: .prompt-tip }

一般系统的性质不一样，预估的方式也不一样。比如秒杀场景，可以参考`Toc`的系统预估方式。

输入参数，系统大致的用户量，但消息服务并不是`Toc`的，而是作为公共服务，提供给内部的第三方系统来使用，这时`Toc`的方式就失效了。

可以简单做个调查，逆向的方式定一个流量规模。跟内部的第三方系统的`owner`做一个调研，基本上可以确定当时服务的流量需求。但不排除特殊情况，比如业务临时排进一个需求，需要`1s`推出`100w`条短信，这时`qps`非常大。所以没办法做很准确的流量预估。

这种高水位值和低水位值相差太大的场景，只能先按照平常预估一个`qps`来做，比如`10w`。`100w`用`10s`发完，消息系统一般允许有些延迟。大部分公司`1w` `qps`就能满足绝大多数场景，但作为基础中台，流量峰值还是比较高的。

#### **容量规模**

其次是**数据容量**（数量存储规模），不同阶段存储记录条数多少，数据规模大小多少`TB`（比如一年或两年内会数据存储规模达`10TB`或`100TB`），需要提前考虑`100TB`或者`500TB`要怎么分表？

>预估下每天消息量是`10w`，`100w`，还是`1000w`。<br/>
- 按照每天`1000w`消息量，`2`年内保持稳定的要求，表数据量规划如下，
    - 两年的消息总量：`1000w`*`730`天=`73`亿；
    - `73`亿，约为`100`亿，约为`10G`，一条消息`20k`，则为`200T`的规模。
- 按照每天`100w`消息量，`2`年内保持稳定的要求，
    - 两年的消息总量：`100w`*`730`天=`7.3`亿；
    - `7.3`亿，约为`10`亿，约为`1G`，一条消息`20k`，则为`20T`的规模。
{: .prompt-tip }

#### **规划MySQL库表数量的Case**

>按照每天`1000w`消息量，`2`年内保持稳定的要求，表数据量规划怎么规划？
{: .prompt-tip }

两年的消息总量为`73`亿，假设每张表的条数上限规定为`500w`，就得需要`1460`张表，比较接近`2`的幂——`1024`。`73`亿/`1024`=`692`w，这个也是在接受范围内的，所以可以按照`1024`张表来规划。

不过考虑到成本控制，刚开始一般表的数量很少，后边会按需扩容。比如刚开始`4`张表，后边翻倍扩容就好了。扩容涉及到数据迁移，具体实施一般由`DBA`来做，专业的事还得交给专业的人来做，就算是系统架构师做数据迁移，一般也不是特别熟悉。实际工作中也有安全责任制，`DBA`也不会允许其他职能的人做这些事情。

>按照`5w` `qps`的流量峰值要求，库的数量怎么规划？
{: .prompt-tip }

假设每个库正常承载的写入并发量为`1500` `qps`，`32`个库就能承载`32*1500=4.8w`写并发。

如果可以的话，悲观预估下，每秒写入超过`5w` `qps`，可以使用`MQ`削峰、批量写入等降级策略。`MQ`的写入吞吐量可以轻松到达`10w`级别。

自然，按照`10w` `qps`的流量峰值要求，进行库的规划，需要`64`个库。

>**综合规划**<br/>
- 一个数据库实例假如`1T`，`20T`就得`20`个数据库实例，变成`2`的倍数，就是`16`；<br/>
- 一个数据库实例`qps`峰值`1500`，`5w` `qps`需要`32`个库；<br/>
- 容量和流量一起评估出来，最后取个最大值。无非就是`32`个库`2s`推完`10w`；`16`个库`4s`推完`10w`。
{: .prompt-tip }

### **流量维度**

这里说的是整个系统的流量架构，它是单独的。整个分层的流量架构在设计时也是一层一层来的。比如接入层做流量规划，需要哪些组件，而上面说的数据存储维度中的流量规划实际上是数据层的。

>看流量架构怎么从`0-1`开始？
{: .prompt-tip }

#### **流量规模预估（流量规划）**

要考虑前瞻性，一年或两年之内会增长到多少用户？预估一般会分为多个阶段去估。

消息服务作为一个基础中间件系统，流量峰值不太好预估，只能先按额定量的模式来做，比如先定`10w` `qps`。

>**ToC场景，可以使用2-8法则，预估吞吐量规模**<br/>
假设用户量有`10w`，
- 平均`20%`是活跃用户，每个用户平均`30`次点击（参考淘宝）
    - `20% * 10w * 30 = 60w`
- `80%`的点击量发生在20%的时间里
    - `60w * 80% / 5 * 3600 = 27`
- 设置一个偏离系数，算出来`qps/tps`
    - `27 * 4 = 108 qps`
- 假设一个请求`20k`，算出每秒的流量
    - `108 * 20k = 2M`
- 注意，在`ToC`是这样的，`ToB`后台就不一定是这样，中台也不能按照面向用户这么规划，内部消息有时会很频繁，这时就得按照实际场景去做实际的流量规划理论分析。
{: .prompt-tip }

#### **了解服务中各个组件的服务吞吐能力**
这个指标，一般得参考组件的硬件和系统资源，这里只做一个大致的预估，这里的数据仅供参考，仅用这里的估值来预算中间件的规模。

>悲观地预估，
- 接入层
    - `LVS` `10w-17w` `qps`
    - `Nginx` `5w-10w` `qps`
- 服务层
    - `SpringCloud gateway` `5k` `qps`
- 异步削峰
    - `MQ` `10w` `qps`
- Java服务
    - `Tomcat` `1k` `qps`
- 存储
    - `MySQL` `1.5k` `qps`
- NoSQL集群，不是单机的指标
    - `Redis` `5w+` `qps`
    - `Hbase` `7w+` `qps`
    - `ES` `7w+` `qps`
{: .prompt-tip }

#### **真实按照每一层的流量预估，做流量架构**

![Desktop View](/assets/img/20250626/qps_arch.png){: width="600" height="200" }
_示例 流量预估_

`10w` `qps`，刚开始肯定不会那么高，有可能一年达到，有可能两年。

`Java`服务这里根据时效、性能要求，估计`Tomcat`数量。收到消息，到发出去，`rt`时间去规划。

`10w / 1k = 100`

到`Redis`集群这，经过服务削峰这一层，流量已经下来了。

### **部署维度**
部署的行为，就和规划没有那么强关联了，需要做的就是根据每个阶段的量去做实际的部署。比如考虑高可用，看资源怎么规划，网络结构是怎么样的，每一层有多少组件，分别部署在哪个网段。如果你的网络结构比较简单，搞一下两个跨域的可用区，每个可用区各一套，做一下每个可用区的规划。

这里就是一个五花八门、变化多样的架构设计，跟网络环境有很大的关系。各个公司的架构师技术栈不一样，解决方案多种多样，没有标准解决方案，但是目标是一样的。有的部署在公有云，有的部署在私有云，有的是自建的`IDC`机房。部署架构要有一个分而治之的思路（分层的部署架构方案，一层一层剥洋葱，一层一层来设计），有两个非常重要的考量要素，首当其冲的是高可用（很复杂的目标），另一个是单元化，每一层都要做单元化和高可用的考量。

#### **高可用**
- **接入层高可用**
    - 常见的方案有异地多活、`HTTPDNS`、`KeepAlive`保活、防`DDOS`攻击等。
    - 异地多活，避免机房级别，甚至是地域级别的灾难，也就是常说的跨国多活、跨域多活、同城多活。
- **服务层高可用**
    - 注册中心高可用，微服务实例数`>=2`
- **缓存层高可用**
    - 代理模式、哨兵模式、`Cluster`模式
- **数据库层高可用**
    - 代理型高可用（`MyCat`）、客户端高可用（`Shardingjdbc`）、配置型高可用（进行高可用地址的更换）
- **其他中间件高可用**
    - 保证服务总线、反向代理、`ES`、`MQ`高可用
- **容错方案**
    - 重试机制，比如客户端容错（`HTTPDNS`）、错误重试机制、备用服务地址重试
- **运维与监控**
    - 立体化监控预警
    - 基础设施的监控，比如虚拟机、容器、`Nginx`、数据库、缓存等
    - 应用层监控，比如核心业务指标监控、分布式链路跟踪、慢查询、慢调用等
    - 接口自动巡检，假死自启动等

#### **单元化**
单元化，是在做异地多活时，一个地域形成完整的业务闭环（即在一个地域上，这个单元是可以独立可运行的），地域之间不依赖另外一个地域存活，脱离另一个单元，这个单元不会受到任何影响。单元化是垂直拆分的，根据用户`id`划分单元即可。在高可用场景下，系统做跨地域的部署（一般系统用不到）。

虽然要满足单元化，但是也要满足单元之间具备故障转移机制，比如北京的某个服务挂了，流量要可以转移到上海机房的相应服务上。

机房之间既要独立，又要进行数据同步，还要考虑同步的延迟问题，考虑数据不一致的时间问题。如果出现数据同步的时间太长，或者数据不一致的时间太长，一旦出现故障的时候，就要考虑切换。数据不一致的周期也要考虑。

#### **Case**
下面基于私有云做部署架构。私有云不叫机房，叫可用区`zone`。假设有两个可用区在两个城市，比如北京、上海。

![Desktop View](/assets/img/20250626/deploy_arch.png){: width="600" height="200" }
_示例 流量预估_

上游业务实际上会通过内网域名或者`K8S ServiceName`方式访问消息服务，这里我们只画了内网域名的接入层。

业务系统通过域名访问消息服务，用智能`ldns`（本地`dns`）解析正确的`vip`，当一个可用区的`dns`挂掉后，可以转移至另一个可用区的`dns`解析`vip`。

接入层通过`vip`访问`LVS`，`vip`由`keepalive`提供，也就是通过`keepalive`来做故障转移。

接下来是整个微服务治理体系，`RocketMQ`最好单独做到两个可用区高可用。

最后是缓存和数据层。

### **扩容维度**

>**为什么在架构设计时，要考虑扩容预案？**<br/>
做架构要避免返工，重新推倒重来。怎么避免返工，就要提前做好规划，这种规划会影响很多算法的选择，比如`ID`生成、`ID`分片算法怎么分等。这就是前瞻性。
{: .prompt-tip }

梳理分库分表扩容的场景，有两大场景，

1. 流量瓶颈后，扩容的目标库数量计算，

    `2`个库约为`2000-3000` `qps`，可以实现`10w`消息，`50s`内完成。
    
    目标：`10w`消息要在`10s`内完成
    
    扩容到`4`个库够吗，不行就`80`个库。

2. 容量瓶颈后，扩容的目标库数量计算，

    `2`库`4`表，共`8`个表，目前有`4000w`记录
    
    需要扩容到`16`个表

>从架构师视角来看，扩容的工作大致包括哪些步骤？
{: .prompt-tip }

扩容工作，是以`DBA`为主导，架构师是打配合的。`DBA`不懂或者不管上面说的规划，那是架构师规划的，一般落地得`DBA`来做。

配合`dba`做好存量复制，和增量数据复制，
1. 建好要扩展的库表；
2. 开启增量数据的复制，可以使用`Canal`这样的`Binlog`增量订阅和消费工具， 把增量数据进行二次路由，插入到新的库表中；
3. 开启存量数据的复制， 使用 `select into` 临时表 `from` 原始表 `where` `id%2` 导出数据，复制到新表。

更换的方式，
1. 冷更换：准备好数据后，在一个没有人烟的深夜，去停服重启，更换数据源；
2. 热更换：准备好数据后，在访问的低峰时段，开启更换按钮，更换数据源。

## **总结**
架构的工作，就是`Trade-off Balance`，从宏观到微观，由粗到细，千万不要一开始就掉到细节的汪洋大海里边去。

>**Trade-off Balance，怎么理解？**<br/>
`Trade-off Balance`，衡量工作怎么做才能更好，这需要有宽广的知识面和一定的技术深度来支撑。<br/>
架构师做一件工作，要随时对这件工作需要耗费的成本和所要达到的成果之间进行权衡评估。
{: .prompt-tip }

`2025`年 于 北京 记
