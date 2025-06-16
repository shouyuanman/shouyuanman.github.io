---
title: ID生成器（简单实用型，适用于小规模应用）
date: 2025-06-12 17:18:00 +0800
categories: [后端, ID生成器]
tags: [后端, ID生成器]
music-id: 2703621578
---

## **序言**

一款好用、可靠的`ID`生成器，是业务稳定运行的基石。

## **场景**
假如你的业务系统是微服务分布式架构，但数据库是**单库单表**，没有分库分表就意味着服务单库并发不高，且单库数据量也不大。

**业务需求**：在单库单表场景下，生成并使用**全局唯一**`ID`（比如可用作短链`ID`、订单`ID`、交易单`ID`、消息流水`ID`、调查问卷`ID`、用户`ID`、聊天会话`ID`等等），要求`ID`**业务可读**，**`Long`整型且按天相对递增**（不需要绝对的单调递增），‌能**防爬取/猜测**。

## **思路**
服务自己闭环，将数据库自增主键转换为短且无规律的公开`ID`，用算法隐藏业务数据的递增特性，防止被遍历或猜测。前面加上业务`code`和日期相关的因子，就能达到业务可读、按天相对递增的效果。一般业务服务都会用到`MySQL`数据库，生成`ID`时也用`MySQL`数据库，这样就意味着没有引入更多的组件依赖。

>如果服务单库并发不高，但因为单库数据量过大导致了分库分表，理论上可以结合分库分表规则用这个方案来生成全局唯一`ID`，但不建议这么用。如果服务并发过高，此方案就不能用了，需另辟蹊径。<br/>
无论用什么解决方案，都需要根据具体的业务需求和系统规模进行评估和选择。在设计和实现过程中，需要考虑`ID`的唯一性、性能、可用性、扩展性以及与分库分表规则的结合等因素，确保生成的`ID`满足业务需求并能够良好地支持分库分表环境。
{: .prompt-tip }

## **优缺点分析**
### **优点**
1. 简单易用，适合简单任务场景；
2. 对小规模应用，性能足够好。

### **缺点**
1. 高并发场景下，可能会遇到性能瓶颈；
2. 高度依赖`MySQL`，使用时要对`MySQL`不可用问题做好兜底。

## **代码实现**
### **创建 `t_auto_no` 表**
创建表，表的`Dao`层`Logic`代码（`AutoNoDaoLogic`）这里省略，这张表里只有两个字段（这里是最简数据结构，具体表结构可以根据所在团队的表创建规范来定），每次生成一个`ID`，都往这个表里插入一条没什么业务含义的数据，然后获取一个数据库自增`ID`，如下，
```sql
CREATE TABLE `t_auto_no` (
`id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='business_name|自增记录表|owner|20250612'
```
### **定义业务枚举和id格式**
定义业务枚举，让生成的`ID`业务可读，生成的`ID`格式可以根据自己的习惯定义，这里的格式暂且定义为“业务`Code`（手动分配`2`位）+ `yymmdd`（`6`位）+ 经过自增`ID`转换后的`n`位长整型数（`n`可以自己指定）”如下，
```java
// SystemEnum.java
import lombok.AllArgsConstructor;
import lombok.Getter;
/**
 * @author shouyuanman
 * @date 2025/6/12
 * @desc
 */
@AllArgsConstructor
@Getter
public enum SystemEnum {
    /**
     * 系统枚举
     */
    BUSINESS_XXX_10("business_xxx_10", 10, "业务10"),
    BUSINESS_XXX_11("business_xxx_11", 11, "业务11"),
    BUSINESS_XXX_12("business_xxx_12", 12, "业务12"),
    UNKNOWN("", 99, "未知"),
    ;
    public String appId;
    public int systemCode;
    public String systemDesc;
    public static SystemEnum fromAppId(String appId) {
        for (SystemEnum systemEnum : SystemEnum.values()) {
            if (systemEnum.appId.equalsIgnoreCase(appId)) {
                return systemEnum;
            }
        }
        return SystemEnum.UNKNOWN;
    }
}

