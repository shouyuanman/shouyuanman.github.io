---
title: 线上服务紧急告警，调优Java GC
date: 2025-03-27 23:34:35 +0800
categories: [后端, 垃圾回收]
tags: [后端, 分布式微服务, 垃圾回收, JavaGC]
music-id: 442503037
---

## **Case**

相信很多微服务开发老`baby`经常会碰到这样一种情景，当你正沉浸式地敲着纯正老代码的时候，`OnCall`告警群里突然蹦出一堆线上紧急告警，告警信息里明晃晃地写着`服务OverLoad过载`。这时你不得不从沉浸的编码世界里抽身出来，本着线上告警优先处理的原则，观察基础/业务监控大盘和详细的服务过载日志来定位解决问题。

>Tips: 如果短时间找不到问题，可以先临时扩容（虽然不一定有效），先恢复服务业务，再去定位问题，及时止损。
{: .prompt-tip }

首先看到的是，业务流量激增，

![Desktop View](/assets/img/20260328/biz_qps_surge_01.png){: width="300" height="150" }
_业务请求量激增_

相应地，会直接带来机器负载升高，应用`gc`压力增大，甚至还会导致依赖的下游服务或者存储中间件耗时增加，可以调取观察机器负载（比如`CPU`、内存、网络、磁盘等）、`JVM`、外部调用及耗时指标监控，直观地获取服务运行状况的第一手资料。

| 机器负载 |
| 服务`2C4G`小型配置 |
| `CPU`、内存相对敏感 |
| :------------- | :------------- | :------------- |
| ![Desktop View](/assets/img/20260328/machine_cpu_01.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_memory_01.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_core_per_minute_01.png){: width="500" height="300" } |
| ![Desktop View](/assets/img/20260328/machine_disk_rw_01.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_disk_space_01.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_network_01.png){: width="500" height="300" } |


| `JVM`，把时间线拉长一些，可以看到许多时刻有尖刺 |
| :-------------------------- |
| ![Desktop View](/assets/img/20260328/G1_gc_base_info_global_01.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/G1_gc_base_info_02_global_01.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/G1_gc_collector_info_global_01.png){: width="800" height="600" } |

| `JVM`，聚焦到尖刺的时刻，可以看到更详细直观的曲线 |
| 前提：虽有业务流量增加，但也不是能达到需要限流的地步，否则就得考虑限流或者扩容了 |
| <font color="red">看图说话：老年代平常用的其实不多，当新生代疯狂gc，基本都是一次性的对象，导致CPU和内存过载</font> |
| <font color="red">随后导致短期内新生代晋升老年代激增，老年代空间不足以容纳从年轻代晋升的对象，导致了频繁FullGC</font> |
| <font color="orange">看起来2C4G配置对CPU、内存相对敏感，服务用G1 Collector收益不高</font> |
| :-------------------------- |
| ![Desktop View](/assets/img/20260328/G1_gc_base_info_local_01.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/G1_gc_base_info_02_local_01.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/G1_gc_collector_info_local_01.png){: width="800" height="600" } |

如有需要，还可以继续通过查看过载时打印的日志，了解过载时应用内的线程被哪些请求使用、线程`stack`日志，以及过载的请求明细，来进一步定位问题。

>如果在一个组织健全、分工明确的团队，找到原因后，可通过以下方式针对性地去解决，否则的话，只能默认你能后端全栈了。
1. 业务流量激增：先确定流量来源，之后和上游调用方确认是否是正常流量；
2. 机器负载升高：找运维协助排查`CPU`、内存、网络等基础因素，如果是单台机器问题，可以执行迁移；
3. 依赖外部系统耗时增加：找下游服务`Owner`或者`DBA`协助排查解决；
4. `gc`问题：分析服务内存，如果没有`dump`文件，找运维执行`dump`，分析`dump`文件（这一步最好在线下环境复现，线上`dump`容易产生`FullGC`）。
{: .prompt-tip }

## **`JVM`参数**

