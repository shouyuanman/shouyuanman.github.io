---
title: MySQL 死锁排查
date: 2024-12-08 21:50:10 +0800
categories: [后端, MySQL]
tags: [后端, MySQL]
music-id: 393703
---

## 现象

> 发生死锁，服务会出现如下报警，<br/>Deadlockfound when trying to get lock; try restarting transaction.
{: .prompt-danger }


## 思路

> 出现类似问题，应先评估死锁对业务和数据的影响范围。如果有损，选择合适的止损方式，然后再去分析解决问题。
{: .prompt-tip }

### 查服务日志和 MySQL 死锁日志，定位死锁发生地

![Desktop View](/assets/img/20241208/deadlock_es_log.png){: width="750" height="450" }
_es死锁日志_

![Desktop View](/assets/img/20241208/deadlock_mysql_log.jpg){: width="750" height="450" }
_mysql死锁日志_

### 分析代码，定位死锁原因

- 一般有了服务和 MySQL 死锁日志，就很容易定位到代码死锁发生地，看代码分析逻辑就行了。

### 复现一个死锁 case（间隙锁非互斥）

- 一般死锁都是因为多个事务并发操作数据库造成的，确定数据库隔离级别：RR；本地安装 MySQL，构造死锁测试数据（数据纯属构造，如有雷同，纯属巧合），去复现下，

    ![Desktop View](/assets/img/20241208/deadlock_mysql_test_table.jpg){: width="750" height="450" }
    _mysql表测试数据_

    ![Desktop View](/assets/img/20241208/deadlock_mysql_test_table_index.jpg){: width="750" height="450" }
    _mysql表测试索引_

    ![Desktop View](/assets/img/20241208/deadlock_mysql_test_table_case.jpg){: width="750" height="450" }
    _mysql死锁测试case_

- 注：`1235905358376715619`，比它小一个的是`1235697761479571791`，比它大一个的是`1242077848279131794`。

- 原因是`间隙锁非互斥`，`UPDATE table SET user_id = null WHERE user_id = #{userId}`，这条语句本身就容易产生间隙锁死锁问题，最好增加 `flag` 标志来标识记录状态。

**分析，**

1. 先开启事务一，然后执行更新`user_id`=`1235905358376715619`的数据，`user_id`=`1235905358376715619`这条记录不存在，说明它会在`1235697761479571791`-`1242077848279131794`之间加间隙锁；

2. 开启事务二，然后也执行更新`user_id` = `1235905358376715619`的数据，事务二也会对 `1235697761479571791`-`1242077848279131794` 之间加间隙锁；

3. 此时我们需要做的操作就是让事务一在`1235697761479571791`-`1242077848279131794`之间插入或更新数据，会发现此时事务已经被阻塞，无法执行insert/update，因为事务二已经对该区间加了间隙锁；

    ![Desktop View](/assets/img/20241208/deadlock_mysql_test_table_lock.jpg){: width="750" height="450" }
    _mysql表-间隙锁_

4. 在事务一等待锁的同时，让事务二同时在 1235697761479571791-1242077848279131794 之间插入数据，这个时候会发现，只要事务二一旦执行 insert/update，立即报了死锁。

    ![Desktop View](/assets/img/20241208/deadlock_mysql_test_table_result.jpg){: width="750" height="450" }
    _死锁发生地_

## 总结

1. 类似两次加排他锁的线性更新操作，先用悲观锁查询，如果存在更新；不存在保存，不用更新。比如分布式锁防抖动，可以优先解决重放并发导致的死锁问题，缺点是锁操作较重；

2. 不要频繁更新索引字段，最好使用 id 来单条更新记录，更新时锁的范围会变小；

3. 不要执行批量大事务，批量大事务，会占用大锁，存在并发时，就会有死锁。有条件的话，一条一条插入/更新；如果非要批量插入数据，可以执行批量 insert SQL，保证原子性（insertBatch）；批量保存时，最好按照索引字段排序后，再保存。