// AutoNoBiz.java
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.RandomUtils;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;
import java.util.Date;
/**
 * @author shouyuanman
 * @date 2025/6/12
 * @desc
 */
@Service
@Slf4j
public class AutoNoBiz {
    @Resource
    private AutoNoDaoLogic autoNoDaoLogic;
    /**
     * 获取BusinessXxx10 id
     * @return BusinessXxx10 id
     */
    public Long genBusinessXxx10No() {
        return genCustomNo(SystemEnum.BUSINESS_XXX_10);
    }
    
    /**
     * 获取BusinessXxx11 id
     * @return BusinessXxx11 id
     */
    public Long genBusinessXxx11No() {
        return genCustomNo(SystemEnum.BUSINESS_XXX_11);
    }
    
    /**
     * 获取BusinessXxx12 id
     * @return BusinessXxx12 id
     */
    public Long genBusinessXxx12No() {
        return genCustomNo(SystemEnum.BUSINESS_XXX_12);
    }
    
    /**
     * 根据数据库自增id生成自定义id
     * @param systemEnum 系统
     * @return 自定义id
     */
    public Long genCustomNo(SystemEnum systemEnum) {
        try {
            AutoNo autoNo = new AutoNo();
            autoNo.setCreateTime(new Date());
            autoNoDaoLogic.insert(autoNo);
            Long id = autoNo.getId();
            // public final static String DATE_PATTERN_SHORT = "yyMMdd";
            return Long.parseLong(String.format("%s%s%s", systemEnum.systemCode,
                    DateUtils.getCurDateStr(DateUtils.DATE_PATTERN_SHORT), IdUtils.genId(id, 10)));
        } catch (Exception e) {
            log.warn("[AutoNoBiz] auto no difuse, exec fail, e=>{}", e.getMessage(), e);
            // 生成id失败时的简单兜底
            return Long.parseLong(String.valueOf(systemEnum.systemCode) +
                    RandomUtils.nextLong(1000000000000000L, 9000000000000000L));
        }
    }
}
```

## **ID生成算法实现**

### **贴代码**
```java
// IdUtils.java
/**
 * @author shouyuanman
 * @date 2025/6/12
 * @desc
 */