### **查看`JVM`参数**
服务启动时，配置的`JVM`参数，如下，
```terminal
$ ps -ef | grep java
/usr/local/jdk1.8.0_192/bin/java \
  -Dfile.encoding=utf-8 \
  -Djava.awt.headless=true \
  -Djava.net.preferIPv4Stack=true \
  -server \
  -Xms2g -Xmx2g \
  -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m \
  -XX:CICompilerCount=4 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/xxx/logs/xxx_logs/dump \
  -XX:ErrorFile=/xxx/logs/jvm_error_file_logs/java/java_error%p.log \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -XX:G1HeapRegionSize=4M \
  -XX:ParallelGCThreads=2 \
  -XX:+UseStringDeduplication \
  -Dcom.sun.management.jmxremote.port=50000 \
  -Dcom.sun.management.jmxremote.rmi.port=50000 \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Dcom.sun.management.jmxremote.authenticate=false \
  com.xxx.Main
```

### **参数解析**

| 基础配置 |
| :------------------------------- | :-------------------------------------------- |
| `-Dfile.encoding=utf-8`          | 文件编码设置为`UTF-8`                        |
| `-Djava.awt.headless=true`       | 无头模式，服务端不需要图形界面                   |
| `-Djava.net.preferIPv4Stack=true`| 优先使用`IPv4`                                  |
| `-server`                        | 使用`Server`模式`JVM`（针对长时间运行的服务优化） |

| 内存配置 |
| :------------------------------- | :-------------------------------------------- |
| `-Xms2g` `-Xmx2g`           | 堆内存固定`2GB`（初始=最大，避免运行时扩容） |
| `-XX:MetaspaceSize=128m`    | 元空间初始大小`128MB `                     |
| `-XX:MaxMetaspaceSize=256m` | 元空间最大`256MB`                          |

| `G1`垃圾收集器配置 |
| :------------------------------- | :-------------------------------------------- |
| `-XX:+UseG1GC`                | 使用`G1`垃圾收集器                     |
| `-XX:MaxGCPauseMillis=100`    | 目标最大`GC`停顿时间`100ms`            |
| `-XX:G1HeapRegionSize=4M`     | `G1 Region`大小`4MB`                  |
| `-XX:ParallelGCThreads=2`     | 并行`GC`线程数`2`                      |
| `-XX:+UseStringDeduplication` | 开启字符串去重（减少重复字符串内存占用）  |

| `JIT`编译配置 |
| :------------------------------- | :-------------------------------------------- |
| `-XX:CICompilerCount=4` | `JIT`编译器线程数`4` |

| 故障诊断配置 |
| :------------------------------- | :-------------------------------------------- |
| `-XX:+HeapDumpOnOutOfMemoryError` | `OOM`时自动生成堆转储                   |
| `-XX:HeapDumpPath`=...            | 堆转储文件路径                          |
| `-XX:ErrorFile=`...               | `JVM`崩溃日志路径（`%p`会替换为进程`ID`）|

| `JMX`远程监控 | 通过`JMX`端口`50000`连接`VisualVM/JConsole`实时监控 |
| :------------------------------- | :-------------------------------------------- |
| `-Dcom.sun.management.jmxremote.port=50000`         | `JMX`端口`50000`     |
| `-Dcom.sun.management.jmxremote.rmi.port=50000`     | `RMI` 端口同为`50000` |
| `-Dcom.sun.management.jmxremote.ssl=false`          | 不使用`SSL`         |
| `-Dcom.sun.management.jmxremote.authenticate=false` | 不需要认证         |

>`JMX`安全风险: `ssl=false + authenticate=false`，意味着任何人都可以连接`JMX`端口进行监控甚至执行操作，生产环境建议开启认证。<br/>
监控建议：也可以加上`GC`日志参数，来观察实际触发情况。<br/>
`-XX:+PrintGCDetails`（打印详细`GC`信息）<br/>
`-XX:+PrintGCDateStamps`（`GC`日志带时间戳）<br/>
`-Xloggc:/xxx/logs/gc.log`（`GC`日志输出路径）
{: .prompt-tip }

### **`G1`内存分布**

监控大盘显示，年轻代（`Young Generation`）在疯狂`gc`，我们可以根据`JVM`参数，估算下 `G1`内存分布。

