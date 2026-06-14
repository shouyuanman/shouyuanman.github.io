---
title: MySQL主从复制，以及数据一致性
date: 2026-06-13 18:00:00 +0800
categories: [后端, MySQL]
tags: [后端, MySQL]
music-id: 2734272059
---

## **序言**

>如果没有主从复制，会面临什么样的问题？
{: .prompt-tip }

比如`ES`和`RocketMQ`的高可用，实现高可用的策略，就是主从复制。`MySQL`也要做高可用，既然做高可用，主从复制的环节就避免不了。主从复制的环境，就是主从集群的搭建。

>Q：在`MySQL`主从复制的过程中，主从节点复制的是什么内容？
{: .prompt-tip }

复制的不是具体数据，而是操作日志。

下面按照三大日志的脉络，简略看下`Master`节点上面事务提交的流程，以及`Slave`节点上面`binlog`同步的过程。

![Desktop View](/assets/images/20260613/master_slave_replica_flow.png){: width="600" height="300" }
_Master节点上面事务提交的流程，Slave节点上面binlog同步的过程_

- 先看看事务从开始到提交过程中，三大日志的流程。数据从磁盘读入内存，先在内存中做数据更新，然后记录两大日志，一个是`undolog`，一个是`redolog`，等到提交事务时，做了两件事，一是记录`binlog`，一个是提交`redolog`。
- 在事务提交阶段，会写入`binlog`，如果从节点和主节点建立了复制关系，`Master`服务端会启动`dump`线程，`Slave`客户端会启动`IO`线程，两边建立连接，进行数据同步。
- 从节点接收到`binlog`后，在从节点叫中继日志，`SQL`线程会对中继日志进行回放，形成`sql`再执行，生成实际上的数据。

## **一键搭建MySQL主从集群**

### **docker**

准备`docker-compose.yml`、`Master`主配置文件`my.cnf`、`Slave`从配置文件`my.cnf`，

![Desktop View](/assets/images/20260613/mysql_master_slave_docker_yml.png){: width="600" height="300" }
_docker-compose yaml_

![Desktop View](/assets/images/20260613/mysql_master_conf.png){: width="600" height="300" }
_Master主配置文件my.cnf_

![Desktop View](/assets/images/20260613/mysql_slave_conf.png){: width="600" height="300" }
_Slave从配置文件my.cnf_

创建对应的目录结构，为目录赋予权限后，直接启动`MySQL`服务主从集群。

```terminal
$ mkdir -p /home/docker-compose/mysqlmasterslave/{master,slave}-{log,data}
$ cd /home/docker-compose/
$ chmod 777 -R /home/docker-compose/mysqlmasterslave
$ chmod 644 /home/docker-compose/mysqlmasterslave/master/my.cnf
$ chmod 644 /home/docker-compose/mysqlmasterslave/slave/my.cnf
$ chown -R mysql:mysql /home/docker-compose/mysqlmasterslave/slave/log
$ chown -R mysql:mysql /home/docker-compose/mysqlmasterslave/master/log
$ cd mysqlmasterslave
$ docker-compose --compatibility up -d
$ docker-compose  logs -f
```

查看启动的容器，如下，

![Desktop View](/assets/images/20260613/mysql_docker_ps.png){: width="600" height="300" }
_运行起来的容器列表_

从上面的`yaml`里，获取服务创建的网络名称，查询服务`ip`地址，如下，

![Desktop View](/assets/images/20260613/mysql_docker_network.png){: width="600" height="300" }
_服务网络拓扑_

通过`docker`编排的方式，一键搭建`MySQL`高可用主从集群。如果不用`docker`编排的方式，就得一行行地手动敲`shell`命令来搭建，工作量非常大。当然对`DBA`另当别论，他们有很多工具可以方便的维护高可用集群，如果开发人员来弄这种环境，还是用`docker`编排比较方便，需要考虑的配置和因素也没有这么多，更重要地是为掌握主从复制、高可用这些基础原理做准备。

虽然看到`docker`服务起来了，但是如果要做到主从复制的效果，还得对主从集群的`Master`、`Slave`进行配置。

### **配置Master**

```terminal
$ docker exec -it mysql-master bash
$ mysql -uroot -p123456
$ show variables like '%server_id%';
```

进入`MySQL`主服务，查看`server_id`是否生效，

![Desktop View](/assets/images/20260613/show_variables_like_server_id.png){: width="400" height="200" }
_server id_

```terminal
$ show master status;
```

看`Master`信息`File`和`Position`，从节点服务上要用，

![Desktop View](/assets/images/20260613/show_master_status.png){: width="600" height="300" }
_master status_

给`root`用户开远程权限，

```terminal
$ GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456';  
$ grant replication slave,replication client on *.* to 'slave'@'%' identified by "123456";
$ flush privileges;
```

在主节点上，为从节点复制创建一个用户`slave`，配置上对应密码，这个用户账号有复制的权限，专门用来做主从同步的，

```terminal
$ grant replication slave,replication client on *.* to 'slave'@'%' identified by "123456";
$ flush privileges;
```

![Desktop View](/assets/images/20260613/master_grant_slave.png){: width="600" height="300" }
_grant slave_

### **配置Slave**

```terminal
$ docker exec -it mysql-slave bash
$ mysql -uroot -p123456
$ show variables like '%server_id%';
```

进入`MySQL`从服务，查看从节点`server_id`是否生效，

![Desktop View](/assets/images/20260613/show_variables_like_server_id_slave.png){: width="400" height="200" }
_slave server id_

建立主从关系，从节点上通过在主节点建立的用户账号，连上去，开启主从同步机制（这里会用到前面在主节点上，查看`binlog`的文件名和偏移量），

```terminal
$ change master to master_host='mysql-master',master_user='slave',master_password='123456',master_port=3306,master_log_file='bin-log.000003',master_log_pos=1346,master_connect_retry=30;
```

连接主`MySQL`参数说明，
- `master_host`：主节点的`ip`（或主机名称）
- `master_port`：`Master`的端口号，指的是容器的端口号
- `master_user`：用于数据同步的用户
- `master_password`：用于同步的用户的密码
- `master_log_file`：指定`Slave`从哪个日志文件开始复制数据，即上文中提到的`File`字段的值
- `master_log_pos`：从哪个`Position`开始读，即上文中提到的`Position`字段的值
- `master_connect_retry`：如果连接失败，重试的时间间隔，单位是秒，默认是`60`秒

