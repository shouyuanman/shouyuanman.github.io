---
title: OnlineDDL（在 MySQL 5.7 数据库里，InnoDB引擎，执行一条DDL会发生什么事情）
date: 2024-12-08 23:00:10 +0800
categories: [后端, MySQL]
tags: [后端, MySQL]
music-id: 187908
---

## Case

`MySQL`数据库用的大版本是`5.7`，`InnoDB`引擎， 

![Desktop View](/assets/img/20241208/onlineddl_mysql_version.jpg){: width="500" height="300" }
_MySQL数据库版本_

>想个问题，<br/>在 MySQL 5.7 数据库里，InnoDB引擎，执行⼀条DDL会发生什么事情？<br/>举个例子，表记录数 400W+，表字段数 20+，索引4个，数据大小 1.7G，配置4核16G，主从复制，读写分离。<br/>
![Desktop View](/assets/img/20241208/onlineddl_table_count.jpg){: width="500" height="300" }
_MySQL table count统计_
![Desktop View](/assets/img/20241208/onlineddl_mysql_table_fields.jpg){: width="500" height="300" }
_MySQL table 字段统计_
![Desktop View](/assets/img/20241208/onlineddl_mysql_table_indexes.jpg){: width="500" height="300" }
_MySQL table 索引统计_
在测试环境下给上面的表执行下面的DDL（给表新增⼀个字段），
```markdown
alter table t_bussiness_table add new_field datetime NOT NULL DEFAULT '1900-01-01' COMMENT '新字段注释' after last_field; 
```
{: .prompt-tip }

MySQL 5.7 会使用到`MySQL`原生的`OnlineDDL`，整个过程涉及下面不同的`SQL`阶段， 

| SQL执行阶段                                | 说明          | 备注 |
| :-----------------------------------------| :--------------- | ------: |
| starting                                  | 通常指的是从接收SQL语句到准备执行计划（比如解析查询树、生成执行计划、打开表等）的时间     |  |
| checking permissions                      | 记录了MySQL检查用户权限的过程    |       |
| init          | 通常指的是在查询执⾏前的准备阶段，包括解析SQL语句、优化查询计划、生成执行计划、打开表和索引、读取和解析数据字典等 |    |
| Opening tables                            | 通常表示MySQL在查询执行过程中为表打开文件并准备读取数据的时间     |  |
| setup         | 通常指的是从存储引擎获取和解析查询的初始化阶段，这个阶段包括了解析查询、生成执行计划、打开表、取得行锁等操作    |       |
| Creating table                            | 创建表 (Creating table) 阶段的信息 |    |
| After create                              | 通常指的是创建和开始会话的时间，这个时间点发⽣在SQL语句执行之前     |  |
| System lock                               | 通常表示获取系统锁的时间，数据库管理系统用来控制并发操作的⼀种机制 当多个事务同时尝试修改同⼀数据时，为了避免数据不⼀致，数据库系统会实施某种形式的锁定    |       |
| <font color='orange'>Preparing for alter table</font> | 主要是指MySQL在执行ALTER TABLE操作前的准备⼯作，包括检查权限、分析和 预处理语句、⽣成新的表结构等 |    |
| <font color="red">Altering table</font>   | 服务器就地执⾏，修改表结构，包括改变列的数据类型、添加、删除或重命名 列，以及添加或删除索引等     |  |
| <font color='orange'>Committing alter table to storage engine</font> | 表示MySQL已经准备好了所有的更改，并准备将这些更改提交到存储引擎中    |       |
| end                                       | 主要涉及将数据发送到客⼾端的活动，包括将数据从MySQL服务器的缓冲区传输 到客⼾端的⽹络缓冲区中 |    |
| query end                                 | 表⽰查询的所有阶段都已经完成，并且所有的结果行都已经发送到客⼾端     |  |
| waiting for semi- sync ack                | 阶段通常与MySQL的半同步复制机制有关在等待来自从库的应答，表⽰事务的数据已经被安全地记录    |       |
| Closing tables                            | 表示MySQL在执行查询之后关闭表的时间<br/>这个阶段通常很短，并且通常是在执⾏查询的最后阶段，即在结果集被获取和返回之后 |    |
| freeing items                             | 通常指的是在查询执行完毕后，为查询结果的行、列定义等资源进行清理和释放的过程     |  |
| Cleaning up                               | 通常指的是查询执行完毕后的清理工作，比如释放临时表、清理会话状态等。这 个阶段的耗时反映了MySQL在查询结束后进⾏内部清理⼯作所需的时间    |       |

## Online DDL

>不同 MySQL 版本，执行DDL的表现都不⼀样。同⼀MySQL版本，不同DDL的表现也不⼀样。 
{: .prompt-tip }

### 背景

简单介绍下OnlineDDL的背景， 

`MySQL Online DDL`功能从`5.6`版本开始正式引入，发展到现在的`8.0`版本，经历了多次的调整和完善。其实早在`MySQL 5.5`版本中就加入了`INPLACE DDL`方式，但是因为实现的问题，依然会阻塞`INSERT、UPDATE、DELETE`操作，这也是 MySQL 早期版本长期被吐槽的原因之⼀。 

在`MySQL 5.6`版本以前，最昂贵的数据库操作之⼀就是执行`DDL`语句，特别是`ALTER`语句，因为在修改表时，`MySQL`会阻塞整个表的读写操作。对于那些要提供全天候服务（`24*7`）或维护时间有限的⼈来说，在大表上执行`DDL`⽆疑是⼀场真正的噩梦。 