从显示配置的参数中，可以知道，
- 堆大小: `-Xms2g -Xmx2g = 2GB`
- `Region`大小: `-XX:G1HeapRegionSize=4M`

`G1GC`没有固定的`Eden`区大小，它是动态调整的，但我们可以估算其初始默认值和动态变化范围，
1. `Young Generation`默认占比：`G1`默认`Young Gen`占堆的`5%-60%`，初始约`5%`;
2. `Eden`与`Survivor`比例：默认`-XX:SurvivorRatio=8`，即`Eden:S0:S1=8:1:1`。

| `G1`内存区域估算 |
| :------------- | :------------- |
| 堆总大小                | `2048 MB`     |
| `Young Gen`初始 (`5%`)  | `~102 MB`     |
| `Young Gen`最大 (`60%`) | `~1228 MB`   |
| `Eden`占`Young Gen`    | `8/10 = 80%` |
| `Eden`初始大小         | `~82 MB`      |
| `Eden`最大大小         | `~983 MB`     |

综上可知，`Young Gen`区在此配置下大约在`102MB ~ 1228MB`之间动态变化，`Eden`区在此配置下大约在`82MB ~ 983MB`之间动态变化，初始约`82MB`左右。`G1GC`会根据`-XX:MaxGCPauseMillis=100`的目标暂停时间自动调整`Young Gen`和`Eden`的大小。由于`G1`是动态调整的，我们也可以通过以下方式查看实际值。
```markdown
# 添加 GC 日志参数
-Xlog:gc*:file=/path/to/gc.log

# 或者使用 jmap 查看
jmap -heap <pid>
```

### **这个配置下的`GC`触发时机**

根据这个配置（`G1GC` + `2GB` 堆 + `4M Region`），`GC`会在以下情况被触发。

| `Young GC` |
| 最频繁 | 频率: 取决于对象分配速率，通常几秒到几十秒一次 |
| 触发条件          | 说明      |
| :--------------- | :----------------------------------------------------- |
| `Eden`区满    | 当`Eden`区分配满时触发，`G1`会动态调整`Eden`大小            |
| 目标停顿时间   | `G1`根据`MaxGCPauseMillis=100`预测并调整回收的`Region`数量 |

| `Mixed GC` |
| `Young` + 部分`Old` |
| 触发条件          | 说明      |
| :--------------- | :----------------------------------------------------- |
| 堆占用超过阈值 | 默认`InitiatingHeapOccupancyPercent=45%`，即堆使用超过`~920MB`时启动并发标记 |
| 并发标记完成后 | 接下来的`Young GC`会变成`Mixed GC`，回收部分老年代`Region`                   |

| `Full GC` | 
| 最慢，应避免 |
| 触发条件          | 说明      |
| :--------------- | :----------------------------------------------------- |
| 晋升失败          | 老年代空间不足以容纳从年轻代晋升的对象                    |
| 并发标记期间堆耗尽 | 标记速度跟不上分配速度                                    |
| 大对象分配失败    | 无法找到足够的连续`Region`（大于`2MB`的对象，即`Region/2`） |
| `Metaspace`不足   | 元空间超过`256MB`                                        |

>关键配置
- 堆大小: `2GB`
- `Region`大小: `4MB`
- `Region`数量: `2048MB / 4MB`=`512`个
- 大对象阈值: `4MB / 2`=`2MB`（超过此大小直接进老年代）
- 触发并发标记: 堆占用`~45%`≈`920MB`
{: .prompt-tip }

## **JVM调优**

### **问题分析**

结合上面`JVM`参数和监控大盘的表现，来分析下导致告警的原因，以及接下来的优化方向。