在从库上，启动`slave`线程，

```terminal
start slave;
```

![Desktop View](/assets/images/20260613/start_slave.png){: width="500" height="250" }
_slave start_

看下从节点状态，`Slave_IO_Running`和`Slave_SQL_Running`都是`YES`，表示主从复制环境配置成功了。

```terminal
$ show slave status \G;
```

![Desktop View](/assets/images/20260613/show_slave_status_ok.png){: width="600" height="300" }
_slave run ok_

>如果不配置`change master to`，就查不到`slave status`，执行`start slave`也会报错。
{: .prompt-tip }

![Desktop View](/assets/images/20260613/start_slave_error.png){: width="600" height="300" }
_start slave error_

如果没执行`start slave`，查看从节点的状态，

![Desktop View](/assets/images/20260613/show_slave_status_no_run.png){: width="600" height="300" }
_slave not running_

`Slave_IO_Running: No`，说明从节点负责接收`binlog`的线程没有在运行，已经停下来了，主从关系现在有问题，检查下主节点的配置对不对。

>如果主节点没正常授权`slave`，执行`start slave`也会报错`error connecting to master 'slave@mysql-master:3306' - retry-time: 30  retries: 4`。
{: .prompt-tip }

![Desktop View](/assets/images/20260613/start_slave_error_connect.png){: width="600" height="300" }
_start slave error_

授权成功后，需要停止`slave`，并重置下`slave`，

```terminal
stop slave;
reset slave;
```

如果授权之后，没有重置，报错如下，

![Desktop View](/assets/images/20260613/start_slave_error_not_find_binlog_index_file.png){: width="600" height="300" }
_start slave error_

重置完之后，再看下从节点状态，可以看到接收`binlog`的线程起来了。

### **准备数据源**

在主库创建表，确保从库也有以上两个表。这时，主库创建的表，会自动复制到从库。

只在主库执行创建，会看到从库也会同步生成。

![Desktop View](/assets/images/20260613/create_table_in_master.png){: width="600" height="300" }
_主库建表_

![Desktop View](/assets/images/20260613/sync_table_to_slave.png){: width="600" height="300" }
_从库同步_

## **把脉三大日志**

下面看下`MySQL`三大日志，
- `binlog`，归档日志，或者二进制日志
- `redolog`，重做日志
- `undolog`，回滚日志

### **binlog原理、使用场景、持久化策略**

前面说过，在主从复制的过程中，主从节点复制的不是具体数据，而是操作日志。之前在配置主从节点时，在配置里也打开了`log-bin`选项，这样就可以把`binlog`持久化。

从节点开始同步时，即执行`start slave`，主从节点间就开始了日志的传输。从节点通过`IO`线程拿到数据，在从节点本地，就不叫`binlog`，而叫`relay-log`（中继日志），拿到中继日志后，通过`SQL`线程解析，重新执行这个`SQL`生成数据。

**`binlog`的作用**
- 用于复制，在主从复制中，从库利用主库上的`binlog`进行回放，实现主从同步。
- 用于基于时间点的数据还原，主要是用于增量数据还原。

**`binlog`的使用场景**
- 平时会做数据备份，数据备份一般是周期性的备份，常用命令是`mysqldump`。假设第二天中午机器故障，`MySQL`崩溃，之前的备份数据还是昨天晚上的，昨天晚上到今天中午没有来得及备份，但数据还是要恢复，恢复数据时分为两部分，一部分是把昨天晚上全量的数据导入进去，再把增量的数据倒进来。
- 全量是每次周期性压缩成压缩包，需要时全量导入即可。
- 增量的部分数据，就是在`binlog`里边，把`binlog`文件导出一个`SQL`，再把这个`SQL`导入到库里边。

`binlog`保存的内容，跟格式有关系，**`binlog`有三种格式**，分别是`STATEMENT`、`ROW`、`MIXED`，
- `STATEMENT`模式：`binlog`里面记录的就是`SQL`语句的原文。优点是并不需要记录每一行的数据变化，减少了`binlog`日志量，节约`IO`，提高性能。缺点是在某些情况下会导致`master-slave`中的数据不一致，因为并没有记录修改之前的状态。
- `ROW`模式：不记录每条`SQL`语句的上下文信息，仅需记录哪条数据被修改了，修改成什么样了，解决了`STATEMENT`模式下出现`master-slave`中的数据不一致。缺点是会产生大量的日志，尤其是`alter table`的时候会让日志暴涨，因为它会影响到所有的行。比如`SQL`里边只改了一个字段，在该模式下会把有变化的和没变化的一并记录了下来，即整个行都记录下来了，修改之前和修改后的都记录下来了，一行一行来，假如是一条`update`语句，命中改了`3`行就会记录`3`行`binlog`。
- `MIXED`模式：折中方案，以上两种模式的混合使用，一般的复制使用`STATEMENT`模式保存`binlog`，对于`STATEMENT`模式无法复制的操作使用`ROW`模式保存`binlog`，`MySQL`会根据执行的`SQL`语句选择日志保存方式。

**`binlog`持久化策略**，可以用参数`sync_binlog`来配置，
- 在进行事务的过程中，首先会把`binlog`写入到`binlog cache`中（因为写入到`cache`中会比较快，一个事务通常会有多个操作，避免每个操作都直接写磁盘导致性能降低），事务最终提交的时候再把`binlog`写入到磁盘中。当然事务在最终`commit`的时候，`binlog`是否马上写入到磁盘中是由参数`sync_binlog`（同步刷盘） 配置来决定的。
- `sync_binlog=0`（不同步刷盘） 的时候（由后台线程异步刷盘），表示每次提交事务`binlog`不会马上写入到磁盘，而是先写到`page cache`（文件系统缓存），相对于磁盘写入来说，写`page cache`要快得多，但在异步刷盘的模式下，数据是不可靠的，在`MySQL`崩溃的时候会有丢失日志的风险。
- `sync_binlog=1`（同步刷盘） 的时候，表示每次提交事务都会执行`fsync`写入到磁盘（`binlog`文件，文件目录在配置文件里配置）。
- `sync_binlog`的值大于`1`（积累了`n`个事务才刷盘）的时候，表示每次提交事务都先写到`page cache`，只有等到积累了`n`个事务之后才`fsync`写入到磁盘，同样在此设置下`MySQL`崩溃的时候会有丢失`n`个事务日志的风险。