`MySQL`官方不断对`DDL`语句进行增强，自`MySQL 5.6`起，开始支持更多的`ALTER TABLE`类型操作来避免数据拷贝，同时支持了在线上`DDL`的过程中不阻塞`DML`操作，真正意义上的实现了`Online DDL`， 即在执行`DDL`期间允许在不中断数据库服务的情况下执行`DML`(`insert、update、delete`)。然而**并不是所有的`DDL`操作都支持在线操作**。到了 `MySQL 5.7`，在`5.6`的基础上又增加了⼀些新的特性，比如，增加了重命名索引支持，支持了数值类型长度的增大和减小，⽀持了`VARCHAR`类型的在线增大等。但是基本的实现逻辑和限制条件相比`5.6`并没有大的变化。 

`MySQL 5.7`中，不同`DDL`操作的具体支持程度， 

![Desktop View](/assets/img/20241208/onlineddl_support_column_operations.jpg){: width="500" height="300" }
_MySQL 5.7 中，不同DDL操作的具体支持程度_

>参考`MySQL5.7`官方文档<br/>
[14.13 InnoDB and Online DDL](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl.html)<br/>
[14.13.1 Online DDL Operations](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html#online-ddl-column-operations)
{: .prompt-tip }

### 原理

>Online DDL主要包括3个阶段，prepare阶段，ddl执行阶段，commit阶段<br/>看看这三个阶段都做了什么事情
![Desktop View](/assets/img/20241208/onlineddl_mysql_prepare_alter_commit.jpg){: width="750" height="450" }
_prepare阶段，ddl执行阶段，commit阶段_
{: .prompt-tip }

| Adding a column | DDL阶段  | 说明          | 加锁情况 |
|                 | prepare  | 初始化阶段会根据存储引擎、⽤⼾指定的操作、计算 DDL 过程中允许的并发量，这个过程中会获取⼀个 MDL写锁，用来保护 表的结构定义。| MDL写锁，禁止DML |
|                 | ddl执行  | 重建表，执行期间MDL写锁降级为MDL读锁，保证了不会同时 执行其他的 DDL，但 DML 能可以正常执行。| MDL读锁，允许并发DML |
|                 | commit   | 将 MDL读锁 升级为 MDL 写锁，禁止DML，然后删除旧的表定义，提交新的表定义。| MDL写锁，禁止DML |

**Online DDL 过程中占用 MDL写锁 的步骤执行很快，几乎不会阻塞 DML 语句，可以认为1秒内级别锁表。**

- 引入了`Online DDL`之后，重建表的流程： 
```markdown
建立⼀个临时⽂件，扫描表A主键的所有数据页； 
用数据页中表A的记录⽣成B+树，存储到临时文件中； 
生成临时⽂件的过程中，将所有对A的操作记录在⼀个日志文件（row log）中； 
临时文件生成后，将日志文件中的操作应用到临时文件，得到⼀个逻辑数据上与表A相同的数据文件； 
用临时文件替换表A的数据文件。
```

- 解读
```markdown
在重建表的过程中，允许对表A做增删改操作。这也就是Online DDL名字的来源。 
对于一个大表来说，Online DDL最耗时的过程就是拷贝数据到临时表的过程，这个步骤的执行期间可以接受增删改操作。所以，相对于整个DDL过程来说，锁的时间非常短。对业务来说，就可以认为是Online的。 
这些重建方法都会扫描原表数据和构建临时文件。对于很大的表来说，这个操作是很消耗IO和CPU资源的。 
```

### 经验总结
```markdown
优先使用开源 DDL ⼯具，平滑无锁修改表结构； 
条件不允许的话，如果使用MySQL原声的OnlineDDL，⼀定要在业务低峰执行。对于很大的表来说，这个操作是很消耗IO和CPU资源的。如果是线上服务，要很小心地控制操作时间； 
执行DDL前，要结合表的大小，谨慎操作； 
避免程序中存在长事务，事务不提交，就会⼀直占着MDL锁，⻓事务会阻塞OnlineDDLMDL读锁。 
```

### Case结论 
上面这个case，整个`DDL`执行流程，耗时50+s，`MDL`写锁90ms左右（可以认为是秒级锁表，这个时间会根据执行时的服务器情况和硬件情况，上下有浮动）

![Desktop View](/assets/img/20241208/onlineddl_mysql_phases.jpg){: width="500" height="300" }
_onlineddl_case_phases_

**系统资源使用情况**

![Desktop View](/assets/img/20241208/onlineddl_mysql_cpu_quota.jpg){: width="500" height="300" }
_cpu quota_

![Desktop View](/assets/img/20241208/onlineddl_mysql_disk_quota.jpg){: width="500" height="300" }
_disk quota_

![Desktop View](/assets/img/20241208/onlineddl_mysql_disk_tmp_table_quota.jpg){: width="500" height="300" }
_disk tmp table quota_

**彩蛋**

目前可用的`DDL`操作工具包括`pt-osc`，`github`的`gh-ost`，以及`MySQL`提供的在线修改表结构命令`Online DDL`。`pt-osc`和`gh-ost`均采用拷表方式实现，即创建个空的新表，通过`select+insert`将旧表中 的记录逐次读取并插入到新表中，不同之处在于处理`DDL`期间业务对表的`DML`操作。 

到了`MySQL 8.0`官⽅也对`DDL`的实现重新进行了设计，其中⼀个最大的改进是`DDL`操作⽀持了原⼦特性。另外，`Online DDL`的`ALGORITHM`参数增加了⼀个新的选项：`INSTANT`，只需修改数据字典中的元数据，无需拷贝数据也无需重建表，同样也⽆需加排他`MDL`锁，原表数据也不受影响。整个`DDL`过 程几乎是瞬间完成的，也不会阻塞`DML`，不过⽬前8.0的`INSTANT`使⽤范围较小，后续再对8.0的`INSTANT`做详细介绍吧。 
