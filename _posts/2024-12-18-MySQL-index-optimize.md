---
title: MySQL 索引优化
date: 2024-12-18 16:28:30 +0800
categories: [后端, MySQL]
tags: [后端, MySQL]
music-id: 2116852179
---

## **索引实践**
这里所有的实践都是基于`MySQL 5.7.26`版本的`InnoDB`存储引擎！！！

在实操之前，需要确定当前使用的`MySQL`版本，因为不同的版本，对同一问题的表现不同。
```terminal
mysql> SELECT VERSION();
+-----------+
| VERSION() |
+-----------+
| 5.7.26    |
+-----------+
1 row in set (0.00 sec)
```
查看当前数据库支持的存储引擎，
```terminal
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES | YES  | YES |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO | NO | NO |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO | NO | NO |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO | NO | NO |
| CSV                | YES     | CSV storage engine                                             | NO | NO | NO |
| MyISAM             | YES     | MyISAM storage engine                                          | NO | NO | NO |
| ARCHIVE            | YES     | Archive storage engine                                         | NO | NO | NO |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO| NO | NO |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL | NULL | NULL |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
```
查看当前数据库默认的存储引擎，
```terminal
mysql> show variables like '%storage_engine%';
+----------------------------------+--------+
| Variable_name                    | Value  |
+----------------------------------+--------+
| default_storage_engine           | InnoDB |
| default_tmp_storage_engine       | InnoDB |
| disabled_storage_engines         |        |
| internal_tmp_disk_storage_engine | InnoDB |
+----------------------------------+--------+
4 rows in set (0.01 sec)
```
查看具体某个表使用的存储引擎，
```terminal
mysql> show table status from sakila where name="actor" \G
*************************** 1. row *************************** Name: actor
            Engine: InnoDB Version: 10
        Row_format: Dynamic Rows: 200
    Avg_row_length: 81
       Data_length: 16384
   Max_data_length: 0
      Index_length: 16384
         Data_free: 0
    Auto_increment: 201
       Create_time: 2023-08-06 17:15:38
       Update_time: 2023-08-06 17:16:33 Check_time: NULL
         Collation: utf8mb4_general_ci Checksum: NULL
    Create_options:
           Comment:
1 row in set (0.00 sec)
```
查看样本库的数据量，
```terminal
+----------------------------+----------+------------+
| Tables                     | Columns  | Total Rows |
+----------------------------+----------+------------+
| actor                      | 4        | 200        |
| actor_info                 | 4        | 200        |
| address                    | 9        | 603        |
| category                   | 3        | 16         |
| city                       | 4        | 600        |
| country                    | 3        | 109        |
| customer                   | 9        | 599        |
| customer_list              | 9        | 599        |
| film                       | 13       | 1000       |
| film_actor                 | 3        | 5462       |
| film_category              | 3        | 1000       |
| film_list                  | 8        | 1000       |        
| film_text                  | 3        | 1000       |
| inventory                  | 4        | 4581       |
| language                   | 3        | 6          |
| nicer_but_slower_film_list | 8        | 1000       |
| payment                    | 7        | 16044      |
| rental                     | 7        | 16044      |
| sales_by_film_category     | 2        | 16         |
| sales_by_store             | 3        | 2          |
| staff                      | 11       | 2          |
| staff_list                 | 8        | 2          |
| store                      | 4        | 2          |
+----------------------------+----------+------------+
```

### **索引设计原则**
#### **设计原则**
1. 搜索的索引列，不一定是所要选择的列。
- 最适合索引的列是出现在`where`子句里的列，或者连接子句中的列，而不是出现在`select`关键字后的选择列表中的列。
2. 使用唯一索引
- 考虑列中值的分布，索引中列的基数越大，索引的效果越好。
3. 使用短索引
- 如果对字符串列进行索引，应该指定一个前缀长度，只要有可能就应该这么做。
- 这样能够节省大量索引空间，也可能会是查询更快，较小的索引涉及的磁盘`ID`较少，较短的值比较起来更快。
- 对于较短的键值，索引高速缓存中的块能容纳更多的键值，这样，`MySQL`可以在内存中容纳更多的值，就增加了找到行而不用读取索引中较多块的可能性。
4. 利用最左前缀
- 在创建一个`n`列的索引时，实际是创建了`MySQL`可利用的`n`个索引，多列索引可起几个索引的作用，可利用索引中最左边的列集来匹配行。 
5. 不要过度索引
- 索引不是越多越好，每个额外的索引都要占用磁盘空间，并降低写操作性能。
- 在修改表的内容时，索引必须进行更新，有时可能需要重构。索引越多，花的时间越长。如果一个索引很少使用或从不使用，就不必要地减缓表的 修改速度。
- `MySQL`在生成一个执行计划时，要考虑各个索引，也要花费时间。创建多余的索引给查询优化带来了更多的工作。
- 索引太多，也可能会使`MySQL`选择不到所要使用的最好索引，只保持所需的索引有利于查询优化。
6. `InnoDB`表记录默认会按照一定顺序保存，
- 如果有明确定义的主键，则按照主键顺序保存。
- 如果没有主键，但有唯一索引，就按照唯一索引的顺序保存。
- 如果既没有主键，也没有唯一索引，表中会自动生成一个内部列，按照这个列的顺序保存。按照主键或者内部列进行的访问最快，所以`InnoDB`表尽量自己指定主键。
- 当表中同时有几个列都是唯一的，都可以作为主键时，要选择最常访问条件的列作为主键，提高查询效率。
- `InnoDB`表的普通索引都会保存主键的键值，所以主键要尽可能选择较短的数据类型，减少索引的磁盘占用，提高索引的缓存效果。

#### **应该创建索引的列**
- 在经常使用在`WHERE`子句中的列上面创建索引，加快条件的判断速度。
- 在经常需要搜索的列上，可以加快搜索的速度
- 在作为主键的列上，强制该列的唯一性和组织表中数据的排列结构
- 在经常用在连接（`JOIN`）的列上，这些列主要是一外键，可以加快连接的速度
- 在经常需要根据范围（`<`，`<=`，`=`，`>`，`>=`，`BETWEEN`，`IN`）进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的
- 在经常需要排序（`order by`）的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间。