>很显然三种模式下，`sync_binlog=1`是强一致的选择，选择`0`或者`n`的情况下，在极端情况下就会有丢失日志的风险，具体选择什么模式还是得看系统对于一致性的要求。
- 对数据一致性要求很高，一点都不能丢失，就用同步刷盘；
- 对数据可靠性要求不高，就用异步刷盘。
{: .prompt-tip }

### **redolog原理、使用场景、持久化策略**

`redolog`和`undolog`是和事务有关的，根本目的是**保证事务的原子性和持久性**，隔离性是通过锁来保障的。
- `redolog`，就是在崩溃后，可以对提交的事务进行重做。
- `undolog`，是在崩溃后，对回滚的事务做撤销。
- `binlog`，不管有没有事务，`MySQL`都是有的。

`MySQL`中，事务和引擎有关，`MySQL`整体来看就有两块。一块是`Server`层，主要做的是`MySQL`功能层面的事情，比如`binlog`是`Server`层自己的日志。还有一块是引擎层，负责存储相关的具体事宜，比如`redolog`是`InnoDB`引擎特有的日志。

`MyISAM`是`MySQL`的默认数据库引擎（`5.5`版之前），由早期的`ISAM`所改良。虽然性能极佳，但却有一个缺点，不支持事务处理（`transaction`）。就是说`MyISAM`没有`redolog`和`undolog`。但不管是`InnoDB`，还是`MyISAM`，都有`binlog`。

`redolog`能保证对于已经`commit`的事务产生的数据变更，即使是系统宕机崩溃，也可以通过它来进行数据重做，达到数据的一致性，这也就是事务持久性的特征，一旦事务成功提交后，只要修改的数据都会进行持久化，不会因为异常、宕机而造成数据错误或丢失，所以解决异常、宕机而可能造成数据错误或丢失是`redolog`的核心职责。

>`redolog`是怎样的保障机制？
{: .prompt-tip }

`WAL`（`Write Ahead Log`），**预写日志（写前日志）**，当事务提交时，先写`redolog`，再修改页。这和`ES`写日志完全相反，`ES`是反过来的，先写数据，后写日志。

保障数据能够有效地恢复，还有一个策略，就是**两阶段提交**。

通过预写和两阶段提交，来保障`MySQL`数据高可靠（事务的原子性和持久性）。

#### **预写**

看下流程，在事务里边修改数据，先把数据读到内存，先写`redolog`，然后再写数据（更新数据），写入日志和数据还不算，最后会提交日志（日志的写入分两阶段，准备阶段和提交阶段）。

>先看写入的次序，数据和日志，谁先写？
{: .prompt-tip }

先写日志，再写数据，就是所谓的预写日志。

>为什么要做预写日志？
{: .prompt-tip }

假设一个程序在执行某些操作的过程中，机器掉电了，在重新启动时，若是使用了`WAL`，程序就能够检查`log`文件，对忽然掉电时计划执行的操作内容，跟实际上执行的操作内容进行比较（看哪些数据丢失了，再补回来）。在这个比较的基础上，程序就能够决定，是撤销已做的操作，还是继续完成已做的操作，或者是保持原样。

具体到`MySQL`里边，当有一条记录需要更新的时候，`InnoDB`会先把记录写到`redolog`里面，并更新`Buffer Pool`的`page`（文件系统缓存里边的`page`，`MySQL`也有自己的`page`页，比如文件系统缓存里边的`4K`或`16K`），这个时候更新操作就算完成了。

页面数据什么时候刷盘？这些页面会放在一个池子里边（`Buffer Pool`），
`Buffer Pool`是物理页的缓存，对`InnoDB`的任何修改操作都会首先在`Buffer Pool`的`page`上进行，然后这样的页将被标记为脏页并被放到专门的`Flush List`上，后续将由专门的刷脏线程阶段性的将这些页面写入磁盘。

**`redolog`的存储方式**

不是滚动增长的方式，和`RocketMQ`的日志文件或者`binlog`不同，`InnoDB`的`redolog`是固定大小的，比如可以配置为一组`4`个文件，每个文件的大小是`1GB`，循环使用，从头开始写，写到末尾就又回到开头循环写（顺序写，节省了随机写磁盘的`IO`消耗）。

里边有`write_pos`和`check_point`，简单理解下，

![Desktop View](/assets/images/20260613/redolog_store_struct.png){: width="400" height="200" }
_redolog存储方式_

`check_point`理解为尾巴，`write_po`s理解为头部，最开始时，这俩相等，每次增加，`write_pos`就往前推进，黄色部分是有`redolog`日志记录的。

>`redolog`日志记录和数据之间，有什么关系呢？
{: .prompt-tip }

这个图里只是日志，另外一部分是相关的数据，如果把数据更新到文件里边去之后，`check_point`就往前推动，也就是说黄色部分记录的是物理数据没有刷盘，如果物理数据刷盘了，标志位`check_point`会往后移动。这个环代表`4`个`G`的日志，是在`4`个文件里边。

如果头部追上了尾巴，说明有很多`MySQL`的数据没有刷盘，全都在`Buffer Pool`（物理内存）里边。

`redolog`肯定不会每次都写文件，还有对应的缓存，写`redolog`时，不是立马就写到文件里边去了，`InnoDB`首先将`redolog`放入到`redo log buffer`，然后按一定频率将其刷新到`redo log file`。下列三种情况下会将`redo log buffer`刷新到`redo log file`，
- `Master Thread`每一秒将`redo log buffer`刷新到`redo log file`
- 每个事务提交时会将`redo log buffer`刷新到`redo log file`
- 当`redolog`缓冲池剩余空间小于`1/2`时，会将`redo log buffer`刷新到`redo log file`

#### **两阶段提交**

准备阶段、提交阶段

假设事务是执行一个更新语句，流程是怎样的？

>举个例子，当我们执行`update t_flow set c=c+1 where id=1;`的时候大致流程如下，
1. 从磁盘读取到`id=1`的记录，放到内存；
2. 修改内存中的记录；
3. 记录`undo log`日志；
4. 记录`redo log`（`prepare`预提交状态）；
5. 记录`binlog`；
6. 提交事务，写入`redo log`（`commit`状态）。
{: .prompt-tip }