| 存在问题 |
| :------------------------------- | :-------------------------------------------- |
| `2GB`堆内存               | 对服务日常业务场景够用，但如果有大量并发任务或数据处理，可能需要增加（成本允许的情况下）。 |
| `G1`在小堆上收益低         | `G1`适合`6G+`大堆，`2G`堆开销反而大                       |
| `ParallelGCThreads=2`      | 数值偏低，已经限制了并行度，`G1`优势发挥不出来。但对于`2C`的服务器来说，不能再高了，根本在于资源受限。如果服务器`CPU`核心数较多，这个值会限制`GC`效率。通常建议设为`CPU`核心数的`1/4`到`1/2`。 |
| `UseStringDeduplication`   | 需要额外`CPU`做字符串去重，`2C`下不划算，`CPU`资源比较重要   |
| 短生命周期对象多          | 适当加大`Young`区，减少`Young GC`频率更有效              |
| `MaxGCPauseMillis=100` | 对于后台任务的服务来说，是合理的，不需要特别低的延迟。 |

>整体来看，上面的`JVM`参数是一个针对小型后台任务服务的配置，内存适中，`GC`配置保守，适合资源受限的环境。
但**对于2C4G且大量短生命周期对象的场景，CMS更合适，新生代可以相对给大点。**
{: .prompt-tip }

### **CMS + ParNew**

修改对应`CMS GC`的`JVM`参数，如下，仅供参考，
```terminal
$ ps -ef | grep java
/usr/local/jdk1.8.0_192/bin/java \
  -Dfile.encoding=utf-8 \
  -Djava.awt.headless=true \
  -Djava.net.preferIPv4Stack=true \
  -server \
  -Xms2g -Xmx2g \
  -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m \
  -XX:CICompilerCount=4 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/xxx/logs/xxx_logs/dump \
  -XX:ErrorFile=/xxx/logs/jvm_error_file_logs/java/java_error%p.log \
  -XX:+UseConcMarkSweepGC \
  -XX:+UseParNewGC \
  -XX:NewRatio=1 \
  -XX:SurvivorRatio=8 \
  -XX:MaxTenuringThreshold=6 \
  -XX:CMSInitiatingOccupancyFraction=75 \
  -XX:+UseCMSInitiatingOccupancyOnly \
  -XX:+CMSParallelRemarkEnabled \
  -XX:+CMSScavengeBeforeRemark \
  -XX:+UseCMSCompactAtFullCollection \
  -XX:ParallelGCThreads=2 \
  -Dcom.sun.management.jmxremote.port=50000 \
  -Dcom.sun.management.jmxremote.rmi.port=50000 \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Dcom.sun.management.jmxremote.authenticate=false \
  com.xxx.Main
```

### **参数解析**

| 参数说明 |
| :------------------------------- | :----------------------- |:-------------------------------------- |
| `‌-XX:+UseConcMarkSweepGC`               | -                    | 启用并发标记清除 (`CMS`)‌ 垃圾收集器       |
| `-XX:+UseParNewGC`                      | -                    | 启用并行新生代收集器 (`ParNew`)‌ 与`CMS`配合使用  |
| `-XX:NewRatio=1`                        | `Young:Old = 1:1`    | `Young`区 = `1G`，比默认大很多，可选         |
| `-XX:SurvivorRatio=8`                   | `Eden:S0:S1 = 8:1:1` | `Eden ≈ 800MB`，可选                       |
| ‌‌`-XX:+UseCMSInitiatingOccupancyOnly`    | -                    | ‌只使用`CMSInitiatingOccupancyFraction`设置的阈值‌来触发`CMS`回收，而不是使用默认的自适应策略  |
| `-XX:MaxTenuringThreshold=6`            | `6`次                | 短生命对象不容易晋升到老年代               |
| `-XX:CMSInitiatingOccupancyFraction=75` | `75%`                | 老年代占用`75%`时启动`CMS`                 |
| `‌-XX:+CMSParallelRemarkEnabled`         | -                    | 启用`CMS`并行标记阶段的重新标记‌，提高`CMS`回收的效率  |
| `-XX:+CMSScavengeBeforeRemark`          | -                    | `Remark`前先做次`Young GC`，减少重标记耗时  |
| `-XX:+UseCMSCompactAtFullCollection`    | -                    | `Full GC`时对老年代进行压缩整理，减少碎片   |

`CMS`垃圾收集器，可以减少垃圾回收时的停顿时间，适合对响应时间要求较高的应用。