### **索引问题**
```terminal
mysql> desc rental;
+--------------+-----------------------+------+-----+-------------------+-----------------------------+
| Field        | Type                  | Null | Key | Default | Extra |
+--------------+-----------------------+------+-----+-------------------+-----------------------------+
| customer_id  | smallint(5) unsigned  | NO   | MUL | NULL    |       |
+--------------+-----------------------+------+-----+-------------------+-----------------------------+
7 rows in set (0.00 sec) 
 
mysql> show index from rental;
+--------+------------+---------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table  | Non_unique | Key_name            | Seq_in_index | Column_name  | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+--------+------------+---------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| rental | 0 | PRIMARY                      | 1            | rental_id    | A | 16008 | NULL | NULL | | BTREE | | |
| rental | 1 | idx_fk_inventory_id          | NULL         |              | BTREE | |
| rental | 1 | idx_fk_customer_id           | NULL         |              | BTREE | |
| rental | 1 | idx_fk_staff_id              | NULL         |              | BTREE | |
| rental | 1 | idx_rental_date              | NULL         |              | BTREE | |
| rental | 1 | idx_rental_date              | NULL         |              | BTREE | |
| rental | 1 | idx_rental_date              | NULL         |              | BTREE | | |
+--------+------------+---------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
7 rows in set (0.01 sec)
 
mysql> desc payment;
+--------------+----------------------+------+-----+-------------------+-----------------------------+
| Field        | Type                 | Null | Key | Default | Extra |
+--------------+----------------------+------+-----+-------------------+-----------------------------+
| payment_id   | smallint(5) unsigned | NO   | PRI | NULL | auto_increment |
| customer_id  | smallint(5) unsigned | NO   | MUL | NULL | |
| staff_id     | tinyint(3) unsigned  | NO   | MUL | NULL | |
| rental_id    | int(11)              | YES  | MUL | NULL | |
| amount       | decimal(5,2)         | NO   | | NULL | |
| payment_date | datetime             | NO   | | NULL | |
| last_update  | timestamp            | NO   | | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+--------------+----------------------+------+-----+-------------------+-----------------------------+
7 rows in set (0.00 sec)
 
mysql> show index from payment;
+---------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table   | Non_unique | Key_name           | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+---------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| payment | 0          | PRIMARY            | 1            | payment_id  | A | 16086 | NULL | NULL | | BTREE | | |
| payment | 1          | idx_fk_staff_id    | 1            | staff_id    | A | 2 | NULL | NULL | | BTREE | | |
| payment | 1          | idx_fk_customer_id | 1            | customer_id | A | 599 | NULL | NULL | | BTREE | | |
| payment | 1          | fk_payment_rental  | 1            | rental_id   | A | 16044 | NULL | NULL | YES | BTREE | | |
+---------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
4 rows in set (0.00 sec)
 
mysql> desc film_text;
+-------------+--------------+------+-----+---------+-------+
| Field       | Type | Null  | Key  | Default | Extra |
+-------------+--------------+------+-----+---------+-------+
| film_id     | smallint(6)  | NO   | PRI     | NULL | |
| title       | varchar(255) | NO   | MUL     | NULL | |
| description | text         | YES  |         | NULL | |
+-------------+--------------+------+-----+---------+-------+
3 rows in set (0.00 sec)
 
mysql> show index from film_text;
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------
+----------+--------+------+------------+---------+---------------+
| Table     | Non_unique | Key_name              | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| film_text | 0          | PRIMARY               | 1 | film_id | A | 1000 | NULL | NULL | | BTREE | | |
| film_text | 1          | idx_title_description | 1 | title | NULL | 1000 | NULL | NULL | | FULLTEXT | | |
| film_text | 1          | idx_title_description | 2 | description | NULL | 1000 | NULL | NULL | YES  | FULLTEXT | | |
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
3 rows in set (0.00 sec)
```

#### **能够使用索引的典型场景**
1. 匹配全值，对索引中所有列都指定具体值，即是对索引中的所有列都有等值匹配的条件。

    ```terminal
// type const
mysql> explain select * from rental where rental_date='2005-05-25 17:22:10' and inventory_id = 373 and customer_id = 343 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: rental 
   partitions: NULL
         type: ref
possible_keys: idx_fk_inventory_id,idx_fk_customer_id,idx_rental_date 
          key: idx_rental_date
      key_len: 10
          ref: const,const,const 
         rows: 1
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
    ```

2. 匹配值的范围查询，对索引的值能进行范围查找。

    ```terminal
mysql> explain select * from rental where customer_id >= 373 and customer_id < 400 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: rental 
   partitions: NULL
         type: range 
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id 
      key_len: 2
          ref: NULL 
         rows: 718
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
    ```

3. 匹配最左前缀，仅仅使用索引中的最左边列进行查找。最左匹配原则算是`MySQL`中`BTREE`索引使用的首要原则。

    ```terminal
mysql> alter table payment add index idx_payment_date(payment_date, amount, last_update); Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
--------------------------------------------------------------
mysql> explain select * from payment where payment_date = '2006-02-14 15:16:03' and last_update = '2006-02-15 22:12:32' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ref 
possible_keys: idx_payment_date
          key: idx_payment_date 
      key_len: 5
          ref: const 
         rows: 182
     filtered: 10.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from payment where amount = 3.98 and last_update = '2006-02-15 22:12:32' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 1.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

4. 仅仅对索引进行查询，查询的列都在索引的字段中时，查询的效率更高，即索引覆盖。

    ```terminal
mysql> explain select last_update from payment where payment_date = '2006-02-14 15:16:03' and amount = 3.98 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ref 
possible_keys: idx_payment_date
          key: idx_payment_date 
      key_len: 8
          ref: const,const 
         rows: 8
     filtered: 100.00 
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
    ```

5. 匹配列前缀，仅仅使用索引中的第一列，且只包含索引第一列的开头一部分进行查找，即前缀索引。

    ```terminal
mysql> create index idx_title_desc_part on film_text(title(10), description(20)); Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0
--------------------------------------------------------------
mysql> explain select title from film_text where title like 'AFRICAN%' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: film_text 
   partitions: NULL
         type: range
possible_keys: idx_title_desc_part,idx_title_description 
          key: idx_title_desc_part
      key_len: 42
          ref: NULL 
         rows: 1
     filtered: 100.00 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

6. 能够实现索引匹配部分精确，而其他部分进行范围匹配。

    ```terminal
mysql> explain select inventory_id from rental where rental_date = '2006-02-14 15:16:03' and customer_id >= 300 and customer_id <= 400 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: rental
   partitions: NULL
         type: ref
possible_keys: idx_fk_customer_id,idx_rental_date 
          key: idx_rental_date
      key_len: 5
          ref: const 
         rows: 182
     filtered: 16.85
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
    ```

7. 如果列名是索引，那么使用`column_name is null`就会使用索引。

    ```terminal
mysql> explain select * from payment where rental_id is null \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ref 
possible_keys: fk_payment_rental
          key: fk_payment_rental 
      key_len: 5
          ref: const rows: 1
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
    ```

8. `MySQL 5.6`引入了索引下推特性，进一步优化了查询。索引下推，就是把操作下放，某些情况下的条件过滤操作下放到操作引擎。（同第2点）