`redolog`，记录的是修改之后的值，用来重做，崩溃重启之后的重做。
`undolog`，记录的是修改之前的值，用来回滚，这里指的崩溃重启之后的回滚，不是每次回滚都会用它，
这俩次序不太分先后，三四步都是在内存里边进行的，这时做完预提交，还没有生效。
什么时候生效？要做一个提交的操作，提交事务时，先生成`binlog`，通过主从复制机制，还可以复制到从节点。`binlog`先写入内存，什么时候刷盘，由刷盘机制来确定。
写完`binlog`，`redolog`进入第二阶段（提交阶段），使`redolog`生效，提交事务的工作就做完了。

**两阶段提交的原因**

两阶段提交，是为了`binlog`和`redolog`两份日志之间的逻辑一致。
`redolog`和`binlog`都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。
如果不用两阶段提交，那么有两种可能，要么就是先写完`redolog`，再写`binlog`，或者采用反过来的顺序。


>为什么用两个阶段，而不是一个阶段就搞定？<br/>
因为有两个日志（`binlog`、`redolog`），如果不用两阶段提交，就会有数据不一致。
{: .prompt-tip }

`update`语句来做例子，
假设当前`id=2`的行，字段`c`的值是`0`，再假设执行`update`语句过程中，在写完第一个日志后，第二个日志还没有写完期间发生了`crash`崩溃，会出现什么情况呢？
1. 先写`redolog`后写`binlog`

    假设在`redolog`写完，`binlog`还没有写完的时候，`MySQL`进程异常重启。

    由于`redolog`写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行`c`的值是`1`。

    但是由于`binlog`没写完就`crash`了，这时候`binlog`里面就没有记录这个语句。

    因此，之后备份日志的时候，存起来的`binlog`里面就没有这条语句。

    然后你会发现，如果需要用这个`binlog`来恢复临时库的话，由于这个语句的`binlog`丢失，这个临时库就会少了这一次更新，恢复出来的这一行`c`的值就是`0`，与原库的值不同。

2. 先写`binlog`后写`redolog`

    如果在`binlog`写完之后`crash`，由于`redolog`还没写，崩溃恢复以后这个事务无效，所以这一行`c`的值是`0`。

    但是`binlog`里面已经记录了把`c`从`0`改成`1`这个日志。

    所以，在之后用`binlog`来恢复的时候就多了一个事务出来，恢复出来的这一行`c`的值就是`1`，与原库的值不同。

如果不使用两阶段提交，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。

使用两阶段提交，先写入`redolog`，再写`binlog`，写完`binlog`后，再标识`redolog`是`commit`阶段（标识有效），这时`binlog`得到了一个保障，`redolog`也得到了一个保障，保证数据的一致性。

![Desktop View](/assets/images/20260613/two_phase_commit_flow.png){: width="600" height="300" }
_两阶段提交_

#### **redolog刷盘（写入）策略**

也有一个参数来配置

`redolog`占用的空间是一定的，并不会无限增大（可以通过参数设置），写入的时候是顺序写的，所以写入的性能比较高。

当`redolog`空间满了之后，又会从头开始以循环的方式进行覆盖式的写入。

在写入`redolog`的时候也有一个`redo log buffer`，日志什么时候会刷到磁盘是通过`innodb_flush_log_at_trx_commit`参数决定。
- `innodb_flush_log_at_trx_commit=0`，表示每次事务提交时，不刷盘，都只是把`redolog`留在`redo log buffer`中；
- `innodb_flush_log_at_trx_commit=1`，表示每次事务提交时都将`redolog`直接持久化到磁盘；
- `innodb_flush_log_at_trx_commit=2`，表示每次事务提交时都只是把`redolog`写到`page cache`。

除了上面几种机制外，还有其它两种情况会把`redo log buffer`中的日志刷到磁盘，
- 定时处理：有线程会定时（每隔`1`秒）把`redo log buffer`中的数据刷盘。
- 根据空间处理：`redo log buffer`占用到了一定程度（`innodb_log_buffer_size`设置的值一半），这个时候也会把`redo log buffer`中的数据刷盘。

#### **redolog & Write-Ahead的本质**

`Write-ahead`，不是针对内存来说的，而是针对磁盘来说的。不是说在内存里先写日志，再写数据。在磁盘的角度，日志先落盘，数据再落盘。事务里边修改的实际业务数据的落盘。

这就回到了`IO`的本质，随机写，还是顺序写。

现在有两类数据，一个是日志数据，一个是业务数据，业务数据在磁盘上是分散的，而日志数据写入是顺序的，随机写和顺序写在性能上能相差几百倍。

`Write-ahead`的本质，一个是顺序写提高性能，另一个是系统宕机，只要日志在，数据就可以恢复。为了提升性能，数据就干脆晚一点写，不是立即写入磁盘，而是由后台线程异步刷入磁盘。这就回到了三高里面的高性能架构部分。

提交事务时，把日志追加到`redolog`的尾部，所有的日志文件都是以追加的形式写入的。

>一个事务要修改多张表的多条记录，多条记录分布在不同的`Page`里面，对应到磁盘的不同位置。如果每个事务都直接写磁盘，一次事务提交就要多次磁盘的随机`IO`，性能达不到要求。怎么办？
{: .prompt-tip }

不写磁盘，在内存中进行事务提交。然后再通过后台线程，异步地把内存中的数据写入到磁盘中。
但有个问题，机器宕机，内存中的数据还没来得及刷盘，数据就丢失了。
为此，就有了`Write-ahead Log`的思路。

先在内存中提交事务，然后写日志（所谓的`Write-ahead Log`），然后后台任务把内存中的数据异步刷到磁盘。
是顺序地在尾部`Append`，从而也就避免了一个事务发生多次磁盘随机`IO`的问题。
明明是先在内存中提交事务，后写的日志，为什么叫作`Write-Ahead`呢？
这里的`Ahead`，其实是指相对于真正的数据刷到磁盘，因为是先写的日志，后把内存数据刷到磁盘，所以叫`Write-Ahead Log`。


#### **redolog和binlog日志的不同**

`redolog`是物理记录，`binlog`是逻辑记录

`binlog`是逻辑记录，格式为`row`模式。比如`update t_user set age =18 where name ='shouyuan'`，如果这条语句修改了三条记录的话，那么`binlog`记录就是，

```markdown
UPDATE `db`.`t_user` WHERE @1=5 @2='shouyuan' @3=91 @4='1543571201' SET  @1=5 @2='shouyuan' @3=18 @4='1543571201'
UPDATE `db`.`t_user` WHERE @1=6 @2='shouyuan' @3=91 @4='1543571201' SET  @1=5 @2='shouyuan' @3=18 @4='1543571201'
UPDATE `db`.`t_user` WHERE @1=7 @2='shouyuan' @3=91 @4='1543571201' SET  @1=5 @2='shouyuan' @3=18 @4='1543571201'
```