>效果预期
- `Eden`区: 从`~82MB`提升到`~800MB`
- `Young GC`频率: 大幅降低
- `CPU`消耗: 去掉`StringDeduplication`后更低
{: .prompt-tip }

| 潜在问题及应对方案 |
| :----------------------- | :----------------------- |
| `CMS`碎片化导致`Full GC`  | 老年代用的不多，问题不大；真出现这种问题，线上服务可加`-XX:+UseCMSCompactAtFullCollection`，对老年代做压缩整理，减少碎片 |
| `Young`区太大单次`GC`变长 | 监控`Young GC`时间，如果超过`100ms`，可调小`NewRatio`，或直接使用默认值（MewRatio=2） |
| `CMS`在`JDK 9+`被废弃     | `JDK 8`没问题，暂不用担心  |

>`-XX:NewRatio=1 -XX:SurvivorRatio=8`，也可以先不用指定，一般使用默认值就行。
{: .prompt-tip }

这两个参数在`JDK 8`中的默认值如下，

| 参数              | 默认值     | 说明                                        |
| :---------- ----- | :-------- |:------------------------------------------- |
| `-XX:NewRatio`      | `2`         | 新生代与老年代的比例为`1:2`，即新生代占堆的`1/3` |
| `-XX:SurvivorRatio` | `8`         | `Eden`区与`Survivor`区的比例为`8:1:1`          |

使用默认值时，堆内存分布如下，（以`2G`堆为例）
- 总堆: `2G`
- 新生代:` ~682M` (`1/3`)
    - `Eden`: `~546M` (`8/10 of` 新生代)
    - `S0`: `~68M` (`1/10 of` 新生代)
    - `S1`: `~68M` (`1/10 of` 新生代)
- 老年代: `~1365M `(`2/3`)

如果不使用默认值，用当前配置，那么
- `-XX:NewRatio=1`，新生代占 `1/2`（比默认更大）
- `-XX:SurvivorRatio=8`，与默认相同

>相比之下，使用默认值，新生代会变小（`1/3 vs 1/2`），对于大多数应用，默认值是合理的，毕竟`Young`区太大，单次`GC`就会变长。
如果你的应用对象存活时间短、创建频繁（如`Web`服务、短任务处理），默认的`NewRatio=2`通常就够用了。
如果发现`Young GC`频繁或晋升过快，可以再考虑调大新生代。
{: .prompt-tip }

## **上线观察**

优化上线后的业务流量激增，增幅比之前还稍大点，

![Desktop View](/assets/img/20260328/biz_qps_surge_02.png){: width="300" height="150" }
_业务请求量激增（优化后）_

| 机器负载 |
| 明显下降的关键指标 |
| `CPU`、内存 |
| 一分钟单核负载指标 |
| :------------- | :------------- | :------------- |
| ![Desktop View](/assets/img/20260328/machine_cpu_02.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_memory_02.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_core_per_minute_02.png){: width="500" height="300" } |
| ![Desktop View](/assets/img/20260328/machine_disk_rw_02.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_disk_space_02.png){: width="500" height="300" } | ![Desktop View](/assets/img/20260328/machine_network_02.png){: width="500" height="300" } |

| 看图说话：新生代使用量降下来了，同时`Young GC`对`CPU`资源没那么敏感了 |
| 不会导致`CPU`过载导致的短期内晋升老年代太多的问题，也就不会频繁`FullGC`，服务平稳运行 |
| :-------------------------- |
| ![Desktop View](/assets/img/20260328/CMS_gc_base_info_local_02.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/CMS_gc_base_info_02_local_02.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/CMS_gc_collector_info_local_02.png){: width="800" height="600" } |

| 再看看，优化上线前后的`GC`指标对比，效果立马显现 |
| :-------------------------- |
| ![Desktop View](/assets/img/20260328/G1_gc_base_info_global_02.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/G1_gc_base_info_02_global_02.png){: width="800" height="600" } |
| ![Desktop View](/assets/img/20260328/G1_gc_collector_info_global_02.png){: width="800" height="600" } |

## End 🌈🌈🌈
生活依然那么美好，可以继续撸代码了。