#### **存在索引但是不能使用索引的典型场景**
1. 以`%`开头的`like`查询不能使用`BTREE`索引
- 因为`BTREE`索引的结构，`%`开头的查询无法利用索引，一般推荐使用全文索引解决类似的全文检索问题。
- 或者考虑利用利用`InnoDB`表都是聚簇表的特点，采取一种轻量级别的解决方式。
- 一般情况下，索引都会比表小，扫描索引要比扫描表更快（某些特殊情况下，索引比表大，不在本例讨论范围内），而`InnoDB`表上的二级索引实际上存储除了字段的值，还有主键，理想的访问方式应该是首先扫描二级索引获得满足条件`like`匹配的主键列表（这是一个子查询的索引覆盖），之后根据主键回表检索记录，这样访问就避开了全表扫描产生的大量`IO`请求。

    ```terminal
mysql> explain select * from actor where last_name like '%NI%' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 200
     filtered: 11.11 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

2. 数据类型出现隐式转换时，不会使用索引，特别常见的是字符串，`where`条件要把字符常量值引号引起来。

    ```terminal
mysql> explain select * from actor where last_name = 1 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ALL
possible_keys: idx_actor_last_name 
          key: NULL
      key_len: NULL
          ref: NULL 
         rows: 200
     filtered: 10.00 
        Extra: Using where
1 row in set, 3 warnings (0.01 sec)
--------------------------------------------------------------
mysql> explain select * from actor where last_name = '1' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ref
possible_keys: idx_actor_last_name 
          key: idx_actor_last_name
      key_len: 182
          ref: const 
         rows: 1
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
    ```

3. 复合索引条件下，查询条件不包含索引列的最左边部分，不满足最左原则，不会使用复合索引。

    ```terminal
mysql> explain select * from payment where amount = 3.98 and last_update = '2006-02-15 22:12:32' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 1.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

4. `MySQL`估计使用索引比全表扫描更慢，则不使用索引。在查询时，筛选性越高，越容易使用到索引，筛选性越低，越不容易使用索引。

    ```terminal
mysql> select count(*) from film_text where title like 's%';
+----------+
| count(*) |
+----------+
| 119      |
+----------+
1 row in set (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from film_text where title like 's%' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: film_text 
   partitions: NULL
         type: range
possible_keys: idx_title_desc_part,idx_title_description 
          key: idx_title_desc_part
      key_len: 42
          ref: NULL 
         rows: 119
     filtered: 100.00 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> update film_text set title = concat('s', title); 
Query OK, 1000 rows affected (0.08 sec)
Rows matched: 1000  Changed: 1000  Warnings: 0
--------------------------------------------------------------
mysql> explain select * from film_text where title like 's%' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: film_text 
   partitions: NULL
         type: ALL
possible_keys: idx_title_desc_part,idx_title_description 
          key: NULL
      key_len: NULL
          ref: NULL 
         rows: 1000
     filtered: 100.00 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

5. 用`or`分割开的条件，如果`or`前的条件中的列有索引，后面的列没有索引，涉及的索引不会被用到。 因为`or`后面的条件列没有索引，后面的查询会走全表扫描，在存在全表扫描的情况下，没必要多一次索引扫描增加`IO`访问。
    
    ```terminal
mysql> explain select * from payment where customer_id = 203 or amount = 3.96 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL
possible_keys: idx_fk_customer_id
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 10.15 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

6. 索引列上有计算
    
    ```terminal
mysql> explain select * from payment where payment_id = 1 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: const 
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 2
          ref: const 
         rows: 1
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.01 sec)
--------------------------------------------------------------
mysql> explain select * from payment where payment_id + 1 = 2 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

7. 索引列使用了函数

    ```terminal
mysql> explain select * from actor where last_name like 'PE%' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: range 
possible_keys: idx_actor_last_name
          key: idx_actor_last_name 
      key_len: 182
          ref: NULL 
         rows: 6
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.01 sec)
--------------------------------------------------------------
mysql> explain select * from actor where substr(last_name, 1, 2) = 'PE' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ALL 
possible_keys: NULL
         key: NULL 
     key_len: NULL
         ref: NULL 
        rows: 200
    filtered: 100.00 
       Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

8. Not in 和 not exists

    ```terminal
mysql> explain select * from actor where last_name in ('WAHLBERG', 'DAVIS') \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: range 
possible_keys: idx_actor_last_name
          key: idx_actor_last_name 
      key_len: 182
          ref: NULL 
         rows: 5
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from actor where last_name not in ('WAHLBERG', 'DAVIS') \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ALL
possible_keys: idx_actor_last_name 
          key: NULL
      key_len: NULL
          ref: NULL 
         rows: 200
     filtered: 97.50 
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
    ```

9. `Order by`的坑（见`优化 order by`）

### **优化 order by**

```terminal
mysql> desc customer;
+-------------+----------------------+------+-----+-------------------+-----------------------------+
| Field       | Type | Null | Key | Default | Extra |
+-------------+----------------------+------+-----+-------------------+-----------------------------+
| customer_id | smallint(5) unsigned | NO   | PRI | NULL | auto_increment |
| store_id    | tinyint(3) unsigned  | NO   | MUL | NULL | |
| first_name  | varchar(45)          | NO   |     | NULL | |
| last_name   | varchar(45)          | NO   | MUL | NULL | |
| email       | varchar(50)          | YES  |     | NULL | |
| address_id  | smallint(5) unsigned | NO   | MUL | NULL | |
| active      | tinyint(1)           | NO   |     | 1    | |
| create_date | datetime             | NO   |     | NULL | |
| last_update | timestamp            | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------------+----------------------+------+-----+-------------------+-----------------------------+
9 rows in set (0.00 sec)

mysql> show index from customer;
+----------+------------+-------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+----------+------------+-------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| customer | 0 | PRIMARY | 1 | customer_id | A | 599 | NULL | NULL | | BTREE | | |
| customer | 1 | idx_fk_store_id | 1 | store_id | A | 2 | NULL | NULL | | BTREE | | |
| customer | 1 | idx_fk_address_id | 1 | address_id | A | 599 | NULL | NULL | | BTREE | | |
| customer | 1 | idx_last_name | 1 | last_name | A | 599 | NULL | NULL | | BTREE | | |
+----------+------------+-------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
4 rows in set (0.00 sec)

mysql> desc payment;
+--------------+----------------------+------+-----+-------------------+-----------------------------+
| Field | Type | Null | Key | Default | Extra |
+--------------+----------------------+------+-----+-------------------+-----------------------------+
| payment_id   | smallint(5) unsigned | NO | PRI | NULL | auto_increment |
| customer_id  | smallint(5) unsigned | NO | MUL | NULL | |
| staff_id     | tinyint(3) unsigned  | NO | MUL | NULL | |
| rental_id    | int(11)              | YES  | MUL | NULL | |
| amount       | decimal(5,2)         | NO | | NULL | |
| payment_date | datetime             | NO | MUL | NULL | |
| last_update  | timestamp            | NO | | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+--------------+----------------------+------+-----+-------------------+-----------------------------+
7 rows in set (0.00 sec)

mysql> show index from payment;
+---------+------------+--------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name  | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+---------+------------+--------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| payment | 0 | PRIMARY | 1 | payment_id | A | 16086 | NULL | NULL | | BTREE | | |
| payment | 1 | idx_fk_staff_id | 1 | staff_id | A | 2 | NULL | NULL | | BTREE | | |
| payment | 1 | idx_fk_customer_id | 1 | customer_id  | A | 599 | NULL | NULL | | BTREE | | |
| payment | 1 | fk_payment_rental  | 1 | rental_id | A | 16044 | NULL | NULL | YES  | BTREE | | |
| payment | 1 | idx_payment_date | 1 | payment_date | A | 15815 | NULL | NULL | | BTREE | | |
| payment | 1 | idx_payment_date | 2 | amount | A | 15864 | NULL | NULL | | BTREE | | |
| payment | 1 | idx_payment_date | 3 | last_update  | A | 16038 | NULL | NULL | | BTREE | | |
+---------+------------+--------------------+--------------+--------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
7 rows in set (0.00 sec)
```