`redolog`是物理记录，上面的修改语句，在`redolog`里面记录的可能就是下面的形式，

```markdown
把表空间10、页号5、偏移量为10处的值更新为18。
把表空间11、页号1、偏移量为2处的值更新为18。
把表空间12、页号2、偏移量为9处的值更新为18。
```

>
`redolog`是`InnoDB`引擎特有的；`binlog`是`MySQL`的`Server`层实现的，所有引擎都可以使用；<br/>
`redolog`是物理日志，记录的是在磁盘上某个位置某个数据上做了什么修改；`binlog`是逻辑日志，记录的是这个语句的原始逻辑，比如给`id=2`这一行的`c`字段加`1`；<br/>
`redolog`是循环写的，空间固定会用完；`binlog`是可以追加写入的，`binlog`文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。
{: .prompt-tip }


### **undolog原理、使用场景**

回滚日志
- 作用：保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（`MVCC`），也即非锁定读；
- 内容：逻辑格式的日志，根据每行记录进行记录。

在执行`undo`的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于`redolog`的。

>`undolog`与`redolog`的不同
- `undolog`用于记录事务开始前的状态，用于事务失败时的回滚操作；
- `redolog`记录事务执行后的状态，用来恢复未写入`data file`的已成功事务更新的数据。
{: .prompt-tip }

例如，某一事务的事务序号为`T1`，其对数据`c`进行修改，假设`c`的原值是`0`，修改后的值为`1`，那么`undolog`为`<T1, c, 0>`，`redolog`为`<T1, c, 1>`。

`redolog`和`undolog`的核心是，为了保证`InnoDB`事务机制中的持久性和原子性，事务提交成功由`redolog`保证数据持久性，而事务可以进行回滚，来保证事务操作的原子性，即原子性是通过`undolog`来保证的。

要对事务数据回滚到历史的数据状态，所以我们也能猜到`undolog`是保存的是数据的历史版本，通过历史版本让数据在任何时候都可以回滚到某一个事务开始之前的状态。
`undolog`除了进行事务回滚的日志外还有一个作用，就是为数据库提供`MVCC`多版本数据读的功能。

`binlog`可以配置文件在哪，`redolog`可以配置一组是几个文件，`undolog`物理文件在哪？
- `MySQL5.6`之前，文件目录不用配置，就是位于数据文件目录中，`undo`表空间位于共享表空间的回滚段中，共享表空间的默认的名称是`ibdata`，位于数据文件目录中。
- `MySQL5.6`之后，`undo`表空间可以配置成独立的文件，但是提前需要在配置文件中配置，完成数据库初始化后生效且不可改变`undolog`文件的个数。

### **MySQL crash崩溃恢复**

前面简单串一下三大日志的总体生成流程，

>执行`update t_flow set c=c+1 where id=1;`的时候大致流程如下，
1. 从磁盘读取到`id=1`的记录，放到内存；
2. 修改内存中的记录；
3. 记录`undo log`日志；
4. 记录`redo log`（`prepare`预提交状态）；
5. 记录`binlog`；
6. 提交事务，写入`redo log`（`commit`状态）。
{: .prompt-tip }

我们根据上面的流程来看，如果在上面的某一个阶段`MySQL`数据库崩溃，如何恢复数据（保障数据的一致性）？

1. 在第一步、第二步、第三步，执行时，数据库崩溃，因为这个时候数据还没有发生任何变化，所以没有任何影响，不需要做任何操作。
2. 在第四步，修改内存中的记录时，数据库崩溃，因为此时事务没有`commit`，所以这里要进行数据回滚，所以这里会通过`undolog`进行数据回滚。
3. 第五步，写入`binlog`时，数据库崩溃，这里和第四步一样的逻辑，此时事务没有`commit`，所以这里要进行数据回滚，会通过`undolog`进行数据回滚。
`binlog`不存在事务记录，那么这种情况事务还未提交成功，`redolog`也没有`commit`标记，所以会对数据进行回滚。
4. 执行第六步，事务提交时，数据库崩溃，如果数据库在这个阶段崩溃，那其实事务还是没有提交成功，但是这里并不能像之前一样对数据进行回滚，因为在提交事务前，`binlog`可能成功写入磁盘了，所以这里要根据两种情况来做决定。
- 一种情况，`binlog`不存在事务记录，主从也是一致的，那么这种情况事务还未提交成功，`redolog`就算刷盘了也没有`commit`标记，所以会对数据进行回滚。
- 一种情况， 是`binlog`存在数据记录（已经刷盘了），而`binlog`写入后，那么依赖于`binlog`的其它扩展业务（比如从库已经同步了日志进行数据的变更）数据就已经产生了，如果这里进行数据回滚，那么势必就会造成主从数据的不一致，此时不能回滚（虽然事务在数据层还没提交，但在业务侧肯定做提交的动作了，不然不会走到这一步），**怎么办呢？**

>因为提交崩溃了，`redolog`不会有`commit`标志，但是这时也可以根据`redolog`来重做，我们之前说在第六阶段，写入`redolog`，实际上并不完全是这样，
在高可靠的场景下，如果把`innodb_flush_log_at_trx_commit`设置成`1`，那么`redolog`在`prepare`阶段就要持久化一次，崩溃恢复逻辑是要依赖于`prepare`的`redolog`，再加上`binlog`来恢复的。结合`binlog`的状态，进行`redo`。如果`binlog`存在事务记录，那么就认为事务已经提交了，这里可以根据`binlog`对数据进行重做。其实这个阶段发生崩溃了，最终的事务是没提交成功的，这里应该对数据进行回滚。
{: .prompt-tip }

### **如何解决主从服务之间的延时较大的问题**

从库`B`和主库`A`之间维持了一个长连接。主库`A`内部有一个线程，专门用于服务从库`B`的这个长连接。一个事务`binlog`日志同步的完整过程如下，
- 在从库`B`上通过`change master`命令，设置主库`A`的`IP`、端口、用户名、密码，以及要从哪个位置开始请求`binlog`，这个位置包含文件名和日志偏移量；
- 在从库`B`上执行`start slave`命令，这时从库会启动两个线程，就是图中的`IO`线程和`SQL`线程。其中`IO`线程负责与主库建立连接；
- 主库`A`校验完用户名、密码后，开始按照从库`B`传过来的位置，从本地读取`binlog`，发给`B`；
- 从库`B`拿到`binlog`后，写到本地文件，称为中继日志`relay log`；
- `SQL`线程读取中继日志`relay log`，解析出日志里的命令，并执行。