public class IdUtils {
    private static final int[] WIDTH_4 = { 11, 2, 10, 3, 1, 0, 7, 9, 12, 6, 4, 5, 8 };
    private static final int[] WIDTH_5 = { 4, 3, 13, 15, 7, 8, 6, 2, 1, 10, 5, 12, 0, 11, 14, 9 };
    private static final int[] WIDTH_6 = { 2, 7, 10, 9, 16, 3, 6, 8, 0, 4, 1, 12, 11, 13, 18, 5, 15, 17, 14 };
    private static final int[] WIDTH_7 = { 18, 0, 2, 22, 8, 3, 1, 14, 17, 12, 4, 19, 11, 9, 13, 5, 6, 15, 10, 16, 20, 7, 21 };
    private static final int[] WIDTH_8 = { 11, 8, 4, 0, 16, 14, 22, 7, 3, 5, 13, 18, 24, 25, 23, 10, 1, 12, 6, 21, 17, 2, 15, 9, 19, 20 };
    private static final int[] WIDTH_9 = { 24, 23, 27, 3, 9, 16, 25, 13, 28, 12, 0, 4, 10, 18, 11, 2, 17, 1, 21, 26, 5, 15, 7, 20, 22, 14, 19, 6, 8 };
    private static final int[] WIDTH_10 = { 3, 32, 1, 28, 18, 21, 12, 7, 30, 22, 20, 13, 16,15, 6, 17, 9, 25, 11, 8, 4, 14, 27, 31, 5, 23, 24, 29, 10, 0, 19, 2, 26 };
    private static final int[] WIDTH_11 = { 18, 2, 13, 29, 11, 32, 14, 33, 24, 8, 27, 4, 22, 20, 5, 0, 21, 25, 17, 28, 34, 6, 23, 26, 30, 7, 3, 19, 16, 15, 12, 31, 1, 35, 10, 9 };
    private static final int[] WIDTH_12 = { 31, 1, 33, 16, 35, 29, 17, 37, 12, 28, 32, 22, 6, 10, 14, 26, 0, 9, 8, 3, 20, 2, 13, 5, 36, 27, 23, 15, 19, 34, 38, 11, 24, 25, 30, 21, 18, 7, 4 };
    private static final int[][] WIDTH_ARRAY = { WIDTH_4, WIDTH_5, WIDTH_6, WIDTH_7, WIDTH_8, WIDTH_9, WIDTH_10, WIDTH_11, WIDTH_12 };
    public static long genId(long id, int width) {
        long maxValue = (long) Math.pow(10, width) - 1;
        int superNum = (int) (Math.log(maxValue) / Math.log(2));
        long r = 0;
        long sign = (long) Math.pow(2, superNum);
        id |= sign;
        int[] mapBit = WIDTH_ARRAY[width - 4];
        for (int x = 0; x < superNum; x++) {
            long v = (id >> x) & 0x1;
            r |= (v << mapBit[x]);
        }
        r += maxValue - Math.pow(2, superNum) + 1;
        return r;
    }
}
```

### **代码分析**
`IdUtils`类实现了一种基于位操作的`ID`生成算法，将输入的数值`id`转换为指定宽度（`width`）的、具有非连续特性的长整型`ID`。以下是核心分析，

1. ‌代码结构与关键变量‌
    - ‌静态数组‌：`WIDTH_4`到`WIDTH_12`定义了不同宽度下的位映射规则，每个数组表示原始位到目标位的映射关系。例如，`WIDTH_4`表示宽度为`4`时的位重排规则。
    - 宽度映射‌：`WIDTH_ARRAY`将不同宽度（`4`-`12`）映射到对应的位规则数组。
    - ‌方法逻辑‌：`genId`方法接受原始`id`和目标`width`，通过位操作生成最终`ID`。

2. ‌核心逻辑‌
    - ‌步骤`1`：计算关键值‌
        - `‌maxValue`‌：`10^width - 1`，表示目标宽度的最大值（如`width=4`时，`maxValue=9999`）。
        - `‌superNum`‌：`log2(maxValue)`，计算覆盖`maxValue`所需的最小二进制位数（如`maxValue=9999`时，`‌superNum=13`，因为`2^13=8192`）。
        - `‌sign`‌：`2^‌superNum`，用于设置`id`的最高位，确保后续位操作不越界。
    - ‌步骤`2`：位重排‌
        - `‌id |= sign`‌：将`id`的二进制最高位设为`1`，确保其长度至少为`‌superNum + 1`位。
        - 位映射‌：根据`width`选择对应的位映射规则（如`WIDTH_4`），将原始`id`的每一位按映射规则重新排列到目标位置，生成中间值`r`。
        - 循环中，`x`从`0`到`12`，每次取出`id`的第`x`位，然后移动到`mapBit[x]`的位置。比如，如果`mapBit[0]=10`，那么`id`的第`0`位会被放到`r`的第`10`位。这样，`r`的各个位是根据`mapBit`数组重新排列后的结果。
    - 步骤`3`：数值平移‌
        - 调整范围‌：将`r`的值平移到`[maxValue - 2^‌superNum + 1, maxValue]`区间，确保最终`ID`符合目标宽度范围。例如，`width=4`时，`r`的范围从`[0, 8191]`平移到`[1808, 9999]`。如果`r`是经过位重排后的值，可能范围是`0`到`8191`，加上`1808`后变成`1808`到`9999`，刚好覆盖了`9999`的范围。
    
3. ‌设计意图‌
    - ‌唯一性‌：通过位重排（唯一映射规则）确保不同`id`生成不同结果。
    - 非连续性‌：位重排破坏了原始`id`的递增特性，生成的`ID`在视觉上无规律。
    - ‌范围控制‌：通过平移操作使结果始终落在`10^width`范围内，满足固定位数的需求。

>该代码运算类似一种简单的混淆算法，通过位重排和平移生成固定宽度的唯一`ID`，适用于需要隐藏自增特性的场景，但需严格控制输入范围和映射规则安全性，使用时考虑`ID`容量的问题。<br/>
注意：位重排数组中元素的位置一旦交换，就相当于变了一种混淆规则，可以做到千组千面。
```markdown
示例验证
假设width=4，验证过程如下，
long id = 1234;
int width = 4;
// 计算关键值
long maxValue = 9999;               // 10^4 -1
int superNum = 13;                  // log2(9999) ≈13
long sign = 8192;                   // 2^13
id |= sign;                         // 设置最高位为1（id=1234|8192=9426）
// 位重排（示例：WIDTH_4的映射规则）
int[] mapBit = WIDTH_4;             // [10,2,11,3,0,1,9,7,12,6,4,8,5]
long r = 0;
for (int x=0; x<13; x++) {
    long v = (id >> x) & 0x1;       // 提取第x位
    r |= (v << mapBit[x]);          // 按映射规则重排
}
// 平移操作
r += 1808;                          // maxValue - 2^superNum +1 = 9999 - 8192 +1
return r;                           // 最终ID
```
{: .prompt-tip }

### **QA**

>**Q1**：当`width`较大时，`maxValue`可能会超过`Long`的最大值吗？<br/>
**A1**：这里返回的是`long`型，所以当`width`超过`18`时，`10^19`会超过`Long`的范围，但代码中的`width`参数是从`4`到`12`，所以没问题。
{: .prompt-tip }

>**Q2**：考虑位重排数组的作用？<br/>
**A2**：`WIDTH_4`到`WIDTH_12`的数组长度各不相同，里边存储的数值对应位映射关系。例如，`WIDTH_4`有`13`个元素，对应`13`位的位置。循环中，每个位`x`被映射到`mapBit[x]`的位置，重新排列`id`的位，以达到某种编码或混淆的目的。
{: .prompt-tip }

>**Q3**：位重排数组（比如`WIDTH_4`）的长度和`superNum`之间有什么关联吗？<br/>
**A3**：代码中的数组长度与对应的`width`所需的`superNum`值一致，这样循环时不会越界。比如`WIDTH_4`数组有`13`个元素，`superNum`是`13`的话，循环次数是`13`次，而数组长度刚好是`13`。再看`width=5`时，`maxValue`是`99999`，`log2(99999)≈16.6`，所以`superNum=16`，而`WIDTH_5`的长度是`16`（`0`到`15`）。
{: .prompt-tip }

>**Q4**：位重排数组中的值有什么要求？<br/>
**A4**：数组中不能有重复的值，如果`mapBit`数组中的位置有重复，会导致不同位被映射到同一位置，从而产生冲突。位映射唯一，可以保证不同的`id`生成不同的`r`值。
{: .prompt-tip }

>**Q5**：为什么`width`从`4`到`12`对应到数组索引是`width-4`？<br/>
**A5**：当`width=4`时，用的是`WIDTH_ARRAY` `0` 号位元素，即`WIDTH_4`数组。`WIDTH_ARRAY`的长度是`9`，对应`width`从`4`到`12`，共`9`个元素。
{: .prompt-tip }

>**Q6**：这里的`width`，业务应该怎么选择？<br/>
**A6**：选择合适的`width`，一个是要考虑生成`ID`的长度，另一个很重要的要考虑的因素，是`width`对应的`ID`容量（一天生成多少个不重复的唯一`ID`才能满足业务要求），这也是在`genId`之前增加业务`code`和`yymmdd`两种维度的根本原因。
{: .prompt-tip }

>**Q7**：只考虑位重排算法，会生成重复`ID`吗？<br/>
**A7**：会。上述算法有个特点，输入的`id`必须小于`2^superNum`，否则高位丢失导致`id`重复。当原始`id`超过`sign`时，高位的信息会被忽略，导致不同的`id`生成相同的`r`值，可能导致生成的`r`无法唯一映射原始`id`。举个例子，如果`id1=sign + a`，`id2=sign + b`，其中`a`和`b`在低`superNum`位不同，那么处理后的`r`会不同；但如果`id`超过`sign`的范围，例如`id=sign*2 + a`，此时`id`的高位可能没有被处理，导致生成的`r`与较小的`id`冲突。
{: .prompt-tip }

>**Q8**：我们的实现是怎么做到上述高位缺陷导致`ID`重复但不影响业务的？<br/>
**A8**：具体需求具体分析，在`genId`之前增加业务`code`和`yymmdd`两种维度，将`ID`容量扩充到单业务每天的维度。
{: .prompt-tip }

>**Q9**：`width=10`，单业务每天能容纳生成多少个`ID`？<br/>
**A9**：`width=10`，当数据库自增`id`超过`‌8,589,934,591`‌时，生成的`ID`开始重复‌。<br/>
**根本原因‌：**`width=10` 限制了`ID`容量，而`BIGINT`自增范围远大于生成规则的有效范围。<br/>
**建议措施‌：**升级`width`或优化生成算法，确保与`BIGINT`容量匹配。
```sql
-- ‌示例‌：
-- 假设使用无符号`BIGINT`自增主键：
INSERT INTO table VALUES ();  -- id=8,589,934,591 → 生成有效 ID
INSERT INTO table VALUES ();  -- id=8,589,934,592 → 越界，生成的 ID 与 id=0 冲突
```
**以下是详细分析**，<br/>
当`width=10`时，系统最多可生成`8,589,934,592`个唯一`ID`。<br/>
|`maxValue`|`10^10 - 1`|`9,999,999,999`|<br/>
|`superNum`|`log2(maxValue)`|`33（2^33=8,589,934,592）`|<br/>
|`id `有效范围|`0 ≤ id ≤ 2^33 - 1`|`0 – 8,589,934,591`|<br/>
**‌唯一`ID`总数‌的理论容量‌：**输入的`id`范围是`0 – 8,589,934,591`，共`8,589,934,592`（即`2^33`）个不同值‌。<br/>
**平移后的`ID`范围安全‌：**生成的`ID`严格落在`1,406,545,408 – 9,999,999,999`之间，无溢出风险，占满`maxValue`的`85.9%`空间。<br/>
**`[maxValue - 2^33 + 1, maxValue]` → `[1,406,545,408, 9,999,999,999]`**<br/>
该范围可容纳`8,589,934,592`个值‌（`9,999,999,999 - 1,406,545,408 + 1`），与输入`id`数量一致。‌<br/>
**评估：**若系统每日新增`100`万条数据，约`23.5`年‌后达到越界阈值（`8.59`亿 / `100`万 ≈ `8590`天）。需提前规划扩容。<br/>
**‌建议‌：**若需要更多`ID`，需增大`width`或优化映射规则（如使用更密集的排列）。
- ‌扩大`width`‌：将`width`从`10`调整为更大值（如`width=19`），使生成的`ID`范围覆盖`BIGINT`最大值。
```java
// 示例：width=19 时，maxValue=9999999999999999999（19位）
long maxValue = (long) Math.pow(10, 19) - 1;  // 支持 id ≤ 2^63-1
```
- ‌优化`ID`生成规则‌：使用更密集的位映射（如哈希算法）或分段生成策略，避免高位截断。
- 监控与预警‌：在`id`接近`8,589,934,591`时触发告警，提前扩容或迁移数据。
{: .prompt-tip }

>**‌Q10**：数据库自增`BIGINT`的范围‌？<br/>
**A10**：<br/>
范围（有符号）`‌-9,223,372,036,854,775,808`至`9,223,372,036,854,775,807`<br/>
‌范围（无符号）`‌0`至`18,446,744,073,709,551,615`<br/>
思考：我们的实现是在单库单表下，所有业务共用无符号`BIGINT`范围，理论上这个范围已经很大了，但是对于海量业务也有越界风险，思考下该怎么优化呢？<br/>
提示：可以借鉴美团`Leaf segment`算法。
{: .prompt-tip }

>**Q11**：位重排的数组设计，结合我们的`ID`设计规则，是否能确保唯一性和可逆性？<br/>
**A11**：经过上述分析，我们的id设计规则可以保证，在业务每天`ID`生成量不超过预期的情况下，能保证业务每天`ID`唯一。但做不到可逆，因为`id`越界`sign`时，高位丢失，逆向不可找回。
{: .prompt-tip }

## **分库分表扩展**
如果不加以改造，本文的`ID`生成器实现肯定不适用于分库分表，因为一旦分库分表，当前的数据库自增`ID`方案肯定就会生成重复自增`ID`。如果结合分库分表规则来生成全局唯一`ID`，有两种常见的做法。

### **步长偏移法（不推荐）**

对于固定分片场景，可以使用步长偏移法。

1. ‌定义分片规则‌
设定`N`个数据库分片（或表），每个分片分配唯一‌起始偏移量（`offset`）‌ 和相同‌步长（`step`）‌。

2. ‌配置数据库参数‌

    ```markdown
    修改每个分片的MySQL配置（如 my.cnf）
    # 分片1 配置
    auto_increment_offset = 1    # 起始ID
    auto_increment_increment = 4 # 步长=总分片数（示例N=4）

    # 分片2 配置
    auto_increment_offset = 2
    auto_increment_increment = 4

    # 分片3 配置
    auto_increment_offset = 3
    auto_increment_increment = 4

    ‌ID 生成结果‌
    ‌分片‌	生成ID序列
    分片1	1, 5, 9, 13,...
    分片2	2, 6, 10, 14,...
    分片3	3, 7, 11, 15,...
    分片4	4, 8, 12, 16,...
    ```

3. ‌适用场景与限制‌
    - 优点‌
        - 零代码侵入，纯数据库配置实现。
    - 缺点‌：
        - 分片数量固定，扩容需重新分配偏移量（需停机迁移数据）。
        - 单分片性能受限于数据库自增锁。


### **独立发号表法**

对于动态扩容场景，可以用独立发号表法。

>独立库表自增`ID`（系统每次要生成一个`ID`，获取这个数据库自增`ID`写入对应的分库分表里，比如根据‌分库分表路由‌，生成`ID`后，根据分片键<如用户`ID`哈希>路由到对应分片写入数据）。<br/>
在上面代码实现的基础上，仅仅增加单库生成自增`ID`的策略，在高并发场景下是有瓶颈的，再加上事务问题，性能吞吐量整个比较低，即使用预先生成方案，这样的独立库表要承载每秒几万并发肯定不现实。
{: .prompt-tip }

_<font color="red">可以参考Leaf Segment优化</font>_

简单列下重点步骤（细节下回分解）。

1. 创建全局发号表
```sql
CREATE TABLE id_segment (
id BIGINT NOT NULL AUTO_INCREMENT,
biz_code VARCHAR(32) NOT NULL COMMENT '业务线code',
max_id BIGINT NOT NULL COMMENT '最大id',
step INT NOT NULL COMMENT '步长',
PRIMARY KEY (`id`) USING BTREE,
UNIQUE KEY `uk_biz_code` (`biz_code`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='business_name|自增记录表|owner|20250612';
```

2. 应用层批量获取号段‌

每次从数据库申请一个号段（如`[max_id + 1, max_id + step]`），更新`max_id`,
```sql
UPDATE id_segment SET max_id = max_id + step WHERE biz_code = 'order_id';
```
应用本地缓存该号段，内存中分配`ID`（无需重复访问`DB`）。

3. ‌分库分表路由‌

生成`ID`后，根据分片键（如用户`ID`哈希）路由到对应分片写入数据。

4. ‌优化策略‌
    - ‌双`Buffer`机制‌：异步预加载下一个号段，避免分配完才触发更新。
    - 多业务隔离‌：通过`biz_code`区分不同业务线的`ID`生成（如订单、用户）。

5. ‌适用场景‌
    - 动态扩容分片；
    - 高并发写入（如每秒万级`ID`生成）。