#### **MySQL中的两种排序方式**
1. **Index方式**
- 通过有序索引顺序扫描直接返回有序数据，这种方式在执行计划分析时，`Extra`列为`Using index`，不需要额外的排序，操作效率较高；
2. **Filesort方式**
- 通过对返回的数据进行排序，就是`Filesort`排序，所有不是通过索引直接返回排序结果的排序都叫`FileSort`排序。`FileSort`排序并不代表通过磁盘文件进行排序，而只是说明进行了一个排序操作，至于排序操作是否使用了磁盘文件或临时表等，则取决于`MySQL`服务器对排序参数的设置和需要排序数据的大小。
- `Filesort`是通过相应的排序算法，将取得的数据在`sort_buffer_size`系统变量设置的内存排序区中进行的排序，如果内存装载不下，它就会将磁盘上的数据进行分块，再对各个数据块进行排序，然后将各个块合并成有序的结果集。`sort_buffer_size`设置的排序区是每个线程独占的，所以同一时刻，`MySQL`中存在多个`sort buffer`排序区。

    ```terminal
mysql> explain select customer_id from customer order by store_id \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: customer 
   partitions: NULL
         type: index 
possible_keys: NULL
          key: idx_fk_store_id 
      key_len: 1
          ref: NULL 
         rows: 599
     filtered: 100.00 
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from customer order by store_id \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: customer 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 599
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.01 sec)
    ```

#### **order by 优化目标**
尽量使用`Index`方式排序返回有序数据，避免使用`FileSort`方式额外排序。

#### **优化原则**
排序字段加索引，尽量使用索引排序，如果这里使用`ID`排序的话，因为`ID`是索引字段，天生就具备有序的特性，所以这种情况都不需要放到`sort buffer`中去 额外进行排序操作。

`where`条件和`order by`使用相同的索引，并且`order by`的顺序和索引顺序相同，并且`order by`的字段都是升序或者都是降序，否则肯定需要额外的排序操作，出现`Filesort`。

只`select`需要的字段，避免非必要的字段查询，只`select`需要的字段，这点非常重要。

- 因为查询的字段较多可能导致数据会超出`sort buffer`的容量，超出之后就需要用到磁盘临时文件，排序的性能会很差。
- 当`select`的字段大小总和`>max_length_for_sort_data`，排序算法会将全字段排序改为`rowid`排序，增加一次回表查询。
    - 全字段排序，一次扫描，直接输出结果集；
    - `rowid`排序，两次扫描，第一次条件取出排序字段和行指针，在`sort buffer`排序好后，根据行指针回表读取。回表会有大量随机`IO`。

对于联合索引，尽可能在索引列上完成排序操作，遵照索引建的最左前缀。
- `order by`语句使用索引最左前列；
- 使用`where`子句与`order by`子句条件列组合满足索引最左前列；
- `where`子句中如果出现索引的范围查询(即`explain`中出现`range`)会导致`order by`索引失效。

```terminal
select * from tablename order by key_part1, key_part2,...;
select * from tablename where key_part1 = 1 order by key_part1 desc, key_part2 desc;
select * from tablename order by key_part1 desc, key_part2 desc;
 
select * from tablename order by key_part1 desc, key_part2 asc;  // order by ASCDESC
select * from tablename order by where key2 = 1 order by key1; // order by
select * from tablename order by key1, key2; //  order by
```

#### **order by的哪些情况可以走索引**

1. 满足最左匹配原则
- `order by`后面的条件，也要遵循联合索引的最左匹配原则。除了遵循最左匹配原则之外，有个非常关键的地方是，后面还是加了`limit`关键字，如果不加它索引会失效。

    ```terminal
mysql> explain select * from payment order by payment_date \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from payment order by payment_date limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: index 
possible_keys: NULL
          key: idx_payment_date 
      key_len: 12
          ref: NULL 
         rows: 10
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from payment order by payment_date, amount limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: index 
possible_keys: NULL
          key: idx_payment_date 
      key_len: 12
          ref: NULL 
         rows: 10
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
// order by
mysql> explain select * from payment order by payment_date, last_update limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
--------------------------------------------------------------
mysql> explain select * from payment order by payment_date, amount, last_update limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: index 
possible_keys: NULL
          key: idx_payment_date 
      key_len: 12
          ref: NULL 
         rows: 10
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
    ```

2. 配合`where`一起使用
- `order by`还能配合`where`一起遵循最左匹配原则。

    ```terminal
mysql> explain select * from payment where payment_date = '2006-02-14 15:16:03' order by amount \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ref 
possible_keys: idx_payment_date
          key: idx_payment_date
      key_len: 5
          ref: const 
         rows: 182
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.01 sec)
--------------------------------------------------------------
mysql> explain select * from payment where payment_date = '2006-02-14 15:16:03' order by last_update \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ref 
possible_keys: idx_payment_date
          key: idx_payment_date 
      key_len: 5
          ref: const 
         rows: 182
     filtered: 100.00
        Extra: Using index condition; Using filesort
1 row in set, 1 warning (0.00 sec)
    ```

3. `order by`后面如果包含了联合索引的多个排序字段，相同的排序依然能命中索引。

    ```terminal
mysql> explain select * from payment order by payment_date desc, amount desc limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: index 
possible_keys: NULL
          key: idx_payment_date 
      key_len: 12
          ref: NULL 
         rows: 10
     filtered: 100.00 
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
    ```

4. 如果某个联合索引字段，在`where`和`order by`中都有，依然能命中索引。

    ```terminal
mysql> explain select * from payment where payment_date = '2006-02-14 15:16:03' order by payment_date, amount \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ref 
possible_keys: idx_payment_date
          key: idx_payment_date 
      key_len: 5
          ref: const 
         rows: 182
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
    ```

#### **order by的哪些情况下不走索引**
1. 没加`where`或`limit`。如果`order by`语句中没有加`where`或`limit`关键字，该`sql`语句将不会走索引。

2. 对不同的索引做`order by`。前面介绍的基本都是联合索引，这一个索引的情况。但如果对多个索引进行`order by`，索引也会失效。

    ```terminal
mysql> explain select * from payment order by payment_date, amount, customer_id limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
    ```

3. 不满足最左匹配原则

    ```terminal
mysql> explain select * from payment order by amount desc limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
    ```