由于多线程复制方案的引入，`SQL`线程演化成了多个线程。主从复制不是完全实时地进行同步，而是异步实时。

这中间存在主从服务之间的执行延时，如果主服务器的压力很大，则可能导致主从服务器延时较大。

首先能想到的是，从从节点上做优化，`SQL`回放线程是单线程的，为了加速回放，可以设置参数并行回放。

```markdown
slave_parallel_type='LOGICAL_CLOCK'
slave_parallel_workers=8
```

从主节点来说，要减小主节点的压力，利用分库的方式，分成多个主节点，这样瓶颈压力就小些。

## **实现读写分离**

![Desktop View](/assets/images/20260613/shardingjdbc_impl_rw_separation.png){: width="600" height="300" }
_ShardingJDBC实现读写分离_

![Desktop View](/assets/images/20260613/rw_separation_w_master.png){: width="600" height="300" }
_写主节点_

![Desktop View](/assets/images/20260613/rw_separation_r_slave.png){: width="600" height="300" }
_读从节点_

从程序的角度，怎么规避主从复制延迟问题？

刚插入一条数据，然后马上就要去读取，这个时候有可能会读取不到。归根到底是因为主节点写入完之后数据是要复制给从节点的，读不到的原因是复制的时间比较长，也就是说数据还没复制到从节点，你就已经去从节点读取了，肯定读不到。

`MySQL 5.7`的主从复制是多线程了，意味着速度会变快，但是不一定能保证百分百马上读取到，这个问题我们可以有两种方式解决，
- 业务层面妥协，是否操作完之后马上要进行读取；
- 对于操作完马上要读出来的，且业务上不能妥协的，我们可以对于这类的读取直接走主库，当然`ShardingJDBC`也是考虑到这个问题的存在，所以给我们提供了一个功能，可以让用户在使用的时候。指定要不要走主库进行读取。

在读取前使用下面的方式进行设置就可以了。

![Desktop View](/assets/images/20260613/shardingjdbc_impl_rw_separation_r_master.png){: width="600" height="300" }
_读主节点_

## **Canal实现数据一致性**

>什么是`canal`？<br/>
可以简单理解为一个假的（伪装的）`MySQL Slave`。
{: .prompt-tip }

也是使用`dump`协议同步`binlog`数据，最大的区别在于，没有回放线程，或者说回放线程不一样，不是生成数据，而是进行转发。可以通过`Socket`转发出去，也可以发送`RocketMQ`，支持多种方式的转发。

比如，把`canal.serverMode`选项修改为`rocketMQ`类型，

![Desktop View](/assets/images/20260613/canal_server_mode.png){: width="600" height="300" }
_修改canal.serverMode_