4. 不同的排序

    ```terminal
mysql> explain select * from payment order by payment_date asc, amount desc limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
    ```

#### **order by 小结**

对于 `KEY` `a_b_c(a, b, c)`
1. `order by`能使用索引最左前缀（必须要带`where`或`limit`）；
2. 如果`where`使用索引的最左前缀定义为常量，则`order by`能使用索引
- `where a = const order by b, c`
- `where a = const and b = const order by c`
- `where a = const and b > const order by c`
- `where a = const and b > const order by b, c`
3. 不能使用索引进行排序
- `order by a asc, b desc, c desc`  //  排序不一致 
- `where g = const order by b, c`   //  丢失索引`a`
- `where a = const order by c`      //   丢失索引`b`
- `where a = const order by a, d`   //  `d`不是索引的一部分
- `where a in (...) order by b, c`  // 对于排序来说，多个相等条件也是范围查询

```terminal
/**
where a = const and b > const order by b , c // Using file sort b, c
where a = const and b > const order by c // using filesort b order by b  c
*/
mysql> explain select * from payment where payment_date = '2006-02-14 15:16:03' and amount > 1 order by amount, last_update \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: range 
possible_keys: idx_payment_date
          key: idx_payment_date 
      key_len: 8
          ref: NULL 
         rows: 105
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from payment where payment_date = '2006-02-14 15:16:03' and amount > 1 order by last_update \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: range 
possible_keys: idx_payment_date
          key: idx_payment_date 
      key_len: 8
          ref: NULL 
         rows: 105
     filtered: 100.00
        Extra: Using index condition; Using filesort
1 row in set, 1 warning (0.00 sec)
```