`canal [kə'næl]`，译意为水道/管道/沟渠，主要用途是基于`MySQL`数据库增量日志解析，提供增量数据订阅和消费。源码参见[阿里云`DTS`（`Data Transfer Service`）的开源版本](https://github.com/alibaba/canal)。

### **`canal`的工作原理**
- `canal`模拟`MySQL Slave`的交互协议，伪装自己为`MySQL Slave`，向`MySQL Master`发送`dump`请求；
- `MySQL Master`收到`dump`请求，开始推送`binlog`给`Slave`(也就是`canal`)；
- `canal`解析`binlog`对象（原始为`byte`流）；
- `canal`将解析后的对象，根据业务场景，分发到比如`MySQL`、`RocketMQ`或者`ES`中。

### **`canal`使用场景**

在很多业务情况下，我们都会在系统中加入`redis`缓存做查询优化（比如三级缓存的数据一致性，删除或更新缓存），使用`ES`做全文检索，`HBase/MongoDB`做海量存储（分库分表后，需要做关联查询，或者跨库查询等复杂检索，引入索引和存储分离）。

如果数据库数据发生更新，这时候就需要在业务代码中写一段同步更新`redis`、`ES`、`HBase`的代码。这种数据同步的代码跟业务代码糅合在一起会不太优雅，能不能把这些数据同步的代码抽出来形成一个独立的模块呢，答案是可以的，而且架构上也非常漂亮。

![Desktop View](/assets/images/20260613/canal_arch.png){: width="600" height="300" }
_canal架构_

`canal`可以作为`MySQL binlog`增量订阅消费组件 + `MQ`消息队列，将增量数据更新到
- `redis`高速缓存
- `ES`做全文检索 
- `HBase/MongoDB`做海量存储。

图中的`redis`缓存操作服务、`ES`索引操作服务、`HBase`海量存储操作服务，都扮演了`binlog`适配器`adapter`的角色。

>一般地，`redis`缓存和`MySQL`数据一致性解决方案
- 延时双删策略
- 异步更新缓存（基于订阅`binlog`的延迟更新机制）
- ...（更多的，比如金融业务需要强一致性做补偿的方案，以后再起一个议题单独聊聊）
{: .prompt-tip }

### **基于canal的数据一致性搭建环境**

`Master`为`canal`创建专用账号，并且授权。登录`MySQL`输入以下命令，`canal`的原理是模拟自己为`MySQL Slave`，所以这里一定需要做为`MySQL Slave`的相关权限。

```terminal
$ docker exec -it mysql-master bash
$ mysql -uroot -p123456
$ CREATE USER canal IDENTIFIED BY 'canal';  
$ grant select, replication slave,replication client on *.* to 'canal'@'%' identified by "canal";
$ flush privileges;
```

创建`canal`数据库表，

![Desktop View](/assets/images/20260613/canal_database.png){: width="600" height="300" }
_canal数据库表_

准备好`canal-admin`、`canal-server`、`rocketmq-broker`、`rocketmq-namesrv`目录，以及相关配置文件（比如`canal admin`启动脚本、`admin`应用配置、`canal server`实例配置、`rocketmq broker`配置等），

![Desktop View](/assets/images/20260613/canal_admin_startup.png){: width="600" height="300" }
_canal admin 启动脚本_

![Desktop View](/assets/images/20260613/canal_admin_application_yaml.png){: width="600" height="300" }
_canal admin 应用配置_

![Desktop View](/assets/images/20260613/canal_server_instance_config.png){: width="600" height="300" }
_canal server 实例配置_

![Desktop View](/assets/images/20260613/rocketmq_broker_config.png){: width="600" height="300" }
_rocketmq broker配置_

配置`docker-compose`编排文件，

![Desktop View](/assets/images/20260613/canal_docker_compose.png){: width="600" height="300" }
_canal docker compose_

`canal-admin`可以理解为配置中心，在没有`admin`之前，`canal`的配置文件都是通过文件的方式配置的，有了`canal-admin`，就可以在`admin`读取配置`canal`信息，角色有点像`nacos`。

`canal-admin`连的数据库是`mysql-master`，用的`db`是`canal_manager`，用的网络是搭建`MySQL`主从集群的时候创建的子网络`mysql-canal-network`，相对于`canal`编排文件来说，这个网络是一个外部网络，`canal`容器和`MySQL`容器处于一个子网内部，相互之间访问可以不用`ip`，可以直接用主机名，

实际`canal`服务`canal-server`，要连到配置中心，`canal-admin:8089`，

上述准备工作做好之后，启动`canal`服务，

```terminal
$ cd /home/docker-compose/
$ chmod 777 -R canal
$ cd canal
$ docker-compose --compatibility up -d
$ docker-compose logs -f
```

访问`canal admin`，并且配置实例`instance`。如果使用的默认的配置信息，用户名入`admin`，密码输入`123456`即可访问首页。

![Desktop View](/assets/images/20260613/canal_homepage.png){: width="600" height="300" }
_canal首页_

可以看到，`canal-server`已经注册进来了，现在要把它的日志发送到`RocketMQ`，修改`serverMode`，

![Desktop View](/assets/images/20260613/canal_server_mode_config.png){: width="600" height="300" }
_canal配置serverMode_

配置生产者`group`、`nameserver`，

![Desktop View](/assets/images/20260613/canal_server_group_config.png){: width="600" height="300" }
_canal配置生产者group、nameserver_

新建实例配置，配置好之后启动。首先要从`master`接收`binlog`，需要配置下`master`，以及`mq topic`，

![Desktop View](/assets/images/20260613/canal_instance_config.png){: width="600" height="300" }
_canal配置实例_

启动后，查看日志，

![Desktop View](/assets/images/20260613/canal_start_log.png){: width="600" height="300" }
_canal启动日志_

从`master`查看从节点，也可以验证`Canal`成功启动，

![Desktop View](/assets/images/20260613/canal_start_ok_in_master.png){: width="600" height="300" }
_canal启动成功_

启动后打开`RocketMQ`，在`MySQL`侧改动数据，可以看到消息，就调通了。

![Desktop View](/assets/images/20260613/rocketmq_homepage.png){: width="600" height="300" }
_RocketMQ首页_

![Desktop View](/assets/images/20260613/rocketmq_message.png){: width="600" height="300" }
_RocketMQ查询消息_

![Desktop View](/assets/images/20260613/rocketmq_message_detail.png){: width="600" height="300" }
_RocketMQ消息明细_

### **Canal高可用集群架构**

使用`Cannel`，为了保证系统达到`4`个`9`、甚至`5`个`9`的高可用性，`Canal`服务不能是单节点的，一定是高可用集群的形式存在。

为什么呢？如果`Canal`保存数据不成功，就会导致数据库跟缓存或异构存储（比如`ES`、或者`redis`）数据不一致。

`Canal`单节点用于学习、用于测试是没问题的；但是`Canal`单节点用于生产，会严重影响系统健壮性，稳定性，所以得把`Canal`部署成高可用集群。

`Canal`部署成高可用集群的架构如下，

![Desktop View](/assets/images/20260613/canal_arch_cluster.png){: width="600" height="300" }
_canal集群架构_

`Canal`的`HA`（双机集群）分为两部分，`Canal server`和`Canal client`分别有对应的`HA`实现。

#### **Canal server**

为了减少对`MySQL dump`的请求，不同`server`上的实例(`instance`)要求同一时间只能有一个处于`running`，其他的处于`standby`状态。
或者说，由于`instance`由`Canal server`负责执行，所以同一个集群里边的`Canal server`，同一时间只能有一个处于`running`，其他的处于`standby`状态。

#### **Canal client**

为了保证有序性，一份实例(`instance`)同一时间只能由一个`canal client`进行`get/ack/rollback`等远程操作，否则客户端接收无法保证有序。

**`Zookeeper`负责协调**

整个`HA`机制的控制，主要是依赖了`zookeeper`的几个特性，`watcher`和`EPHEMERAL`节点（和`session`生命周期绑定）。
- 同一个集群里边的`Canal server`，需要去创建和监听属于`Server`的唯一的`znode`节点，成功则`running`，失败则`standby`；
- 同一个集群里边的`Canal client`, 需要去创建和监听属于`Client`的唯一的`znode`节点，成功则`running`，失败则`standby`。

`standby`的空闲角色，一直监听唯一的`znode`节点过期状态，随时准备去争抢转正机会。

#### **Canal高可用Server的协作流程**

1. `canal server`要启动某个`canal instance`时, 都先向`zookeeper`进行一次尝试启动判断（实现：创建`EPHEMERAL`节点，谁创建成功就允许谁启动）。
2. 创建`zookeeper`节点成功后，对应的`canal server`就启动对应的`canal instance`，没有创建成功的`canal instance`就会处于`standby`状态。
3. 一旦`zookeeper`发现`canal server A`创建的节点消失后，立即通知其他的`canal server`再次进行步骤`1`的操作，重新选出一个`canal server`启动`instance`。
4. `canal client`每次进行`connect`时，会首先向`zookeeper`询问当前是谁启动了`canal instance`，然后和其建立链接，一旦链接不可用，会重新尝试`connect`。

>注：`canal client`的方式和`canal server`方式类似，也是利用`zookeeper`的抢占`EPHEMERAL`节点的方式进行控制。
{: .prompt-tip }

### **Canal的核心角色**

再理解一下`Canal`的核心角色。

#### **canal server**

可以简单地把`Canal`理解为一个用来同步增量数据的一个工具。我们看一张官网提供的示意图，

![Desktop View](/assets/images/20260613/canal_offical_flow.png){: width="600" height="300" }
_canal flow_

`Canal`的工作原理，就是把自己伪装成`MySQL Slave`，模拟`MySQL Slave`的交互协议向`MySQL Master`发送`dump`协议，`MySQL Master`收到`Canal`发送过来的`dump`请求，开始推送`binlog`给`Canal`，然后`Canal`解析`binlog`，再发送到存储目的地，比如`MySQL`、`RocketMQ`、`Kafka`、`ES`等。

因为在`TCP`模式下，一个`instance`只能有一个`Canal client`订阅，即使同时有多个`Canal client`订阅相同的`instance`，也只会有一个`Canal client`成功获取`binlog`，所以`Canal server`写死`clientId = 1001`，
1. 也正是因为一个`instance`只有一个`Canal client`，所以`Canal server`将`binlog`位点信息维护在了`instance`级别，即`conf/content/meta.dat`文件中
2. 在`TCP`模式下，如果`Canal client`想重新获取以前的`binlog`，只能通过修改`Canal server`的`initial position`配置，并重启服务来达到目的
3. 在`TCP`模式下，`Canal server`主要提供了两个功能
- 维护`MySQL binlog position`信息，目的是作为`dump`的请求参数，这也是`canal server`唯一保存的数据
- 对客户端提供接口以查询`binlog`

`canal.serverMode` 的服务模式有：`tcp`、`kafka`、`rocketMQ`、`rabbitMQ`，可以把选项修改为`rocketMQ`类型，如下，

![Desktop View](/assets/images/20260613/canal_server_mode_config.png){: width="600" height="300" }
_canal配置serverMode_

这时候，就是`Canal server`把收到的`binlog`，按照`instance`的过滤要求完成处理后，写入到`rocketMQ`。

`canal server`负责`canal instance`的启动。`canal server`启动过程中的关键信息如下，

1. 确定binlog first position（通过以上三步，就可以确定`Canal server`启动之后`binlog`初始位点）
- 先从`conf/content/meta.dat`文件中查找`last position`, 也就是最后一次成功`dump binlog`的位点
- 如果不存在`last position`, 则从`conf/content/instance.properties`配置文件中查找`initial position`, 这是我们人为配置的初始化位点
- 如果不存在`initial position`，则执行`show master status`命令获取`mysql binlog lastest position`

2. 将`first position`赋值给`last position`保存在内存中
3. 将`schema`缓存到`conf/content/h2.mv.db`文件中

#### **canal client**

`canal.serverMode`的服务模式有`tcp`、`kafka`、`rocketMQ`、`rabbitMQ`。默认情况下，是`tcp`，就是开启一个`Netty`服务，发送`binlog`到`Client`。

`canal client`需要自己开发`TCP`客户端，可以参考官方的`canal client`实现。
1. `canal client`的`java demo`可以去官方`GitHub`上找一下，记得将`destination`等配置信息改正确。请参考[alibaba canal wiki](https://github.com/alibaba/canal/wiki/ClientExample)
2. `canal client connect`
3. `canal client describe`
- 在收到客户端订阅请求之后，`logs/content/content.log`文件会打印出相关日志
- `conf/content/meta.dat`文件记录了客户端的订阅信息，包括`clientId`、`destination`、`filter`等
4. `canal client getWithoutAck`
- `canal server`在收到`canal client`查询请求之后，以内存中的`last position`作为参数向`MySQL server`发送`dump`请求
- 如果存在比`last position`更新的`binlog`，`canal server`会收到`MySQL server`的返回数据，然后将其转换为`Message`数据结构返回给`canal client`
5. `canal client ack`
- `canal client`收到`canal server`的数据之后，可以发送`ack`确认`last position`的同步位置。
- `canal server`在收到`canal client`确认请求之后，更新内存中的`last position`，并同步保存到`conf/content/meta.dat`文件中，在`logs/content/meta.log`文件中打印日志

#### **canal instance**

`canal server`仅仅是保姆角色，真正完成解析`binlog`日志、`binlog`日志过滤、`binlog`日志转储、位点元数据管理等核心功能，是由`canal instance`角色完成。

`Canal Instance`的架构图如下图所示，

![Desktop View](/assets/images/20260613/canal_arch_instance.png){: width="600" height="300" }
_canal instance架构图_

`Canal`中数据的同步是由`Canal Instance`组件负责，一个`Canal Server`实例中可以创建多个`Canal Instance`实例。每一个`Canal Instance`可以看成是对应一个`MySQL`实例，即案例中需要同步两个数据库实例，故最终需要创建两个`Canal Instance`。

其实也不难理解，因为`MySQL`的`binlog`就是以实例为维度进行存储的。

`Canal Instance`包含了`4`个核心组件：`EventParse`、`EventSink`、`EventStore`、`CanaMetaManager`，在这里主要是阐明其作用，以便更好的指导实践。
1. `EventParse`组件
- 负责解析`binlog`日志，其职责就是根据`binlog`的存储格式将有效数据提取出来，这个不难理解，我们也可以通过该模块，进一步了解一下`binglog`的存储格式。
2. `EventSink`组件
- 在一个数据库实例上通常会创建多个`Schema`，但通常并不是所有的`Schema`都需要被同步，如果直接将`EventParse`解析出来的数据全部传入`EventStore`组件，将对`EventStore`带来不必要的性能消耗；
- 另外本例中使用了分库分表，需要将多个库的数据同步到单一源，可能需要涉及到合并、归并等策略。
- 以上等等等需求就是`EventSink`需要解决的问题域。
3. `EventStore`组件
- 用来存储经`canal`转换的数据，被`Canal Client`进行消费的数据，目前`Canal`只提供了基于内存的存储实现。
4. `CanalMetaManager`组件（元数据存储管理器）
- 在`Canal`中，最基本的元数据至少应该包含`EventParse`组件解析的位点与消费端的消费位点。
- `Canal Server`重启后，要能从上一次未同步位置开始同步，否则会丢失数据。

#### **canal cluster集群**

多个`canal server`，可以在创建的时候，归属到一个集群`cluster`下边。

一个集群`cluster`下边，同时只有一个`cannel server running`，其他的`standby`，实现高可用。

#### **canal admin**

1. 通过图形化界面管理配置参数
2. 动态启停`Server`和`Instance`
3. 查看日志信息

## **附录**

### **一键搭建Canal集群**

`Zookeeper + MySQL + Canal + RocketMQ`配置，感兴趣可以自己玩一下，这里暂且不作介绍，省略一万字。

### **应用：基于canal + RocketMQ实现微服务应用的数据一致性**

实现微服务（`Go`或者`Java`）应用代码，实现监听`RocketMQ`的`binlog`消息，实现刷`Redis`、`ES`、`Nginx`字典等目的。