参考：[Mysql-where索引列+orderby主键+limit索引情况分析](https://zhuanlan.zhihu.com/p/356463167)

### **优化 group by**

默认情况下，`MySQL`对所有`group by col1, col2, ...`的字段进行排序，原理和在查询中指定`order by col1, col2, ...`类似。所以，如果显式包括一个包含相同列的`order by`子句，对`MySQL`的实际执行性能没有什么影响。

如果查询保包括`group by`，但用户想避免排序的消耗，可以指定`order by null`禁止排序。

```terminal
// group by  filesort
mysql> explain select first_name, count(*) from actor group by first_name \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 200
     filtered: 100.00
        Extra: Using temporary; Using filesort
1 row in set, 1 warning (0.00 sec)

mysql> explain select first_name, count(*) from actor group by first_name order by null \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: actor 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 200
     filtered: 100.00
        Extra: Using temporary
1 row in set, 1 warning (0.00 sec)
```

### **优化分页查询**
一般分页查询，通过创建索引覆盖能比较好地提高性能。一个常见又很头疼的分页场景是`limit 1000, 20`，`MySQL`排序出前`1020`条记录后，仅仅需要返回第`1001`到`1020`条记录，前`1000`条记录都会被抛弃，查询和排序的代码非常高。

```terminal
mysql> desc film;
+----------------------+---------------------------------------------------------------------+------+-----+-------------------+-----------------------------+
| Field | Type | Null | Key | Default | Extra |
+----------------------+---------------------------------------------------------------------+------+-----+-------------------+-----------------------------+
| film_id | smallint(5) unsigned | NO | PRI | NULL | auto_increment | 
| title | varchar(128) | NO | MUL | NULL | | 
| description | text | YES  | | NULL | |
| release_year | year(4) | YES  | | NULL | |
| language_id | tinyint(3) unsigned | NO | MUL | NULL | |
| original_language_id | tinyint(3) unsigned | YES  | MUL | NULL | |
| rental_duration | tinyint(3) unsigned | NO | | 3 | |
| rental_rate | decimal(4,2) | NO | | 4.99 | |
| length | smallint(5) unsigned | YES  | | NULL | |
| replacement_cost | decimal(5,2) | NO | | 19.99 | |
| rating | enum('G','PG','PG-13','R','NC-17') | YES  | | G | |
| special_features | set('Trailers','Commentaries','Deleted Scenes','Behind the Scenes') | YES  | | NULL | |
| last_update | timestamp | NO | | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+----------------------+---------------------------------------------------------------------+------+-----+-------------------+-----------------------------+
13 rows in set (0.00 sec)

mysql> show index from film;
+-------+------------+-----------------------------+--------------+----------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+-----------------------------+--------------+----------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| film  | 0 | PRIMARY | 1 | film_id | A | 1000 | NULL | NULL | | BTREE | | |
| film  | 1 | idx_title | 1 | title | A | 1000 | NULL | NULL | | BTREE | | |
| film  | 1 | idx_fk_language_id | 1 | language_id | A | 1 | NULL | NULL | | BTREE | | |
| film  | 1 | idx_fk_original_language_id | 1 | original_language_id | A | 1 | NULL | NULL | YES  | BTREE | | |
+-------+------------+-----------------------------+--------------+----------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
4 rows in set (0.00 sec)

mysql> explain select film_id, description from film order by title limit 50, 5 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: film 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 1000
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)

mysql> explain select a.film_id, a.description from film a inner join (select film_id from film order by title limit 50, 5) b on a.film_id = b.film_id \G
*************************** 1. row *************************** 
           id: 1
  select_type: PRIMARY 
        table: <derived2>
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 55
     filtered: 100.00 
        Extra: NULL
*************************** 2. row *************************** 
           id: 1
  select_type: PRIMARY 
        table: a
   partitions: NULL
         type: eq_ref 
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 2
          ref: b.film_id 
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 3. row *************************** 
           id: 2
  select_type: DERIVED 
        table: film
   partitions: NULL
         type: index 
possible_keys: NULL
          key: idx_title 
      key_len: 514
          ref: NULL 
         rows: 55
     filtered: 100.00 
        Extra: Using index
3 rows in set, 1 warning (0.00 sec)

//limit
mysql> explain select * from payment order by rental_id desc limit 410, 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: ALL 
possible_keys: NULL
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 16086
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)

// last_page_record rental_id 4241
mysql> select * from payment order by rental_id desc limit 410, 10;
+------------+-------------+----------+-----------+--------+---------------------+---------------------+
| payment_id | customer_id | staff_id | rental_id | amount | payment_date | last_update |
+------------+-------------+----------+-----------+--------+---------------------+---------------------+
| 5830 | 214 | 2 | 15639 | 2.99 | 2005-08-23 08:03:25 | 2006-02-15 22:14:15 |
| 15100 | 563 | 2 | 15638 | 3.99 | 2005-08-23 07:54:54 | 2006-02-15 22:22:07 |
| 4686 | 172 | 2 | 15637 | 2.99 | 2005-08-23 07:53:38 | 2006-02-15 22:13:44 |
| 10304 | 380 | 2 | 15636 | 2.99 | 2005-08-23 07:50:46 | 2006-02-15 22:17:21 |
| 107 | 4 | 2 | 15635 | 1.99 | 2005-08-23 07:43:00 | 2006-02-15 22:12:30 |
| 15454 | 576 | 1 | 15634 | 0.99 | 2005-08-23 07:34:18 | 2006-02-15 22:22:32 |
| 13402 | 497 | 2 | 15633 | 0.99 | 2005-08-23 07:31:10 | 2006-02-15 22:20:17 |
| 1668 | 60 | 1 | 15632 | 0.99 | 2005-08-23 07:30:26 | 2006-02-15 22:12:45 |
| 2552 | 93 | 2 | 15631 | 2.99 | 2005-08-23 07:30:23 | 2006-02-15 22:12:57 |
| 15559 | 580 | 1 | 15630 | 6.99 | 2005-08-23 07:29:13 | 2006-02-15 22:22:39 |
+------------+-------------+----------+-----------+--------+---------------------+---------------------+
10 rows in set (0.02 sec)

// limit m,n  limit n
mysql> explain select * from payment where rental_id < 15630 order by rental_id desc limit 10 \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE
        table: payment 
   partitions: NULL
         type: range 
possible_keys: fk_payment_rental
          key: fk_payment_rental 
      key_len: 5
          ref: NULL 
         rows: 8043
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
```

### **优化 or 条件**
对于含有`or`的查询子句，要利用索引，`or`之间的每个条件列都必须用到索引。如果没有索引，则应该考虑增加索引。

`Using union`，对`or`的各个字段分别查询后做了一次`union`。

当在建有复合索引的列上面做`or`时，不能用到索引。

### **SQL HINT**
在`SQL`语句中加入人为的提示，达到优化操作的目的。
- `use index`：提供希望`MySQL`参考的索引列表，就可以让`MySQL`不再考虑其他可用的索引，起到建议的作用。
- `ignore index`：让`MySQL`忽略一个或者多个索引。
- `force index`：强制使用一个特定的索引，这是`MySQL`留给用户的一个自行选择执行计划的权利。

## **SQL优化步骤**

### **1. 通过 show status 命令了解各种SQL的执行频率**
```terminal
mysql> show global status where Variable_name = 'Com_select' or Variable_name = 'Com_insert' or Variable_name = 'Com_update' or Variable_name = 'Com_delete';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Com_delete    | 0     |
| Com_insert    | 1017  |
| Com_select    | 14    |
| Com_update    | 0     |
+---------------+-------+
4 rows in set (0.00 sec)
```

### **2. 定位执行效率较低的SQL**
- 通过慢查询日志定位
- 慢查询日志在查询结束后才记录，在应用反映执行效率出现问题时查询慢查询日志并不能定位问题，可以使用 `show processlist`命令查看当前`MySQL`在进行的线程，包括线程状态、是否锁表等，可以实时查看`SQL`的执行情况，同时对锁表进行优化。

### **3. explain 分析执行计划**

```terminal
mysql> explain select sum(amount) from customer a, payment b where 1=1 and a.customer_id = b.customer_id and email = 'JANE.BENNETT@sakilacustomer.org' \G
*************************** 1. row *************************** 
           id: 1
  select_type: SIMPLE 
        table: a
   partitions: NULL
         type: ALL 
possible_keys: PRIMARY
          key: NULL 
      key_len: NULL
          ref: NULL 
         rows: 599
     filtered: 10.00 
        Extra: Using where
*************************** 2. row *************************** 
           id: 1
  select_type: SIMPLE 
        table: b
   partitions: NULL
         type: ref
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id 
      key_len: 2
          ref: sakila.a.customer_id 
         rows: 26
     filtered: 100.00 
        Extra: NULL
2 rows in set, 1 warning (0.00 sec)
```

![Desktop View](/assets/img/20241218/MySQL_index_optimizer01.jpg){: width="500" height="300" }
_执行计划中的所有列_

`id`列，为执行的顺序，每个号码，表示一趟独立的查询，`id`列越大执行优先级越高，`id`相同则从上往下执行，`id`为`NULL`最后执行。

`select_type`列，查询分为简单查询(`SIMPLE`)和复杂查询(`PRIMARY`)。复杂查询分为三类：简单子查询、派生表（`from`语句中的子查询）、`union`查询。

- `SIMPLE`：简单查询。不包含子查询和`union`
- `PRIMARY`：复制查询中的最外层的`select`
- `DERIVED`：包含在`from`子句中的子查询。`MySQL`会将结果存放在一个临时表中，也称为派生表
- `SUBQUERY`：包含在`select`中的子查询（不在`from`子句中）
- `UNION`：在`union`中的第二个和随后的`select`
- `UNIONRESULT`：从`union`临时表检索结果的`result`
    - `union`结果总是放在一个匿名临时表中，临时表不在`SQL`中出现，因此它的`id`是`NULL`。

![Desktop View](/assets/img/20241218/MySQL_index_optimizer02.jpg){: width="500" height="300" }
_执行计划 select_type value_

`table`列，这一列表示`explain`的一行正在访问哪个表（用户操作的用户表，输出结果集的表）。

- 当`from`子句中有子查询时，`table`列是格式，表示当前查询依赖`id=N`的查询，于是先执行`id=N`的查询。
    - `<derivedN>`:  派生表，由`id`等于`N`的语句产生
    - `<subqueryN>`:  由子查询物化产生的表，由`id`等于`N`的语句产生
- 当有 union 时，UNION RESULT 的 table 列的值为`<union1,2>`，`1`和`2`表示参与`union`的`select`行 `id`。
    - `<unionM,N>`: `UNION`得到的结果表。

`type`列，这一列表示`MySQL`在表中找到所需行的方式，即访问类型。`MySQL`决定如何查找表中的行，查找数据行记录的大概范围。

依次从最优到最差分别为：`system > const > eq_ref > ref > range > index > AL`

性能优化的目标，得保证查询至少达到`range`级别，最好达到`ref`。

![Desktop View](/assets/img/20241218/MySQL_index_optimizer03.jpg){: width="500" height="300" }
_执行计划 type 列_

`possible_keys`列，显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用。

`key`列，实际使用的索引。如果为`NULL`，则没有使用索引。`explain`时可能出现`possible_keys`有列，而 `key`显示`NULL`的情况，这种情况是因为表中数据不多，`mysql`认为索引对此查询帮助不大，选择了全表查询。

`key_len`列，

`ref`列，这一列显示了在`key`列记录的索引中，表查找值所用到的列或常量，常见的有：`const`（常量），字段名，指的是`=`号后面的东西。

`row`列，检查的行数，读取的行数越少越好。

`filtered`列，表示存储引擎返回的数据在`server`层（及其他过滤条件）过滤后，剩下多少满足查询的记录数量的比例。

`Extra`列，这一列展示的是额外信息，

- `Using index`： 直接访问索引就足够获取到所需要的数据，不需要回表，就是覆盖索引扫描。
- `Using where`：查询的`where`条件列未被索引覆盖，表示优化器除了利用索引来加速访问之外，还需要根据索引回表查询数据。
- `Using filesort`：`mysql`会对结果使用一个外部索引排序，而不是按索引次序从表里读取行。此时`mysql`会并保存排序关键字和行指针，然后排序关键字并按顺序检索行信息。这种无法利用索引完成的排序操作称为“文件排序”。这种情况下一般也是要考虑使用索引来优化的。
- `NULL`：查询的列未被索引覆盖，查询的`where`条件走了索引
- `Using index condition`：索引下推优化，查询的列不完全被索引覆盖，条件使用索引，是一个范围
- `Using temporary`：`mysql`需要创建一张临时表来处理查询。

![Desktop View](/assets/img/20241218/MySQL_index_optimizer04.jpg){: width="500" height="300" }
_Extra列的常见值_

### **4. 通过show profile分析SQL**

通过`profile`，可以更清楚地了解`SQL`执行过程。`show profile`能够在`SQL`优化时帮助我们了解时间都耗费在哪了。

查看当前`MySQL`是否支持`profile`，

```terminal
mysql> select @@have_profiling;
+------------------+
| @@have_profiling |
+------------------+
| YES              |
+------------------+
1 row in set, 1 warning (0.00 sec)
```

默认`profiling`是关闭的，需要打开`session`级别的`profiling`，

```terminal
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
| 0           |
+-------------+
1 row in set, 1 warning (0.01 sec) 
--------------------------------------------------------------
mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

执行`sql`后，`show profiles`看到当前`SQL`的`QueryID`是`1`，

```terminal
mysql> select count(*) from payment;
+----------+
| count(*) |
+----------+
| 16044    |
+----------+
1 row in set (0.01 sec)
--------------------------------------------------------------
mysql> show profiles;
+----------+------------+------------------------------+
| Query_ID | Duration   | Query                        |
+----------+------------+------------------------------+
| 1        | 0.00616900 | select count(*) from payment |
+----------+------------+------------------------------+
1 row in set, 1 warning (0.00 sec)
```

查看执行过程中线程的每个状态和消耗的时间，

```terminal
mysql> show profile for query 1;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000209 |
| checking permissions | 0.000029 |
| checking permissions | 0.000006 |
| Opening tables       | 0.000048 |
| init                 | 0.000069 |
| System lock          | 0.000018 |
| optimizing           | 0.000014 |
| statistics           | 0.000081 |
| preparing            | 0.000053 |
| executing            | 0.000004 |
| Sending data         | 0.005517 |
| end                  | 0.000026 |
| query end            | 0.000016 |
| closing tables       | 0.000018 |
| freeing items        | 0.000033 |
| cleaning up          | 0.000028 |
+----------------------+----------+
16 rows in set, 1 warning (0.00 sec)
```

`Sending data`状态表示`MySQL`线程开始访问数据行并把结果返回给客户端，不仅仅是返回结果给客户端。由于该状态时，`MySQL`线程需要做大量的磁盘`IO`，所以经常是整个查询中耗时最长的状态。

为了更清晰地看到排序结果，查看`INFORMATION_SCHEMA.PROFILING`表，并按照时间`desc`排序，

```terminal
mysql> select STATE, SUM(DURATION) as Total_R,
ROUND(100*SUM(DURATION) / 
(
select SUM(DURATION) from INFORMATION_SCHEMA.PROFILING where QUERY_ID = 1
), 2) as Pct_R, 
count(*) as Calls,
SUM(DURATION) / COUNT(*) as "R/Call" 
from INFORMATION_SCHEMA.PROFILING
where QUERY_ID = 1 group by STATE order by Total_R desc;
+----------------------+----------+-------+-------+--------------+
| STATE                | Total_R  | Pct_R | Calls | R/Call       |
+----------------------+----------+-------+-------+--------------+
| Sending data         | 0.005517 | 89.43 | 1     | 0.0055170000 |
| starting             | 0.000209 |  3.39 | 1     | 0.0002090000 |
| statistics           | 0.000081 |  1.31 | 1     | 0.0000810000 |
| init                 | 0.000069 |  1.12 | 1     | 0.0000690000 |
| preparing            | 0.000053 |  0.86 | 1     | 0.0000530000 |
| Opening tables       | 0.000048 |  0.78 | 1     | 0.0000480000 |
| checking permissions | 0.000035 |  0.57 | 2     | 0.0000175000 |
| freeing items        | 0.000033 |  0.53 | 1     | 0.0000330000 |
| cleaning up          | 0.000028 |  0.45 | 1     | 0.0000280000 |
| end                  | 0.000026 |  0.42 | 1     | 0.0000260000 |
| System lock          | 0.000018 |  0.29 | 1     | 0.0000180000 |
| closing tables       | 0.000018 |  0.29 | 1     | 0.0000180000 |
| query end            | 0.000016 |  0.26 | 1     | 0.0000160000 |
| optimizing           | 0.000014 |  0.23 | 1     | 0.0000140000 |
| executing            | 0.000004 |  0.06 | 1     | 0.0000040000 |
+----------------------+----------+-------+-------+--------------+
rows in set, 2 warnings (0.01 sec)
```

获取到最消耗时间的线程状态后，`MySQL`支持进一步选择`all、cpu、block io、context switch、page faults`等明细类型来查看`MySQL`在使用什么资源上耗费了过高的时间。

下面选择查看`CPU`的消耗时间，能够发现`Sending data`状态下，时间主要消耗在`CPU`上了。

```terminal
mysql> show profile cpu for query 1;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| starting             | 0.000209 | 0.000162 | 0.000024   |
| checking permissions | 0.000029 | 0.000026 | 0.000003   |
| checking permissions | 0.000006 | 0.000005 | 0.000001   |
| Opening tables       | 0.000048 | 0.000046 | 0.000002   |
| init                 | 0.000069 | 0.000067 | 0.000002   |
| System lock          | 0.000018 | 0.000017 | 0.000001   |
| optimizing           | 0.000014 | 0.000011 | 0.000002   |
| statistics           | 0.000081 | 0.000081 | 0.000002   |
| preparing            | 0.000053 | 0.000050 | 0.000002   |
| executing            | 0.000004 | 0.000003 | 0.000001   |
| Sending data         | 0.005517 | 0.005432 | 0.000016   |
| end                  | 0.000026 | 0.000010 | 0.000016   |
| query end            | 0.000016 | 0.000015 | 0.000002   |
| closing tables       | 0.000018 | 0.000016 | 0.000001   |
| freeing items        | 0.000033 | 0.000017 | 0.000016   |
| cleaning up          | 0.000028 | 0.000027 | 0.000001   |
+----------------------+----------+----------+------------+
16 rows in set, 1 warning (0.00 sec)
```

### **5. MySQL 5.6 提供了对SQL的跟踪trace**
通过`trace`文件可以了解优化器选择`A`执行计划而不选择`B`执行计划，帮助我们更好地理解优化器的行为。打开`trace`，设置格式为`json`，

```terminal
mysql> set OPTIMIZER_TRACE='enabled=on',END_MARKERS_IN_JSON=on; 
Query OK, 0 rows affected (0.00 sec)
```

执行查询，

```terminal
mysql> select sum(amount) from customer a, payment b where 1=1 and a.customer_id = b.customer_id and email = 'JANE.BENNETT@sakilacustomer.org';
+-------------+
| sum(amount) |
+-------------+
| 100.72      |
+-------------+
1 row in set (0.00 sec)
```

检查`INFORMATION_SCHEMA.OPTIMIZER_TRACE`，就能知道`MySQL`怎么执行`SQL`的。

```yaml
{
    "steps": [
        {
            "join_preparation": {
                "select#": 1,
                "steps": [
                    {
                        "expanded_query": "/* select#1 */ select sum(`b`.`amount`) AS `sum(amount)` from `customer` `a` join `payment` `b` where ((1 = 1) and (`a`.`customer_id` = `b`.`customer_id`) and (`a`.`email` = 'JANE. BENNETT@sakilacustomer.org'))"
                    }
                ]
            }
        },
        {
            "join_optimization": {
                "select#": 1,
                "steps": [
                    {
                        "condition_processing": {
                            "condition": "WHERE",
                            "original_condition": "((1 = 1) and (`a`.`customer_id` = `b`.`customer_id`) and (`a`.`email` = 'JANE.BENNETT@sakilacustomer.org'))",
                            "steps": [
                                {
                                    "transformation": "equality_propagation",
                                    "resulting_condition": "((1 = 1) and (`a`.`email` = 'JANE.BENNETT@sakilacustomer.org') and multiple equal(`a`.`customer_id`, `b`.`customer_id`))"
                                },
                                {
                                    "transformation": "constant_propagation",
                                    "resulting_condition": "((1 = 1) and (`a`.`email` = 'JANE.BENNETT@sakilacustomer.org') and multiple equal(`a`.`customer_id`, `b`.`customer_id`))"
                                },
                                {
                                    "transformation": "trivial_condition_removal",
                                    "resulting_condition": "((`a`.`email` = 'JANE.BENNETT@sakilacustomer.org') and multiple equal (`a`.`customer_id`, `b`.`customer_id`))"
                                }
                            ]
                        }
                    },
                    {
                        "substitute_generated_columns": {}
                    },
                    {
                        "table_dependencies": [
                            {
                                "table": "`customer` `a`",
                                "row_may_be_null": false,
                                "map_bit": 0,
                                "depends_on_map_bits": []
                            },
                            {
                                "table": "`payment` `b`",
                                "row_may_be_null": false,
                                "map_bit": 1,
                                "depends_on_map_bits": []
                            }
                        ]
                    },
                    {
                        "ref_optimizer_key_uses": [
                            {
                                "table": "`customer` `a`",
                                "field": "customer_id",
                                "equals": "`b`.`customer_id`",
                                "null_rejecting": false
                            },
                            {
                                "table": "`payment` `b`",
                                "field": "customer_id",
                                "equals": "`a`.`customer_id`",
                                "null_rejecting": false
                            }
                        ]
                    },
                    {
                        "rows_estimation": [
                            {
                                "table": "`customer` `a`",
                                "table_scan": {
                                    "rows": 599,
                                    "cost": 5
                                }
                            },
                            {
                                "table": "`payment` `b`",
                                "table_scan": {
                                    "rows": 16086,
                                    "cost": 97
                                }
                            }
                        ]
                    },
                    {
                        "considered_execution_plans": [
                            {
                                "plan_prefix": [],
                                "table": "`customer` `a`",
                                "best_access_path": {
                                    "considered_access_paths": [
                                        {
                                            "access_type": "ref",
                                            "index": "PRIMARY",
                                            "usable": false,
                                            "chosen": false
                                        },
                                        {
                                            "rows_to_scan": 599,
                                            "access_type": "scan",
                                            "resulting_rows": 59.9,
                                            "cost": 124.8,
                                            "chosen": true
                                        }
                                    ]
                                },
                                "condition_filtering_pct": 100,
                                "rows_for_plan": 59.9,
                                "cost_for_plan": 124.8,
                                "rest_of_plan": [
                                    {
                                        "plan_prefix": [
                                            "`customer` `a`"
                                        ],
                                        "table": "`payment` `b`",
                                        "best_access_path": {
                                            "considered_access_paths": [
                                                {
                                                    "access_type": "ref",
                                                    "index": "idx_fk_customer_id",
                                                    "rows": 26.855,
                                                    "cost": 1930.3,
                                                    "chosen": true
                                                },
                                                {
                                                    "rows_to_scan": 16086,
                                                    "access_type": "scan",
                                                    "using_join_cache": true,
                                                    "buffers_needed": 1,
                                                    "resulting_rows": 16086,
                                                    "cost": 192812,
                                                    "chosen": false
                                                }
                                            ]
                                        },
                                        "condition_filtering_pct": 100,
                                        "rows_for_plan": 1608.6,
                                        "cost_for_plan": 2055.1,
                                        "chosen": true
                                    }
                                ]
                            },
                            {
                                "plan_prefix": [],
                                "table": "`payment` `b`",
                                "best_access_path": {
                                    "considered_access_paths": [
                                        {
                                            "access_type": "ref",
                                            "index": "idx_fk_customer_id",
                                            "usable": false,
                                            "chosen": false
                                        },
                                        {
                                            "rows_to_scan": 16086,
                                            "access_type": "scan",
                                            "resulting_rows": 16086,
                                            "cost": 3314.2,
                                            "chosen": true
                                        }
                                    ]
                                },
                                "condition_filtering_pct": 100,
                                "rows_for_plan": 16086,
                                "cost_for_plan": 3314.2,
                                "pruned_by_cost": true
                            }
                        ]
                    },
                    {
                        "attaching_conditions_to_tables": {
                            "original_condition": "((`b`.`customer_id` = `a`.`customer_id`) and (`a`.`email` = 'JANE. BENNETT@sakilacustomer.org'))",
                            "attached_conditions_computation": [],
                            "attached_conditions_summary": [
                                {
                                    "table": "`customer` `a`",
                                    "attached": "(`a`.`email` = 'JANE.BENNETT@sakilacustomer.org')"
                                },
                                {
                                    "table": "`payment` `b`",
                                    "attached": null
                                }
                            ]
                        }
                    },
                    {
                        "refine_plan": [
                            {
                                "table": "`customer` `a`"
                            },
                            {
                                "table": "`payment` `b`"
                            }
                        ]
                    }
                ]
            }
        },
        {
            "join_execution": {
                "select#": 1,
                "steps": []
            }
        }
    ]
}
```
{: file='optimizer_trace.yml' .nolineno }

### **6. 确定问题并采取相应的优化措施，比如全表扫描就需要对索引优化。**

## **最后**

没有性能优化的“绝对真理”，而应该是在实际的业务场景下通过测试来验证你关于执行计划以及响应时间的假设，Trade-off balance。
